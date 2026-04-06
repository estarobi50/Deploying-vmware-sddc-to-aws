# deploying-vmware-sddc-to-aws

## Overview
Deploy two separate VMware SDDCs to AWS using VMware Cloud on AWS, both running
on dedicated i4i.metal bare-metal EC2 instances in US East (N. Virginia).
SDDC 1 (CUIMC-VSI-SDDC01) hosts general server workloads across 11 hosts.
SDDC 2 (CUIMC-VDI-SDDC01) is a dedicated 3-host environment running an
Omnissa Horizon virtual desktop infrastructure (VDI). Both SDDCs are grouped
together under CUIMCSDDCGroup01 and connected via a shared VMware Managed
Transit Gateway (vTGW) to on-premises infrastructure and the AWS environment.

## Architecture
On-premises (Campus / Sungard DC)
  → Existing Direct Connect (BGP ASN)
  → Direct Connect Gateway → VMware Managed Transit Gateway (vTGW)
  → vTGW peered to AWS Transit Gateway (aTGW) → Shared Services VPC 172.25.0.0/16
  → SDDC Group: CUIMCSDDCGroup01
      → CUIMC-VSI-SDDC01 (VSI — General Workloads)
            Region: US East (N. Virginia) | AZ: us-east-1c
            Hosts: 11 x i4i.metal | Clusters: 1 | Cores: 108
            CPU: 248.4 GHz | Memory: 1.5 TiB | Storage: 31.1 TiB
            Management CIDR: 10.75.0.0/23
            vCenter Public IP: 54.172.242.74 | Private IP: 10.75.1.196
            NSX Manager IP: 10.75.1.131
      → CUIMC-VDI-SDDC01 (VDI — Omnissa Horizon)
            Region: US East (N. Virginia) | AZ: us-east-1c
            Hosts: 3 x i4i.metal | Clusters: 1 | Cores: 108
            CPU: 248.4 GHz | Memory: 1.5 TiB | Storage: 31.1 TiB
            Management CIDR: 10.75.2.0/23
            vCenter Public IP: 3.234.102.192 | Private IP: 10.75.3.196
            NSX Manager IP: 10.75.3.131
  → Both vCenters linked (Hybrid Linked Mode) for single pane of glass management

## Technologies Used
- VMware vSphere         — compute virtualization layer
- VMware vSAN            — software-defined storage (compression + dedup)
- VMware NSX-T           — software-defined networking, Tier 0/1 routing,
                           micro-segmentation, gateway and distributed firewalls
- VMware vCenter Server  — centralized management (one per SDDC, linked)
- VMware HCX             — workload migration platform (bulk + live migration)
- Hybrid Linked Mode     — unified on-prem + cloud vCenter management view
- Cloud Gateway Appliance (CGA) — on-prem appliance linking vCenters for HLM
- Amazon EC2 i4i.metal  — dedicated bare-metal host instances (both SDDCs)
- AWS Transit Gateway (aTGW) — AWS-side routing to Shared Services VPC
- VMware Transit Gateway (vTGW) — VMware-managed TGW auto-created by SDDC group
- AWS Direct Connect     — dedicated private network connectivity (BGP)
- Omnissa Horizon        — virtual desktop and app delivery (VDI SDDC)
- Active Directory (LDAP/LDAPS) — identity source for vCenter SSO and RBAC

