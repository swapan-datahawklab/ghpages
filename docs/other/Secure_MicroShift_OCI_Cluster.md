**Swapan**: secure private microshift cluster on oracle OCI cloud free tier 

**ChatGPT**: **Crafting a secure MicroShift cluster**

Setting up a secure private MicroShift cluster on Oracle Cloud Infrastructure's free tier involves creating an environment, establishing a compute instance, installing RHEL, and securing the MicroShift setup.

**Setting up and securing**

I’m detailing steps to set up a secure private cluster using OCI Ampere A1, install RHEL, configure MicroShift, and ensure proper firewall settings, while maintaining a single node cluster without a public IP.

**ChatGPT**: Below is an example end-to-end guide for setting up a **single-node MicroShift cluster** on the **Oracle Cloud Infrastructure (OCI) Free Tier** in a **secure and private** manner. This guide focuses on the “bare essentials” of getting MicroShift running in a private subnet (no public IP) and ensuring traffic is restricted. You will connect via an SSH tunnel or Bastion host when you need to access the node or the workloads.

> **Note:** This guide assumes basic familiarity with OCI’s networking concepts, the OCI web console, and the usage of SSH keys. It also assumes you have an active **Oracle Cloud Free Tier** account.

---

## 1. Overview of the Setup

1. **Compute Instance**:  
   - We will use one Ampere A1 (Arm) VM instance (available in Free Tier) running a RHEL, CentOS, or Fedora image.  
   - MicroShift can run on minimal OS footprints, so Fedora 38 (or newer), CentOS 9 Stream, or RHEL 9 is generally recommended.

2. **Networking**:  
   - A **VCN** (Virtual Cloud Network) with:
     - **Private Subnet**: For the MicroShift node (no public IP assigned).  
     - **Public Subnet**: Optionally for a Bastion host to SSH in (if you want to keep the MicroShift node strictly private).  
   - **Security Lists** or **Network Security Groups (NSG)** with rules to allow inbound SSH from your IP (and any other ports you need specifically).

3. **Access**:
   - SSH into your private MicroShift node via an OCI Bastion Session or a publicly accessible jump/bastion host in the public subnet.

4. **MicroShift**:
   - Minimal install of OpenShift features in a single-node deployment.  
   - Since it’s a single node, it hosts both control-plane and worker-plane components.

---

## 2. Create and Configure the VCN with a Private Subnet

1. **Create a VCN**:  
   - In the OCI Console, navigate to **Networking** > **Virtual Cloud Networks**.  
   - Click **Create VCN**. You can use the “VCN with Internet Connectivity” Wizard if you plan on adding a public subnet for a Bastion, or “VCN Only” if you’re comfortable configuring everything manually.

2. **Create a Private Subnet**:  
   - Within your new VCN, click on **Subnets** > **Create Subnet**.  
   - Choose **Private Subnet**.  
   - CIDR block example: `10.0.2.0/24` (just ensure it doesn’t overlap with any existing networks).

3. **Security Lists or NSGs**:  
   - For **Security Lists** or **Network Security Groups** attached to your private subnet, add the following rules (inbound):
     - **SSH** (TCP port 22) from your Bastion Host IP or from your on-prem IP if you plan to directly connect via an OCI Bastion session.
     - (Optional) If you wish to expose certain ports from MicroShift externally (e.g., an application running on NodePort), you must explicitly allow these in the Security List or NSG. However, to keep the cluster private, do not open these ports publicly.

   - Egress rules can be left open (default) so that the node can pull container images from the internet. Alternatively, use Oracle’s NAT Gateway for private subnets if you want a higher level of isolation.

4. **Internet or NAT Gateway** (Optional but Recommended):  
   - If your private subnet has no direct access to the internet, you will need a NAT Gateway to let your MicroShift node pull container images from external registries.  
   - Create a NAT Gateway, attach it to your VCN, and update the **Route Table** for your private subnet to send internet-bound traffic to the NAT Gateway.

---

## 3. Launch the Compute Instance (Ampere A1)

1. **Create a Compute Instance**:  
   - Navigate to **Compute** > **Instances** in the OCI Console.  
   - Click **Create Instance** and enter a name (e.g. `microshift-node`).  
   - **Choose Shape**:  
     - Select “Arm Ampere” shape (VM.Standard.A1.Flex).  
     - For the Free Tier limit, you can use up to 4 OCPUs and 24 GB RAM total. MicroShift runs fine in 2 OCPUs / 8–12 GB for testing.  
   - **Choose Image**:  
     - Select RHEL 9, Fedora 38, or CentOS 9 Stream.  
     - If RHEL is selected, ensure you have a valid developer subscription or method to register the system.  

2. **Instance Configuration**:  
   - **Subnet**: Select the **private subnet** you created in the previous step.  
   - **Assign public IPv4 address**: Uncheck (you do **not** want a public IP for a strictly private node).  
   - **SSH Keys**: Provide or generate an SSH key pair for accessing the node.

3. **Advanced Networking** (Optional):  
   - If you want to keep the instance truly private, no changes are needed. Just ensure you have Bastion / NAT in place for updates, SSH, etc.

