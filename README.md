# deploying-vmware-sddc-to-aws 

## Overview
Deploy two separate VMware SDDCs to AWS using VMware Cloud on AWS, both running
on dedicated i4i.metal bare-metal EC2 instances. SDDC 1 hosts general server
workloads across 11 hosts. SDDC 2 is a dedicated 3-host environment running an
Omnissa Horizon virtual desktop infrastructure (VDI).

## Architecture
On-premises VMware environment
  → AWS Direct Connect or IPsec VPN
  → SDDC 1: General Workloads (11 x i4i.metal)
      → Management Cluster (vCenter, NSX Manager, vSAN)
      → Compute Cluster (general server workloads)
      → NSX network segments (micro-segmented)
  → SDDC 2: Omnissa Horizon VDI (3 x i4i.metal)
      → Management Cluster (vCenter, NSX Manager, vSAN)
      → Horizon Connection Servers
      → Horizon desktop pools (persistent and/or floating)
      → NSX network segments for VDI traffic
  → Native AWS services (S3, IAM, CloudWatch, KMS)

## Technologies Used
- VMware vSphere         — compute virtualization layer
- VMware vSAN            — software-defined storage (compression + dedup)
- VMware NSX             — software-defined networking and micro-segmentation
- VMware vCenter Server  — centralized management console
- VMware HCX             — workload mobility and live migration
- Amazon EC2 i4i.metal  — dedicated bare-metal host instances (both SDDCs)
- Omnissa Horizon        — virtual desktop and app delivery (SDDC 2)
- AWS Direct Connect     — dedicated private network connectivity
- AWS IAM                — access management and permissions
- AWS CloudWatch         — cloud-side monitoring and metrics
- AWS KMS                — encryption key management
- AWS S3                 — data archival and lifecycle management
- vRealize Operations    — performance, capacity, and health monitoring