## Deployment Steps

  SDDC DEPLOYMENT
  ────────────────────────────────────────────────────────────

  1. Deploy both SDDCs in US East (N. Virginia)
       CUIMC-VSI-SDDC01
         - Instance type: i4i.metal
         - Hosts: 11
         - Management CIDR: 10.75.0.0/23
         - Availability Zone: us-east-1c

       CUIMC-VDI-SDDC01
         - Instance type: i4i.metal
         - Hosts: 3
         - Management CIDR: 10.75.2.0/23
         - Availability Zone: us-east-1c

       Wait approximately 90-120 minutes per SDDC for deployment to complete.

  2. Group both SDDCs together
       - In VMware Cloud Console → SDDC Groups → Create Group
       - Group name: CUIMCSDDCGroup01
       - Add both CUIMC-VSI-SDDC01 and CUIMC-VDI-SDDC01
       - Grouping automatically creates a VMware Managed Transit Gateway (vTGW)
       - Transit Connect Status confirms: CONNECTED for both SDDCs

  CONNECTIVITY
  ────────────────────────────────────────────────────────────

  3. Attach vTGW to Direct Connect Gateway
       - In CUIMCSDDCGroup01 → Direct Connect tab → Add Account
       - Attach to CUIMC Direct Connect Gateway
         (Direct Connect Gateway ID: d0f1379b-a962-4a4b-b86d-ee28f8c5261c)
       - Allowed Prefixes: 10.75.0.0/17
       - Attachment location: US East (N. Virginia)
       - Confirm State: CONNECTED

  4. Peer vTGW to AWS Transit Gateway (aTGW)
       - In CUIMCSDDCGroup01 → External TGW tab → Add TGW
       - AWS Account ID: 751405857489
       - TGW ID: tgw-06ef7c9d533bcb76b
       - TGW Location: US East (N. Virginia)
       - Routes: 172.25.0.0/16 (Shared Services VPC)
       - Confirm State: CONNECTED

  5. Verify vTGW route table
       Route table should show paths to:
         - SDDCs to each other (SDDC type routes)
         - SDDCs to on-premises (DGW type routes via Direct Connect Gateway)
         - SDDCs to CUIMC AWS infrastructure (TGW type routes via aTGW)
         Key routes:
           10.75.44.0/22  → VSI SDDC
           10.75.32.0/21  → VSI SDDC
           10.75.2.0/23   → VDI SDDC
           10.75.0.0/23   → VDI SDDC
           172.25.0.0/16  → AWS TGW (Shared Services VPC)
           10.0.0.0/10    → Direct Connect Gateway (on-premises)
           192.168.0.0/24 → VDI SDDC

  HYBRID LINKED MODE (HLM)
  ────────────────────────────────────────────────────────────

  6. Link vCenters (Hybrid Linked Mode)
       - In CUIMCSDDCGroup01 → vCenter Linking → Link All vCenters
       - CUIMC-VDI-SDDC01: vcenter.sddc-3-234-102-192.vmwarevmc.com → Linked
       - CUIMC-VSI-SDDC01: vcenter.sddc-54-172-242-74.vmwarevmc.com → Linked
       - Provides single pane of glass for both SDDCs from one vSphere Client
       - Allows workload migration and shared tag categories across SDDCs

  7. Configure DNS for HLM
       - Change vCenter FQDN resolution to Private IP address
         VSI vCenter FQDN: https://vcenter.sddc-54-172-242-74.vmwarevmc.com
         Resolution Address: Private IP 10.75.1.196 (resolvable from VPN)
         VDI vCenter FQDN: https://vcenter.sddc-3-234-102-192.vmwarevmc.com
         Resolution Address: Private IP 10.75.3.196 (resolvable from VPN)

  8. Deploy Cloud Gateway Appliance (CGA) on-premises
       - Deploy appliance VM in on-premises VxRail cluster
         vCenter Server: sgvxrvc101.mc.cumc.columbia.edu
         Cluster: VxRail-Virtual-SAN-Cluster
         VM Name: SGVAHLM001
         Network: Management Network (static IP)
         IP Address: 10.73.122.103 | Subnet: 255.255.255.128
         Default Gateway: 10.73.122.1
         DNS Servers: 10.144.235.10, 10.104.10.10
         NTP Servers: 10.182.182.10, 10.72.72.10
       - Add VxRail vCenter and VSI vCenter to the CGA

  9. Configure identity sources
       - Add on-premises Active Directory in both CGA and SDDC vCenters
         Identity Source Type: Active Directory over LDAP
         Identity Source Name: MC LDAP
         Domain Name: mc.cumc.columbia.edu
         Primary LDAP: ldaps://sgvwdcmc001.mc.cumc.columbia.edu:636
         Secondary LDAP: ldaps://rbwdcmc001.mc.cumc.columbia.edu:636
         Base DN Users: DC=mc,DC=cumc,DC=columbia,DC=edu

  10. Configure Management Gateway firewall rules for HLM
        Rule: Sungard to vCenter (ID 1015)
          Source: CUMC Sungard | Dest: vCenter | Services: ICMP ALL, SSO, HTTPS
        Rule: vCenter Linking:VC InBound Rule (ID 2030)
          Source: LinkedVCenters | Dest: vCenter | Services: ICMP ALL, HTTPS
        Rule: vCenter Linking:ESX InBound Rule (ID 2031)
          Source: LinkedESX | Dest: ESXi | Services: Provisioning & Remote Con.
        Rule: ESXi Outbound Rule (ID 1013)
          Source: ESXi | Dest: Any | Services: Any | Action: Allow
        Rule: vCenter Outbound Rule (ID 1014)
          Source: vCenter | Dest: Any | Services: Any | Action: Allow
        Default Deny All rule at bottom

  HCX MIGRATION
  ────────────────────────────────────────────────────────────

  11. Deploy HCX Cloud in VSI SDDC (CUIMC-VSI-SDDC01)
        - HCX Manager deployed automatically in SDDC Management Subnet
        - HCX subnet pool allocated from VSI compute subnet: 10.75.43.240/28
        - Network profiles created:
            mgmt-app-network: IP Range 10.75.1.216-10.75.1.231 | GW: 10.75.1.193
            directConnectNetwork1: IP Range 10.75.43.242-10.75.43.254 | GW: 10.75.43.241

  12. Deploy HCX Enterprise Managers on-premises (3 source vCenters)
        Source environments requiring migration:
          - VxRail Cluster      → sgvhcxm001.mc.cumc.columbia.edu (enterprise)
          - Nutanix Cluster     → sgvhcxm002.mc.cumc.columbia.edu (enterprise)
          - vSphere 6.5 Cluster → sgvhcxm003.mc.cumc.columbia.edu (enterprise)
        Each HCX Enterprise Manager pairs to HCX Cloud in CUIMC-VSI-SDDC01
        Only the Hybrid Interconnect appliance deployed per cluster
        (workloads re-IPd at destination — no network extension required)

  13. Create Service Meshes (one per source vCenter)
        3 Service Meshes total:
          - 3 management subnet IPs consumed
          - 3 HCX subnet IPs consumed (from 10.75.43.240/28)
        Each Service Mesh creates 1 Hybrid Interconnect Appliance per cluster

  14. Run Bulk Migrations
        Decision: Re-IP all VMs at destination (no L2 network extension)
        Migration type: Bulk Migration (host-based replication)
        Process per VM:
          - Full synchronization to remote SDDC
          - Delta synchronization
          - Scheduled switchover (maintenance window)
          - Source VM powered off → migrated replica powered on
          - Source VM renamed with POSIX timestamp suffix
          - Original VM copied to Migrated VMs folder in vSphere Templates
        Guest customizations applied during migration:
          - Hostname, IP Address, Gateway, Netmask
          - Primary/Secondary DNS, Windows SID
          - Pre/Post customization scripts supported

  NETWORKING AND SECURITY (NSX-T)
  ────────────────────────────────────────────────────────────

  15. Configure NSX-T network segments
        Segment types available:
          - Routed (default): connectivity to other logical networks and
            external networks via SDDC firewall
          - Extended: extends L2VPN tunnel for single IP space spanning
            SDDC and on-premises
          - Disconnected: isolated, no uplink, VM-only access
        Example segment configured:
          CUIMC Servers | Type: Routed | Subnet: 10.75.44.3/22

  16. Configure NSX-T tagging and grouping
        - Groups used as source/destination in firewall rules
        - Group membership: VMs, IP sets, MAC sets, segment ports,
          AD user groups, other groups
        - Dynamic membership criteria: tag, machine name, OS name,
          computer name (max 5 criteria per group)

  17. Configure Gateway Firewalls
        Management Gateway (Tier 1 — MGW): controls access to SDDC
        management infrastructure (vCenter, ESXi, NSX Manager)
        Default posture: deny all — explicit allow rules required
        Key rules configured:
          Onprem to HCX (ID 2032):
            Source: CUMC Sungard → Dest: HCX
            Services: ICMP Echo, Appliance Mgmt, SSH, HTTPS
          HCX to Onprem (ID 2034):
            Source: HCX → Dest: CUMC Sungard | Services: Any
          Sungard vMotion to ESXi (ID 2033):
            Source: On-Prem vMotion Hosts → Dest: ESXi
            Services: ICMP ALL, Provisioning & Remote Con., VMotion, HTTPS

        Compute Gateway (Tier 1 — CGW): controls north-south workload traffic

  18. Configure Distributed Firewalls (DFW)
        Applied at VM vNIC level — controls east-west traffic within SDDC
        Rules evaluated top-down; default rule allows all traffic
        Rule categories (evaluated in order of precedence):
          1. Ethernet    — Layer 2 traffic (MAC-based)
          2. Emergency   — quarantine and allow rules
          3. Infrastructure — shared services (AD, DNS, NTP, DHCP, backup)
          4. Environment — rules between security zones (prod, dev, etc.)
          5. Application — rules between application tiers / microservices