4. **Click Create**:  
   - Wait until the instance provisioning is complete.

---

## 4. (Optional) Set Up an OCI Bastion or Public Bastion Host

If you selected **no public IP** on your MicroShift instance, you can still access it via:

1. **OCI Bastion Service**:  
   - Go to **Identity & Security** > **Bastion** in OCI.  
   - **Create Bastion** in the same compartment and region as your private subnet.  
   - Create a **SSH session** or a **Port forwarding session** that targets your MicroShift node’s private IP.  
   - Connect using the Bastion’s provided `ssh` command.  

2. **Public Bastion Host** (if you prefer a separate compute instance in the public subnet):  
   - Launch a small VM in the public subnet with minimal firewall rules.  
   - SSH to the public bastion, then SSH from there into the MicroShift node in the private subnet.

---

## 5. Prepare the Operating System

Once on the MicroShift node (via SSH):

1. **Update Packages**:
   ```bash
   sudo dnf update -y
   ```
   or
   ```bash
   sudo yum update -y
   ```
   (depending on your distro)

2. **Enable Required Repos** (Example for Fedora):
   ```bash
   sudo dnf install -y dnf-plugins-core
   ```
   (For RHEL, ensure you have your subscription attached and have the relevant repos enabled.)

3. **Install Container Runtime** (if using Podman or CRI-O):
   - MicroShift can use CRI-O under the hood, which will be installed automatically with MicroShift on supported RHEL/Fedora/CentOS versions.  
   - Ensure `podman` is installed if you want to do local container tasks.  

---

## 6. Install and Configure MicroShift

### 6.1. Install MicroShift

- **Fedora / CentOS Stream / RHEL** (9+ recommended):

  ```bash
  # Add the MicroShift repository:
  # Fedora example:
  sudo dnf copr enable -y @redhat-et/microshift

  # Install MicroShift:
  sudo dnf install -y microshift
  ```

  > For RHEL, the steps can vary. Check official MicroShift docs or Red Hat documentation for the correct repos:
  > https://microshift.io/docs

### 6.2. Configure MicroShift

MicroShift relies on a configuration file at `/etc/microshift/config.yaml`. For a minimal single-node cluster, defaults usually suffice, but you can add custom settings, for example to tweak domain names, NodePorts, etc.

A sample minimal config file:
```yaml
apiVersion: microshift.openshift.io/v1alpha1
kind: MicroShiftConfig
network:
  clusterNetwork:
  - cidr: 10.42.0.0/16
  serviceNetwork:
  - cidr: 10.43.0.0/16
```