## Deployment Steps

  SDDC 1 — General Workloads (11 x i4i.metal)

  1. Prepare AWS account
       - Verify EC2 service quotas for i4i.metal instances (11 hosts needed)
       - Request quota increases via AWS Support if not already approved
       - Select AWS region based on latency and data residency requirements
       - Create or identify a VPC for SDDC connectivity

  2. Configure VMware Cloud Services account
       - Create organization at VMware Cloud Services console
       - Link AWS account by granting required IAM permissions to VMware
       - Verify VMware license entitlements

  3. Plan network and IP addressing
       - Document existing subnets to avoid CIDR conflicts
       - Plan management subnet: /23 (required by VMware)
       - Plan workload network segments per application tier
       - Decide connectivity method: Direct Connect or IPsec VPN

  4. Deploy SDDC 1
       - From VMware Cloud Services console → Create SDDC
       - Select AWS region and availability zone
       - Instance type: i4i.metal
       - Set hosts to 11
       - Name the SDDC (e.g. sddc-general-workloads)
       - Configure management subnet (/23)
       - Configure internet access and connectivity settings
       - Submit deployment — wait approximately 90-120 minutes

  5. Configure networking (post-deploy)
       - Create NSX workload segments per application tier
       - Configure routing between segments and to external networks
       - Set up DNS forwarding between on-premises and cloud
       - Configure DHCP via NSX DHCP services or existing DHCP
       - If Direct Connect: configure BGP and verify routes
       - If VPN: configure customer gateway and IPsec tunnel

  6. Configure firewall and security
       - Implement NSX distributed firewall rules (VM-level east-west control)
       - Implement NSX gateway firewall (north-south perimeter control)
       - Enable NSX micro-segmentation per workload tier
       - Configure vSAN encryption (data at rest)
       - Enable IPsec or MACsec for encryption in transit
       - Integrate vCenter SSO with existing identity provider (AD/LDAP)
       - Apply RBAC roles with minimum necessary permissions
       - Configure AWS IAM integration for AWS resource access
       - Set up AWS KMS for key management

  7. Configure storage policies
       - Define vSAN storage policies per workload performance tier
       - Enable compression and deduplication on vSAN
       - Configure snapshot policies for data protection
       - Set up data lifecycle rules for archival to S3
       - Implement VMware Cloud Disaster Recovery
       - Integrate backup solution and verify restore capability

  8. Set up monitoring
       - Deploy vRealize Operations (vROps) for infrastructure monitoring
       - Integrate vROps with AWS CloudWatch for unified visibility
       - Configure alerts for performance, capacity, and health thresholds
       - Enable audit trail logging for all administrative actions

  ────────────────────────────────────────────────────────────

  SDDC 2 — Omnissa Horizon VDI (3 x i4i.metal)

  9. Deploy SDDC 2
       - From VMware Cloud Services console → Create SDDC
       - Same region as SDDC 1 for low-latency inter-SDDC connectivity
       - Instance type: i4i.metal
       - Set hosts to 3
       - Name the SDDC (e.g. sddc-horizon-vdi)
       - Configure a separate management subnet (/23, non-overlapping with SDDC 1)
       - Submit deployment — wait approximately 90-120 minutes

  10. Configure Horizon networking
        - Create dedicated NSX segments for:
            Horizon management traffic
            Horizon desktop pool traffic
            Horizon DMZ / Unified Access Gateway (UAG)
        - Configure firewall rules to allow:
            Clients → UAG (HTTPS 443, Blast 8443, PCoIP 4172)
            UAG → Connection Servers (internal ports)
            Connection Servers → vCenter and ESXi hosts
        - Set up DNS for Horizon connection server FQDNs

  11. Deploy Omnissa Horizon components
        - Deploy Horizon Connection Servers (minimum 2 for HA)
        - Deploy Unified Access Gateway (UAG) for external client access
        - Connect Horizon to vCenter in SDDC 2
        - Configure Horizon Composer or Instant Clone for desktop pools
        - Create desktop pools:
            Floating (non-persistent) for general users
            Persistent (dedicated) for power users if required
        - Configure application pools if publishing hosted apps
        - Integrate with Active Directory for user entitlements

  12. Configure Horizon storage
        - Define vSAN storage policy optimized for VDI (high IOPS, low latency)
        - Enable vSAN deduplication and compression (high savings on VDI clones)
        - Configure Horizon replica and OS disk placement on vSAN
        - Set up user profile management (FSLogix or Horizon DEM)

  13. Set up Horizon monitoring
        - Enable Horizon built-in monitoring dashboard
        - Integrate with vROps for Horizon for desktop pool health metrics
        - Configure alerts for session counts, pool provisioning failures,
          and datastore capacity thresholds

## Key Learnings
- Both SDDCs run exclusively on i4i.metal — higher NVMe storage performance
  vs i3.metal, better suited for the IOPS demands of both server workloads
  and VDI desktop pools
- SDDC 2 (Horizon) runs the minimum 3-host configuration — monitor vSAN
  capacity closely as VDI clone sprawl can consume storage quickly
- Deploying Horizon in a separate SDDC isolates VDI resource contention
  from general server workloads — the right call for consistent desktop performance
- Omnissa Horizon (formerly VMware Horizon) requires the UAG for secure
  external access — do not expose Connection Servers directly to the internet
- Instant Clone desktop pools provision desktops in seconds but require
  careful vSAN policy tuning to avoid storage performance bottlenecks at peak login
- vSAN deduplication delivers significant savings on VDI workloads due to
  the high similarity between OS clones — enable it on SDDC 2 from day one
- NSX micro-segmentation is especially important in the Horizon SDDC to
  prevent lateral movement between desktop sessions
- Direct Connect is preferred over VPN for both SDDCs — VDI traffic is
  latency-sensitive and VPN jitter will degrade the desktop experience
- Snapshot accumulation consumes vSAN capacity quickly — enforce snapshot
  retention policies from day one on both SDDCs