## Key Learnings
- Grouping both SDDCs together automatically creates the VMware Managed
  Transit Gateway (vTGW) — this is required before Direct Connect or
  AWS TGW peering can be configured
- The vTGW route table provides full mesh routing: SDDC-to-SDDC,
  SDDC-to-on-premises (via DGW), and SDDC-to-AWS (via aTGW) — verify
  all route types are present after peering is complete
- Hybrid Linked Mode requires vCenter DNS to resolve to the Private IP —
  the public IP will not work for HLM; update DNS before linking
- The Cloud Gateway Appliance (CGA) must be deployed on-premises before
  HLM can link the environments — deploy it in the VxRail cluster where
  it has access to both the campus network and Direct Connect path
- HCX deploys one HCX Manager per SDDC management subnet and requires
  a dedicated compute subnet pool for the Hybrid Interconnect appliance —
  plan the 10.75.43.240/28 block before creating service meshes
- Re-IPing VMs at migration (rather than extending L2 networks) simplifies
  the long-term network architecture — the Hybrid Interconnect appliance
  alone is sufficient; Network Extension appliance is not needed
- Bulk Migration keeps source VMs online during replication — only a brief
  downtime occurs at switchover, making it suitable for production workloads
  during a maintenance window using the scheduled migration option
- NSX-T Distributed Firewall default rule allows all east-west traffic —
  build category-based policies (Infrastructure → Environment → Application)
  before restricting traffic to avoid breaking workloads
- Management Gateway default posture is deny-all — firewall rules must be
  explicitly created before HLM, HCX, and vCenter linking will work
- Both SDDCs share the same Org ID; when contacting VMware support always
  provide the specific SDDC ID, SDDC Version, and NSX Manager IP to avoid
  confusion between VSI and VDI environments
