**Swapan**: install microshift on free oracle cloud vm's

**ChatGPT**: Installing MicroShift on free Oracle Cloud VM instances is a great way to experiment with lightweight Kubernetes on a cloud platform. Here's a step-by-step guide to achieve this:

---

## **Step 1: Provision Oracle Cloud VM**
1. **Create a Free VM Instance**:
   - Log in to the [Oracle Cloud Console](https://cloud.oracle.com/).
   - Go to **Compute** > **Instances** and click **Create Instance**.
   - Choose an **Always Free Eligible VM**:
     - Shape: `VM.Standard.A1.Flex` (Ampere ARM-based CPU).
     - Assign at least **2 OCPUs** and **12 GB of RAM** (as MicroShift requires a minimum of 2 CPUs and 8 GB RAM).
   - Select **Canonical Ubuntu 22.04** or **RHEL 9** as the OS (MicroShift works well with these).

2. **Configure Networking**:
   - Open ports for Kubernetes and MicroShift:
     - Go to **Networking** > **Virtual Cloud Network (VCN)** > **Security Lists**.
     - Add Ingress Rules for the following:
       - Ports: 80, 443, 6443, 2379-2380, 10250-10255, 30000-32767 (NodePort range).
       - Protocol: TCP.

---

## **Step 2: Prepare the VM**
1. **Update System Packages**:
   ```bash
   sudo apt update && sudo apt upgrade -y   # For Ubuntu
   sudo dnf update -y                       # For RHEL
   ```

2. **Install Required Dependencies**:
   ```bash
   sudo apt install -y curl tar wget iptables socat conntrack
   ```

3. **Enable IP Forwarding**:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```

---

## **Step 3: Install MicroShift**
1. **Download MicroShift**:
   - Check for the latest release on [MicroShift GitHub](https://github.com/openshift/microshift/releases).
   - Download and install:
     ```bash
     wget https://github.com/openshift/microshift/releases/download/<version>/microshift-linux-arm64.tar.gz
     tar -xvzf microshift-linux-arm64.tar.gz
     sudo mv microshift-linux-arm64 /usr/local/bin/microshift
     ```

2. **Set Up Systemd Service**:
   - Create a systemd service file for MicroShift:
     ```bash
     sudo tee /etc/systemd/system/microshift.service > /dev/null <<EOF
     [Unit]
     Description=MicroShift
     After=network.target

     [Service]
     Type=simple
     ExecStart=/usr/local/bin/microshift
     Restart=always
     RestartSec=10
     LimitNOFILE=65536

     [Install]
     WantedBy=multi-user.target
     EOF
     ```

3. **Enable and Start MicroShift**:
   ```bash
   sudo systemctl enable microshift
   sudo systemctl start microshift
   ```

4. **Check the Status**:
   ```bash
   sudo systemctl status microshift
   ```

---

## **Step 4: Configure kubectl**
1. **Install kubectl**:
   - Follow the official Kubernetes installation guide for kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

2. **Copy kubeconfig**:
   - MicroShift generates a `kubeconfig` file at `/var/lib/microshift/resources/kubeadmin/kubeconfig`.
   ```bash
   mkdir -p $HOME/.kube
   sudo cp /var/lib/microshift/resources/kubeadmin/kubeconfig $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. **Test kubectl**:
   ```bash
   kubectl get nodes
   ```

---

## **Step 5: Expose MicroShift Services**
1. **Access Kubernetes Dashboard**:
   - Use kubectl to expose the Kubernetes Dashboard or other services.
   - Use `NodePort` to access services externally through the VM's public IP.

2. **Port Forwarding Example**:
   ```bash
   kubectl port-forward service/kubernetes-dashboard -n kubernetes-dashboard 8080:443
   ```

---

## **Step 6: Optional - Deploy Sample Application**
1. Create a deployment:
   ```bash
   kubectl create deployment hello-world --image=nginx
   kubectl expose deployment hello-world --type=NodePort --port=80
   ```

2. Access the application:
   - Get the external IP of the VM and access the app on the assigned NodePort:
     ```bash
     kubectl get svc
     ```

---

## **Troubleshooting**
- **Logs**: Check logs for errors:
  ```bash
  sudo journalctl -u microshift -f
  ```
- **Firewall**: Ensure Oracle Cloud's ingress rules are properly set up.

With these steps, you should have MicroShift running on an Oracle Cloud free VM. Let me know if you encounter any issues!

