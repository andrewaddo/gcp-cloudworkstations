# Enterprise Design: Multi-Project & Multi-VPC Private Cloud Workstations

This document outlines the architectural design for managing multiple private Cloud Workstations clusters across different Google Cloud projects and VPCs, utilizing a Hub-and-Spoke networking and DNS model.

## Objective
To provide seamless, secure, and private access to multiple Cloud Workstations clusters from an on-premises environment (or corporate laptop via VPN) without using public DNS or custom domains.

---

## 1. Architectural Overview

### The Components
- **Hub Project**: A centralized networking project (e.g., `net-hub-prod`) that hosts the hybrid connectivity (VPN/Interconnect).
- **Spoke Projects**: Individual projects (e.g., `app-dev-team-a`, `app-dev-team-b`) where the Cloud Workstations clusters and PSC endpoints reside.
- **Shared VPC or VPC Peering**: Connects the Hub VPC to each Spoke VPC to allow IP-level routing to the Private Service Connect (PSC) IPs.

### The Connectivity Flow
1. **Developer Laptop**: Initiates a request to `https://my-ide.cluster-aaa.cloudworkstations.dev`.
2. **Corporate DNS**: Forwards all `*.cloudworkstations.dev` queries to the **GCP DNS Inbound Forwarding IP** in the Hub VPC.
3. **GCP Cloud DNS (Hub)**: Resolves the request by looking up the authorized Private DNS Zones associated with the Hub VPC.
4. **Traffic Routing**: The browser routes the traffic over the VPN/Interconnect to the Hub VPC, which then routes it to the specific PSC IP in the Spoke VPC.

---

## 2. DNS Design: Hub-and-Spoke Model

Cloud DNS is a project-level service, but its **visibility** is linked to specific VPCs. To manage multiple clusters, we use "Cross-Project DNS Visibility."

### Step 1: Configure the Hub DNS Policy
In the **Hub Project**, create an Inbound Server Policy. This provides a stable internal IP for your corporate DNS to target.
```bash
# In Hub Project
gcloud dns policies create hub-inbound-policy \
    --networks=hub-vpc \
    --enable-inbound-forwarding \
    --project=net-hub-prod
```
*Note the allocated IP (e.g., 10.0.0.53) for the next step.*

### Step 2: Configure Corporate DNS
Update your on-premises DNS servers (Active Directory, Bind, etc.) with a conditional forwarder:
- **DNS Suffix**: `*.cloudworkstations.dev`
- **Forward to**: `10.0.0.53` (Hub Inbound IP)

### Step 3: Create Spoke DNS Zones
In each **Spoke Project**, create a Private DNS Zone for its specific cluster and authorize the **Hub VPC** to see it.

**For Cluster A (in Spoke Project 1):**
```bash
# In Spoke Project 1
gcloud dns managed-zones create cluster-aaa-zone \
    --dns-name="cluster-aaa.cloudworkstations.dev." \
    --visibility=private \
    --networks=https://www.googleapis.com/compute/v1/projects/net-hub-prod/global/networks/hub-vpc \
    --project=spoke-project-1

# Add the Wildcard A Record pointing to Cluster A's PSC IP
gcloud dns record-sets transaction start --zone=cluster-aaa-zone --project=spoke-project-1
gcloud dns record-sets transaction add 10.1.1.100 \
    --name="*.cluster-aaa.cloudworkstations.dev." \
    --type=A --zone=cluster-aaa-zone --project=spoke-project-1
gcloud dns record-sets transaction execute --zone=cluster-aaa-zone --project=spoke-project-1
```

---

## 3. Networking & PSC Design

Each cluster requires a unique **Private Service Connect (PSC)** endpoint.

| Resource | Project | VPC | PSC Internal IP |
| :--- | :--- | :--- | :--- |
| Cluster A | Spoke 1 | Spoke-VPC-1 | 10.1.1.100 |
| Cluster B | Spoke 2 | Spoke-VPC-2 | 10.2.2.100 |

### Key Considerations
1. **IP Overlap**: Ensure Spoke VPC subnets do not overlap with each other or the Hub VPC if using VPC Peering.
2. **Routing**: The Hub VPC must have routes to the Spoke VPCs (via Peering or a Transit Gateway/NCC) to reach the PSC IPs.
3. **Firewalls**: Allow ingress traffic on port 443 from the Hub VPC (and VPN ranges) into the Spoke VPCs.

---

## 4. Troubleshooting Summary

| Issue | Root Cause | Solution |
| :--- | :--- | :--- |
| **DNS Resolution Fails** | Corporate DNS is not forwarding to the Hub Inbound IP. | Verify conditional forwarding for `*.cloudworkstations.dev`. |
| **Connection Refused** | The Hub VPC is not authorized to see the Spoke DNS Zone. | Check the `--networks` flag in the Spoke DNS zone configuration. |
| **404 Not Found** | Browser is hitting a port other than 443 (common during auth redirects). | Ensure end-to-end connectivity on port 443. |
| **401 Permission Denied** | User account lacks `roles/workstations.user` on the workstation config. | Grant the IAM role to the specific user/group in the Spoke Project. |