If you wish to customize certificates, auditing, or the node hostname, see the [official MicroShift docs](https://microshift.io/docs).  

### 6.3. Enable and Start MicroShift

```bash
sudo systemctl enable microshift --now
```

- Monitor logs to confirm MicroShift is starting properly:
  ```bash
  sudo journalctl -u microshift -f
  ```
- The first startup will take a few minutes as it pulls container images for Kubernetes control-plane and related components.

---

## 7. Verify the MicroShift Cluster

1. **Check Node and Pods**:
   ```bash
   export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig
   kubectl get nodes
   kubectl get pods -A
   ```
   - You should see one node in `Ready` state.
   - System pods in namespaces like `kube-system`, `openshift-...` should be `Running`.

2. **Deploy a Test Pod**:
   ```bash
   kubectl create deployment hello-microshift --image=nginx
   kubectl expose deployment hello-microshift --port=80 --type=NodePort
   ```
   - Confirm the Pod and Service are running:
     ```bash
     kubectl get pods
     kubectl get svc
     ```

3. **Access the Application**:
   - Because this is a private cluster, you can only reach it from within the subnet (or via SSH tunnel).
   - Use SSH port forwarding or Bastion port forwarding:
     ```bash
     ssh -i /path/to/ssh_key -L 8080:<MicroShiftNodeIP>:<NodePort> opc@<BastionPublicIP>
     ```
     Then browse `http://127.0.0.1:8080` locally to see the NGINX welcome page.

---

## 8. Hardening and Security Tips

1. **Firewall**:
   - If your image uses `firewalld`, ensure rules restrict inbound traffic to only needed ports. For a private cluster with SSH only, keep ports like 22 open.  
   - If you open NodePorts for external testing, add them to the allowed inbound rules carefully.

2. **Minimal Services**:
   - Disable unnecessary system services on the node.  
   - Keep the system updated regularly.  

3. **Limit Access**:
   - Restrict SSH to known IP addresses.  
   - Use an OCI Bastion session with strict IP address whitelisting or short-lived sessions.  

4. **Use NAT Gateway** (instead of a public IP on the node):
   - This ensures the node can pull container images without directly exposing itself to the internet.

5. **TLS / Certificates**:
   - If you’re running routes or ingress with TLS, use valid certificates or trusted self-signed certs for internal usage.  

6. **RBAC**:
   - MicroShift includes Kubernetes RBAC. Restrict admin privileges (`cluster-admin`) to only necessary users or service accounts.

---

## 9. (Optional) Persist Data with an OCI Block Volume

If you want persistent storage for your workloads:

1. **Create Block Volume** in OCI (size 50GB, for example).  
2. **Attach Block Volume** to your MicroShift VM instance (Paravirtualized or iSCSI).  
3. **Format and Mount** the block volume inside the VM.  
4. **Use a PersistentVolume** in MicroShift by referencing the new volume or path.  

---

## 10. Wrap-Up and Next Steps

- You now have a **single-node MicroShift** cluster running in a **private subnet** on OCI’s Free Tier Ampere instance.  
- Applications can be deployed as Pods, Deployments, or Services via standard Kubernetes manifests.  
- You can further refine networking (e.g., bridging in a load balancer if needed for test traffic), expand to multi-node MicroShift clusters, or integrate with advanced OpenShift features.

**MicroShift Resources**:  
- [Official MicroShift Documentation](https://microshift.io/docs)  
- [GitHub Repo: MicroShift](https://github.com/openshift/microshift)

**OCI Resources**:  
- [OCI Documentation](https://docs.oracle.com/en-us/iaas/Content/home.htm)  
- [OCI Bastion Overview](https://docs.oracle.com/en-us/iaas/Content/Bastion/home.htm)

---

### Congratulations!

You have a secure, private MicroShift cluster in OCI. Keep your OS and MicroShift up to date, review security best practices, and enjoy building containerized applications at the edge with minimal overhead!

**Swapan**: do all of this including optional items via terraform

**ChatGPT**: **Assembling the infrastructure**

I'm preparing to set up a private MicroShift cluster on Oracle Cloud using Terraform, covering VCN, subnet, NAT gateway, Bastion host, compute instance, OS selection, security rules, and MicroShift installation.

**Mapping the Terraform plan**

I’m outlining the Terraform configuration to create a private MicroShift cluster on Oracle Cloud, covering VCN, subnet, NAT gateway, security lists, and compute instance.

**Automating setup**

I’m outlining a Terraform configuration for an Oracle Cloud instance to streamline MicroShift setup using cloud-init scripts, ensuring effortless OS preparation and installation automation.

**Setting up the environment**

I’m detailing the configuration for an Oracle Cloud instance, specifying the Ampere shape, OCPUs, memory, and the image ID for Oracle Linux 9.

**Mapping out the plan**

I’m thinking through a simplified Terraform configuration for Oracle Cloud, using conditionals to manage optional NAT or Bastion setup, aiming for a balance between clarity and flexibility.

**Refining the setup**

I've decided to streamline Terraform configuration to a single file, adding comments for optional elements like NAT or Bastion. Finalizing with terraform commands for easy execution.

**ChatGPT**: Below is an **example** Terraform configuration that demonstrates how you could stand up the following **in Oracle Cloud (OCI) Free Tier**:

- A **VCN** with:
  - **Private Subnet** (for the MicroShift node)
  - (Optional) **Public Subnet** (for a separate public Bastion host)
  - (Optional) **NAT Gateway** for outbound traffic  
- **Network Security Group (NSG)** or **Security Lists** to allow SSH (and optionally other ports)
- A **Compute Instance** on **Ampere A1** shape, running **Fedora / RHEL / CentOS** (example below uses Oracle Linux or Fedora).  
  - **No public IP** on the MicroShift node
  - A **cloud-init** script to install & start MicroShift automatically
- (Optional) **OCI Bastion** for private SSH access

> **Important**:
> 1. This is just a sample Terraform config. Adjust **OCIDs**, shapes, region, images, `cidr_blocks`, etc., to fit your environment.  
> 2. You must configure the [OCI Terraform Provider authentication](https://registry.terraform.io/providers/oracle/oci/latest/docs#configuring-the-provider) (e.g., environment variables for `TF_VAR_tenancy_ocid`, `TF_VAR_user_ocid`, etc.) or specify them in your code.  
> 3. **Fedora** or **RHEL/CentOS** is recommended for MicroShift. Oracle Linux 9 can also work, but you may need to adapt the installation steps.  
> 4. Oracle’s **Ampere A1 Free Tier** limits: up to 4 OCPUs and 24 GB RAM total. Adjust shape_config accordingly.  
> 5. This example is fairly verbose. You can split into multiple `.tf` files (e.g., `variables.tf`, `main.tf`, etc.) as you prefer.

---

## File: `main.tf`

```hcl
###############################################################################
# Provider Configuration
###############################################################################
provider "oci" {
  # Authentication can be done via environment variables or here directly.
  # E.g., export TF_VAR_tenancy_ocid, TF_VAR_user_ocid, ...
  tenancy_ocid         = var.tenancy_ocid
  user_ocid            = var.user_ocid
  fingerprint          = var.fingerprint
  private_key_path     = var.private_key_path
  region               = var.region
}

###############################################################################
# Variables
###############################################################################
variable "tenancy_ocid" {}
variable "user_ocid" {}
variable "fingerprint" {}
variable "private_key_path" {}
variable "region" {
  default = "us-ashburn-1"
}

# The Compartment in which to create all resources (must exist).
variable "compartment_ocid" {}

# Customize these CIDRs for your environment
variable "vcn_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
variable "private_subnet_cidr" {
  type    = string
  default = "10.0.2.0/24"
}
variable "public_subnet_cidr" {
  type    = string
  default = "10.0.1.0/24"
}

# Toggle creation of optional resources
variable "create_public_subnet" {
  type    = bool
  default = false
}
variable "create_nat_gateway" {
  type    = bool
  default = true
}
variable "create_bastion_service" {
  type    = bool
  default = false
}

###############################################################################
# 1. Create VCN
###############################################################################
resource "oci_core_vcn" "microshift_vcn" {
  compartment_id = var.compartment_ocid
  display_name   = "microshift_vcn"
  cidr_block     = var.vcn_cidr
}

###############################################################################
# 2. Internet Gateway (only if needed)
###############################################################################
resource "oci_core_internet_gateway" "ig" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.microshift_vcn.id
  display_name   = "microshift_ig"
  enabled        = var.create_public_subnet
  # If create_public_subnet = false, this resource still gets created but is disabled.
  # Alternatively, you can wrap this entire resource in a 'count' or 'for_each' if you want more control.
}

###############################################################################
# 3. NAT Gateway (optional but recommended for private subnet)
###############################################################################
resource "oci_core_nat_gateway" "ngw" {
  count          = var.create_nat_gateway ? 1 : 0
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.microshift_vcn.id
  display_name   = "microshift_nat_gateway"
}

###############################################################################
# 4. Route Table for Private Subnet - direct traffic to NAT Gateway for external access
###############################################################################
resource "oci_core_route_table" "private_routetable" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.microshift_vcn.id
  display_name   = "private_routetable"

  route_rules = [
    # Only add NAT route if we're creating the NAT gateway
    var.create_nat_gateway ? {
      cidr_block         = "0.0.0.0/0"
      network_entity_id  = oci_core_nat_gateway.ngw[0].id
      description        = "Default route to NAT for private subnet"
    } : null
  ]
  # Terraform will ignore the null route. Alternatively, you can do a count/for_each approach.
}

###############################################################################
# 5. Route Table for Public Subnet (if used) - direct Internet traffic to IG
###############################################################################
resource "oci_core_route_table" "public_routetable" {
  count          = var.create_public_subnet ? 1 : 0
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.microshift_vcn.id
  display_name   = "public_routetable"

  route_rules = [
    {
      cidr_block        = "0.0.0.0/0"
      network_entity_id = oci_core_internet_gateway.ig.id
      description       = "Default route to Internet Gateway"
    }
  ]
}

###############################################################################
# 6. Private Subnet
###############################################################################
resource "oci_core_subnet" "private_subnet" {
  compartment_id                  = var.compartment_ocid
  vcn_id                          = oci_core_vcn.microshift_vcn.id
  display_name                    = "private_subnet"
  cidr_block                      = var.private_subnet_cidr
  prohibit_public_ip_on_vnic      = true
  route_table_id                  = oci_core_route_table.private_routetable.id
  dns_label                       = "privsub"
}

###############################################################################
# 7. Public Subnet (Optional)
###############################################################################
resource "oci_core_subnet" "public_subnet" {
  count                           = var.create_public_subnet ? 1 : 0
  compartment_id                  = var.compartment_ocid
  vcn_id                          = oci_core_vcn.microshift_vcn.id
  display_name                    = "public_subnet"
  cidr_block                      = var.public_subnet_cidr
  prohibit_public_ip_on_vnic      = false
  route_table_id                  = oci_core_route_table.public_routetable[0].id
  dns_label                       = "pubsub"
}

###############################################################################
# 8. Network Security Group (NSG) for Private Subnet
###############################################################################
resource "oci_core_network_security_group" "private_nsg" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.microshift_vcn.id
  display_name   = "private_nsg"
}

# - Inbound rule for SSH. Adjust source as needed (e.g., Bastion subnet range, your IP, etc.)
resource "oci_core_network_security_group_security_rule" "private_nsg_ssh" {
  network_security_group_id = oci_core_network_security_group.private_nsg.id
  direction                 = "INGRESS"
  protocol                  = "6"   # TCP
  source_type               = "CIDR_BLOCK"
  source                    = "10.0.1.0/24" # If you're using a public subnet for a Bastion, or your corporate VPN range, etc.
  tcp_options {
    destination_port_range {
      min = 22
      max = 22
    }
  }
  description = "Allow SSH from Bastion or known IP range"
}

# Optionally add NodePort or other inbound rules if you want to expose apps internally.
# For a strictly private cluster, keep it minimal.

###############################################################################
# 9. Data Source for an OS Image (Example: Fedora or Oracle Linux)
###############################################################################
# Oracle-provided images can be looked up via data sources or the web console.
# Example: Oracle Linux 9. Or use "Fedora" if available in the region.
# For brevity, let's demonstrate Oracle Linux 9
data "oci_core_images" "oracle_linux_9" {
  compartment_id   = var.compartment_ocid
  operating_system = "Oracle Linux"
  operating_system_version = "9"
}

###############################################################################
# 10. Compute Instance (MicroShift Node, Ampere A1)
###############################################################################
resource "oci_core_instance" "microshift_instance" {
  compartment_id       = var.compartment_ocid
  display_name         = "microshift-node"
  availability_domain  = data.oci_identity_availability_domains.ads.availability_domains[0].name

  # Ampere A1 shape for Free Tier
  shape = "VM.Standard.A1.Flex"
  shape_config {
    ocpus        = 2   # Adjust for your needs / Free Tier
    memory_in_gbs = 12
  }

  # We place this in the private subnet
  create_vnic_details {
    subnet_id        = oci_core_subnet.private_subnet.id
    display_name     = "microshift-vnic"
    assign_public_ip = false
    nsg_ids          = [oci_core_network_security_group.private_nsg.id]
  }

  # SSH key and cloud-init script
  metadata = {
    ssh_authorized_keys = file("~/.ssh/id_rsa.pub")
    user_data           = base64encode(templatefile("cloud-init.tpl", {}))
  }

  # Use the first image from the data source
  source_details {
    source_type = "image"
    source_id   = data.oci_core_images.oracle_linux_9.images[0].id
  }
}

# (We need to fetch the list of Availability Domains.)
data "oci_identity_availability_domains" "ads" {
  compartment_id = var.tenancy_ocid
}

###############################################################################
# 11. (Optional) Create a Bastion Service to SSH into Private Subnet
###############################################################################
resource "oci_bastion_bastion" "bastion_service" {
  count          = var.create_bastion_service ? 1 : 0
  compartment_id = var.compartment_ocid
  name           = "tf-bastion-service"
  target_vcn_id  = oci_core_vcn.microshift_vcn.id
  target_subnet_id = oci_core_subnet.private_subnet.id
  # The Bastion service can connect directly to private hosts in that subnet
}

# If you want a public Bastion host (Compute instance in the public subnet), create a second instance with count = var.create_public_subnet, etc.

###############################################################################
# Outputs
###############################################################################
output "microshift_private_ip" {
  description = "Private IP of the MicroShift node"
  value       = oci_core_instance.microshift_instance.public_ip
  # For private IP, we can also use private_ip instead of public_ip
  # But note: there's no public IP assigned. 
  # So you can see it under the instance's VNIC details in the console or via 'tf output'.
}

output "bastion_ocid" {
  description = "OCID of Bastion service (if created)"
  value       = length(oci_bastion_bastion.bastion_service) > 0 ? oci_bastion_bastion.bastion_service[0].id : ""
}
```

---

## File: `cloud-init.tpl`

A simple **cloud-init** script to install and start MicroShift on (for example) Oracle Linux 9 or Fedora. You can tailor to RHEL/CentOS or other distributions:

```bash
#!/bin/bash

# Update system
sudo dnf -y update

# Add dependencies
sudo dnf -y install dnf-plugins-core

# If using Fedora or CentOS Stream 9 with a Copr:
# sudo dnf copr enable -y @redhat-et/microshift

# If Oracle Linux 9, we may need to add the MicroShift repo from microshift.io instructions:
# (Example repo steps for Oracle Linux might differ; adapt as needed)

# Example for Fedora or CentOS Stream (comment out if on OL):
# sudo dnf copr enable -y @redhat-et/microshift

# Attempt installing MicroShift (the official route depends on your distro)
# For demonstration, let's assume we have a working repo or manual RPM steps:
sudo dnf -y install microshift

# Enable and start MicroShift
sudo systemctl enable microshift --now

# Optionally set KUBECONFIG for root so future logins can run kubectl without specifying the path
echo "export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig" | sudo tee -a /root/.bashrc

# Done
```

Adjust the steps in `cloud-init.tpl` based on your **exact distro** and the [MicroShift Docs](https://microshift.io/docs). For RHEL, you’ll need valid subscription entitlements or Developer Subscription steps. For Fedora or CentOS Stream, the Copr is typically straightforward.

---

# Usage

1. **Create/Review** the above files:  
   - `main.tf` (Terraform config)  
   - `cloud-init.tpl` (your user data script)  
2. **Export** or define the needed environment variables (or set them in `terraform.tfvars`), e.g.:
   ```bash
   export TF_VAR_tenancy_ocid="ocid1.tenancy.oc1..xxxxx"
   export TF_VAR_user_ocid="ocid1.user.oc1..xxxxx"
   export TF_VAR_fingerprint="xx:xx:xx:xx..."
   export TF_VAR_private_key_path="~/.oci/oci_api_key.pem"
   export TF_VAR_compartment_ocid="ocid1.compartment.oc1..xxxxx"
   ```
3. **terraform init**  
4. **terraform plan**  
5. **terraform apply** (review the plan, then type `yes` to proceed)  

Terraform will create:  
- VCN, Subnets, Route Tables, NAT Gateway (optional), Bastion (optional)  
- A Private Compute instance with cloud-init installing MicroShift.  

Once complete, if you opted for the **OCI Bastion service**, you can create a Bastion **SSH port forwarding session** in the OCI Console to reach your new MicroShift node’s private IP and SSH in. Alternatively, if you created a **public Bastion host** (another compute instance in the public subnet), you can SSH into that, then into the private node.

---

## Verifying MicroShift

1. **SSH** into the MicroShift node (via Bastion or any private connection).  
2. Run:
   ```bash
   sudo su -
   export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig
   kubectl get nodes
   kubectl get pods -A
   ```
3. Optionally deploy a test workload:
   ```bash
   kubectl create deployment hello --image=nginx
   kubectl expose deployment hello --port=80 --type=NodePort
   kubectl get svc hello
   ```
4. Use SSH tunneling or Bastion port forwarding to hit the NodePort from your local machine if needed.

---

## Next Steps

- Customize firewall rules, **NSGs**, or **Security Lists** for your environment.  
- Attach an **OCI Block Volume** for persistent storage, then define `PersistentVolumes` in MicroShift.  
- Regularly **patch** both the OS and MicroShift.  
- Explore advanced [MicroShift configuration](https://microshift.io/docs) for certificate management, custom domain, and more.

With this Terraform template (plus the optional Bastion), you’ll have an **end-to-end** secure and private MicroShift setup on OCI’s Free Tier Ampere A1, with minimal public exposure. Happy automating!

**Swapan**: show a full bash script version instead of terraform

**ChatGPT**: **Showcasing bash script**

We should illustrate a comprehensive bash script using the "oci" command line tool, covering VCN creation, subnets, NAT gateway, optional bastion setup, and microShift node inclusion.

**Crafting the script**

I’m working on a bash script to create a VCN, subnets, NAT gateway, route table, security list, instance, and optional bastion. The script details installation, environment setup, and resource creation.

**Clarifying instance options**

I’m outlining steps for launching instances, including using the OCI bastion service or a public subnet for the Bastion, with specific instructions for connecting and encoding user_data.

**Assembling the final script**

I’m crafting a bash script for VCN setup, subnets, route tables, NAT gateways, and instances. Instructions for user_data encoding and optional Bastion setup are included.

**ChatGPT**: Below is an **example Bash script** (using the **OCI CLI**) that creates a **private MicroShift node** on an **Ampere A1** shape in the **OCI Free Tier**. It covers:

1. **VCN & Private Subnet**  
2. **NAT Gateway** (for outbound internet)  
3. **Route Table** for the private subnet  
4. **Network Security Group** (NSG) with SSH ingress rule  
5. **Compute Instance** with a cloud-init script to install and start MicroShift (on Oracle Linux or Fedora, as a demonstration)

> **Important**  
> - You **must** install and configure the [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/cliconcepts.htm) first.  
> - You need `jq` installed for JSON parsing.  
> - This script is just an example and may need customization for your environment (e.g., compartment OCIDs, region, shapes, images).  
> - Error handling is minimal. Use at your own risk.  
> - For a **Bastion** setup, see the optional snippet at the end of this script.

---

## Example Bash Script

```bash
#!/usr/bin/env bash
set -euo pipefail

###############################################################################
# REQUIRED ENVIRONMENT VARIABLES
###############################################################################
# Adjust these values or set them before running (export VAR=...).
: "${COMPARTMENT_OCID:?Please set COMPARTMENT_OCID}"
: "${DISPLAY_NAME:=microshift-demo}"
: "${REGION:=us-ashburn-1}"
: "${VCN_CIDR:=10.0.0.0/16}"
: "${PRIVATE_SUBNET_CIDR:=10.0.2.0/24}"
: "${SSH_PUBLIC_KEY_PATH:=~/.ssh/id_rsa.pub}"
# AMPERE Free Tier: up to 4 OCPUs / 24GB across all instances
: "${OCPUS:=2}"
: "${MEMORY_GBS:=12}"

# (Optional) If you'd rather specify an image OCID directly:
# : "${IMAGE_OCID:=ocid1.image.oc1...}"

###############################################################################
# 1. Create VCN
###############################################################################
echo ">>> Creating VCN..."
VCN_NAME="${DISPLAY_NAME}-vcn"
vcn_json=$(oci network vcn create \
  --compartment-id "$COMPARTMENT_OCID" \
  --display-name "$VCN_NAME" \
  --cidr-block "$VCN_CIDR" \
  --region "$REGION" \
  --wait-for-state AVAILABLE \
  --query 'data' --raw-output)

VCN_ID=$(echo "$vcn_json" | jq -r '.id')
echo "VCN created with OCID: $VCN_ID"

###############################################################################
# 2. Create NAT Gateway (for private subnet's outbound internet)
###############################################################################
echo ">>> Creating NAT Gateway..."
NGW_NAME="${DISPLAY_NAME}-nat-gw"
ngw_json=$(oci network nat-gateway create \
  --compartment-id "$COMPARTMENT_OCID" \
  --vcn-id "$VCN_ID" \
  --display-name "$NGW_NAME" \
  --region "$REGION" \
  --wait-for-state AVAILABLE \
  --query 'data' --raw-output)

NGW_ID=$(echo "$ngw_json" | jq -r '.id')
echo "NAT Gateway created with OCID: $NGW_ID"

###############################################################################
# 3. Create Route Table for Private Subnet
###############################################################################
echo ">>> Creating Route Table for private subnet..."
RT_NAME="${DISPLAY_NAME}-private-rt"
private_rt_json=$(oci network route-table create \
  --compartment-id "$COMPARTMENT_OCID" \
  --vcn-id "$VCN_ID" \
  --display-name "$RT_NAME" \
  --route-rules "[{\"cidrBlock\":\"0.0.0.0/0\",\"networkEntityId\":\"${NGW_ID}\"}]" \
  --region "$REGION" \
  --wait-for-state AVAILABLE \
  --query 'data' --raw-output)

PRIVATE_RT_ID=$(echo "$private_rt_json" | jq -r '.id')
echo "Private Route Table created with OCID: $PRIVATE_RT_ID"

###############################################################################
# 4. Create Private Subnet
###############################################################################
echo ">>> Creating Private Subnet..."
PRIV_SUBNET_NAME="${DISPLAY_NAME}-private-subnet"
priv_subnet_json=$(oci network subnet create \
  --compartment-id "$COMPARTMENT_OCID" \
  --vcn-id "$VCN_ID" \
  --display-name "$PRIV_SUBNET_NAME" \
  --cidr-block "$PRIVATE_SUBNET_CIDR" \
  --dns-label "privsub" \
  --route-table-id "$PRIVATE_RT_ID" \
  --prohibit-public-ip-on-vnic true \
  --wait-for-state AVAILABLE \
  --query 'data' --raw-output)

PRIV_SUBNET_ID=$(echo "$priv_subnet_json" | jq -r '.id')
echo "Private Subnet created with OCID: $PRIV_SUBNET_ID"

###############################################################################
# 5. Create Network Security Group (NSG) for Private Subnet
###############################################################################
echo ">>> Creating Network Security Group..."
NSG_NAME="${DISPLAY_NAME}-private-nsg"
nsg_json=$(oci network nsg create \
  --compartment-id "$COMPARTMENT_OCID" \
  --vcn-id "$VCN_ID" \
  --display-name "$NSG_NAME" \
  --region "$REGION" \
  --query 'data' \
  --wait-for-state AVAILABLE \
  --raw-output)

NSG_ID=$(echo "$nsg_json" | jq -r '.id')
echo "Private NSG created with OCID: $NSG_ID"

echo ">>> Adding Ingress Rule for SSH (port 22) to NSG..."
# Adjust the source as needed: e.g., if you have a known VPN range or a Bastion subnet
oci network nsg rules add \
  --network-security-group-id "$NSG_ID" \
  --security-rules '[
    {
      "direction": "INGRESS",
      "protocol": "6",
      "source": "10.0.0.0/16",
      "sourceType": "CIDR_BLOCK",
      "tcpOptions": {
        "destinationPortRange": {
          "min": 22,
          "max": 22
        }
      },
      "description": "Allow SSH from 10.0.0.0/16"
    }
  ]' \
  --region "$REGION"

###############################################################################
# 6. Prepare Cloud-Init (User Data) for MicroShift Installation
###############################################################################
# Example for Oracle Linux 9 or Fedora. Adjust as needed for your distro or MicroShift repo steps.
# We'll embed a simple user_data script in a variable (then base64-encode it).
cat > /tmp/microshift-cloud-init.sh <<'EOF'
#!/bin/bash
set -e

# For Oracle Linux or Fedora
sudo dnf -y update
sudo dnf -y install dnf-plugins-core jq

# Example: If on Fedora or CentOS Stream, you could enable the Copr:
# sudo dnf copr enable -y @redhat-et/microshift

# On Oracle Linux, you might need to manually add a MicroShift repo
# or use the official instructions from microshift.io.
# For demonstration, let's assume the MicroShift package is available:
sudo dnf -y install microshift

# Enable & start MicroShift
sudo systemctl enable microshift --now

# Configure kubectl for root so that each SSH login doesn't need to set it manually
echo "export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig" | sudo tee -a /root/.bashrc

EOF

USER_DATA_BASE64=$(base64 /tmp/microshift-cloud-init.sh | tr -d '\n')

###############################################################################
# 7. Launch Compute Instance (Ampere A1) in the Private Subnet
###############################################################################
echo ">>> Launching Compute Instance (Ampere A1)..."
INSTANCE_NAME="${DISPLAY_NAME}-microshift-node"
SSH_KEY=$(cat "$SSH_PUBLIC_KEY_PATH")

# If you prefer a specific image, you can do: --image-id "$IMAGE_OCID"
# Otherwise we can search for Oracle Linux 9 image in your region. 
#  (See https://docs.oracle.com/en-us/iaas/Content/Compute/References/images.htm
#   or "oci compute image list" for more info.)
# For brevity, we'll do a quick filter with 'oci compute image list'
# WARNING: This might pick the first match it finds. Ideally, you'd pin to a known image OCID.
echo "Looking up an Oracle Linux 9 image in $REGION..."
image_search_json=$(oci compute image list \
  --compartment-id "$COMPARTMENT_OCID" \
  --operating-system "Oracle Linux" \
  --operating-system-version "9" \
  --shape "VM.Standard.A1.Flex" \
  --sort-order DESC \
  --region "$REGION" \
  --query 'data[0].id' --raw-output)
IMAGE_ID="$image_search_json"

echo "Using Image OCID: $IMAGE_ID"

instance_json=$(oci compute instance launch \
  --compartment-id "$COMPARTMENT_OCID" \
  --display-name "$INSTANCE_NAME" \
  --availability-domain "$(oci iam availability-domain list --compartment-id "$COMPARTMENT_OCID" --query 'data[0].name' --raw-output)" \
  --shape "VM.Standard.A1.Flex" \
  --shape-config "{\"ocpus\": $OCPUS, \"memoryInGBs\": $MEMORY_GBS}" \
  --subnet-id "$PRIV_SUBNET_ID" \
  --assign-public-ip false \
  --nsg-ids '["'"$NSG_ID"'"]' \
  --image-id "$IMAGE_ID" \
  --metadata "{ \"ssh_authorized_keys\": \"$SSH_KEY\", \"user_data\": \"$USER_DATA_BASE64\" }" \
  --region "$REGION" \
  --wait-for-state RUNNING \
  --query 'data' \
  --raw-output)

INSTANCE_ID=$(echo "$instance_json" | jq -r '.id')
echo "Compute Instance launched with OCID: $INSTANCE_ID"

###############################################################################
# DONE
###############################################################################
echo "================================================================"
echo " All resources created. Private MicroShift node is launching..."
echo "================================================================"
echo " VCN OCID:          $VCN_ID"
echo " Private Subnet:    $PRIV_SUBNET_ID"
echo " NAT Gateway:       $NGW_ID"
echo " NSG:               $NSG_ID"
echo " Instance ID:       $INSTANCE_ID"
echo
echo "The instance does NOT have a public IP. To SSH in, you can:"
echo "  1) Use OCI Bastion, or"
echo "  2) Provision a separate public 'jump' host in your VCN, or"
echo "  3) Use a VPN connection into this VCN."
echo
echo "Once inside the instance, you can run:"
echo "  sudo su -"
echo "  export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig"
echo "  kubectl get pods -A"
echo
echo "MicroShift should be running once the cloud-init completes."
echo "================================================================"
```

### Explanation & Usage

1. **Install OCI CLI** and **jq**.
2. **Set** environment variables:
   ```bash
   export COMPARTMENT_OCID="ocid1.compartment.oc1..xxxxx"
   export DISPLAY_NAME="my-microshift"
   export REGION="us-ashburn-1"
   export SSH_PUBLIC_KEY_PATH="~/.ssh/id_rsa.pub"
   ```
   Adjust them for your environment.
3. **Run** the script (e.g., `bash oci-microshift.sh`).
4. The script will:
   - Create a VCN, Private Subnet, NAT Gateway, NSG  
   - Launch an Ampere A1 instance with your SSH key and a **cloud-init** script that installs MicroShift.  

After the script finishes, you’ll have a single-node MicroShift cluster in a private subnet.

---

## Optional: Create an OCI Bastion to SSH into the Private Instance

Add something like the following at the **end** of the script to create an OCI Bastion resource. You can then create SSH sessions in the OCI Console (or CLI) to connect:

```bash
# Uncomment if you want a Bastion to connect to private IPs
#
# echo ">>> Creating OCI Bastion..."
# BASTION_NAME="${DISPLAY_NAME}-bastion"
# bastion_json=$(oci bastion bastion create \
#   --display-name "$BASTION_NAME" \
#   --compartment-id "$COMPARTMENT_OCID" \
#   --target-subnet-id "$PRIV_SUBNET_ID" \
#   --region "$REGION" \
#   --wait-for-state ACTIVE \
#   --query 'data' --raw-output)
#
# BASTION_ID=$(echo "$bastion_json" | jq -r '.id')
# echo "OCI Bastion created with OCID: $BASTION_ID"
#
# echo "Use 'oci bastion session create ...' or the OCI Console to create an SSH session to your private instance."
```

Then in the OCI console:

1. Navigate to **Identity & Security** > **Bastion**.  
2. Find your Bastion, click **Create Session**.  
3. Provide your target resource’s private IP (the instance’s VNIC private IP).  
4. Use the CLI or web console to get the SSH command.  

---

### Notes & Next Steps

- **Security**: Edit the NSG or security lists to allow only the IP ranges you trust (instead of `10.0.0.0/16`).  
- **Persistent Storage**: Attach a Block Volume if needed, format it, mount it, then create a Kubernetes `PersistentVolume`.  
- **Firewalls**: If you open NodePorts, ensure you configure inbound rules in the NSG.  
- **Maintenance**: Update the OS and MicroShift regularly (`sudo dnf update -y`).  

With this script, you’ll have a minimal **private** single-node MicroShift cluster running in OCI’s Ampere A1 Free Tier, ready for container workloads with minimal overhead!

