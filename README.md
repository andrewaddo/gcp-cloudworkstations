# Private GCP Cloud Workstations Setup

## Project Description
This project provides a reference implementation for setting up a secure, fully private Google Cloud Workstations environment. The goal is to deploy developer workstations that have no public internet exposure, complying with strict organizational policies (e.g., no public IPs, mandatory Shielded VM features). 

## The Solution
To achieve a completely private setup, the architecture utilizes the following components:
1. **Custom VPC & Private Subnet**: A dedicated network with Private Google Access enabled.
2. **Private Workstation Cluster**: A Cloud Workstations cluster deployed with a Private Gateway (`--enable-private-endpoint`), meaning it cannot be reached from the public internet.
3. **Private Service Connect (PSC)**: An internal IP address and a forwarding rule that routes internal VPC traffic to the cluster's hidden service attachment.
4. **Private Cloud DNS**: A private managed zone that maps the cluster's dynamically generated hostname (e.g., `*.cluster-xyz.cloudworkstations.dev`) to the internal PSC IP address.
5. **Secure Workstation Configuration**: Enforces `--disable-public-ip-addresses` and Shielded VM features (`--shielded-secure-boot`, etc.) to meet organization security constraints.

---

## Setup Steps

### 1. Networking Infrastructure
Create a custom VPC, a private subnet, and firewall rules allowing internal communication.
```bash
# Create VPC
gcloud compute networks create workstations-vpc --subnet-mode=custom

# Create Subnet with Private Google Access
gcloud compute networks subnets create workstations-subnet \
    --network=workstations-vpc \
    --range=10.0.0.0/24 \
    --region=us-central1 \
    --enable-private-ip-google-access

# Allow internal traffic
gcloud compute firewall-rules create allow-internal-workstations \
    --network=workstations-vpc \
    --allow=tcp,udp,icmp \
    --source-ranges=10.0.0.0/24
```

### 2. Private Cloud Workstations Cluster
Create the cluster with a private endpoint. This step takes some time.
```bash
gcloud workstations clusters create private-cluster \
    --region=us-central1 \
    --network=projects/$PROJECT_ID/global/networks/workstations-vpc \
    --subnetwork=projects/$PROJECT_ID/regions/us-central1/subnetworks/workstations-subnet \
    --enable-private-endpoint
```
After creation, retrieve the `clusterHostname` and `serviceAttachmentUri`:
```bash
gcloud workstations clusters describe private-cluster --region=us-central1
```

### 3. Private Service Connect (PSC)
Reserve an internal IP and create a forwarding rule pointing to the `serviceAttachmentUri`.
```bash
# Reserve Internal IP
gcloud compute addresses create psc-workstations-ip \
    --region=us-central1 \
    --subnet=workstations-subnet \
    --addresses=10.0.0.100

# Create Forwarding Rule (Replace the target URI with your specific serviceAttachmentUri)
gcloud compute forwarding-rules create psc-workstations-endpoint \
    --region=us-central1 \
    --network=workstations-vpc \
    --address=psc-workstations-ip \
    --target-service-attachment=https://www.googleapis.com/compute/v1/projects/.../serviceAttachments/...
```

### 4. Private DNS Configuration
Create a private DNS zone so the VPC can resolve the workstation URLs to the PSC IP.
```bash
# Replace CLUSTER_HOSTNAME with your actual cluster hostname
gcloud dns managed-zones create workstations-private-zone \
    --description="Private zone for Cloud Workstations" \
    --dns-name=CLUSTER_HOSTNAME. \
    --networks=workstations-vpc \
    --visibility=private

# Add a Wildcard A Record pointing to the PSC IP
gcloud dns record-sets transaction start --zone=workstations-private-zone
gcloud dns record-sets transaction add 10.0.0.100 \
    --name="*.CLUSTER_HOSTNAME." \
    --ttl=300 \
    --type=A \
    --zone=workstations-private-zone
gcloud dns record-sets transaction execute --zone=workstations-private-zone
```

### 5. Workstation Configuration & Instance
Create a configuration enforcing no public IPs and Shielded VM (if required by org policy), then create the workstation.
```bash
gcloud workstations configs create default-config \
    --cluster=private-cluster \
    --region=us-central1 \
    --disable-public-ip-addresses \
    --shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring

gcloud workstations create my-workstation \
    --cluster=private-cluster \
    --config=default-config \
    --region=us-central1

# Start the workstation
gcloud workstations start my-workstation \
    --cluster=private-cluster \
    --config=default-config \
    --region=us-central1
```

---

## IAM Permissions
Ensure your user account has the required Cloud Workstations IAM role to connect to the workstation instances:
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="user:your-email@example.com" \
    --role="roles/workstations.workstationAdmin"
```
*(Without this permission, you will receive a `401 Permission Denied` error during the browser authentication phase).*

---

## Verification & Local Browser Access

Since the workstation has no public IP, it can only be accessed from within the VPC. If you are not connected via a VPN/Interconnect, you must tunnel your browser traffic using a Bastion VM and Identity-Aware Proxy (IAP).

### Step 1: Create a Bastion/Verification VM
Deploy a small VM in the same VPC to act as a jump host.
```bash
gcloud compute instances create verification-vm \
    --zone=us-central1-a \
    --network=workstations-vpc \
    --subnet=workstations-subnet \
    --no-address \
    --shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring

# Allow SSH via IAP
gcloud compute firewall-rules create allow-iap-ssh-workstations \
    --network=workstations-vpc \
    --allow=tcp:22 \
    --source-ranges=35.235.240.0/20
```

### Step 2: Establish the Tunnel

You have two choices for tunneling:

#### Method A: Direct Port Forwarding (Requires Sudo)
If you can run `sudo` on your local machine, bind port 443 locally. This natively handles the Google Auth redirect flow which expects port 443.

1.  Run the IAP SSH tunnel:
    ```bash
    sudo gcloud compute ssh verification-vm \
        --zone=us-central1-a \
        --tunnel-through-iap \
        -- -L 443:10.0.0.100:443
    ```
2.  Add to your local `/etc/hosts`:
    ```text
    127.0.0.1 my-workstation.CLUSTER_HOSTNAME
    ```
3.  Open the browser to: `https://my-workstation.CLUSTER_HOSTNAME`

#### Method B: SOCKS Proxy (Recommended / No Sudo)
This method routes DNS and traffic dynamically through the VM, bypassing the need for local `/etc/hosts` changes.

1.  Create the SOCKS Proxy tunnel:
    ```bash
    gcloud compute ssh verification-vm \
        --zone=us-central1-a \
        --tunnel-through-iap \
        -- -D 5000
    ```
2.  Launch a browser (e.g., Chrome) configured to use this SOCKS proxy. On MacOS:
    ```bash
    /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
        --user-data-dir="$HOME/chrome-proxy-profile" \
        --proxy-server="socks5://localhost:5000" \
        --host-resolver-rules="MAP * ~NOTFOUND , EXCLUDE localhost" \
        "https://my-workstation.CLUSTER_HOSTNAME"
    ```

### Troubleshooting
*   **404 Not Found:** Occurs if you are using an arbitrary local port (like 8443) and the Google Login redirect pushes you back to 443. Use **Method A** or **Method B** to fix.
*   **401 Permission Denied:** The SSH tunnel works perfectly, but your signed-in Google account lacks the `roles/workstations.workstationAdmin` or `roles/workstations.user` permission.
*   **ERR_CONNECTION_REFUSED:** Your SSH tunnel dropped. Restart the tunnel command.
