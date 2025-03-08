**Swapan**: script to install wls2 ubuntu update and upgrade then install docker 

**ChatGPT**: Here’s a script to install WSL2, update and upgrade the Ubuntu distribution, and then install Docker. 

You can save this script in a file, such as `install_wsl2_docker.sh`, and run it in a WSL2 terminal.

```bash
#!/bin/bash

# Script to install updates, upgrades, and Docker on Ubuntu running on WSL2

# Check if the script is being run as root
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root. Please run: sudo $0"
   exit 1
fi

echo "Starting WSL2 Ubuntu setup..."

# Step 1: Update and upgrade the system
echo "Updating package lists..."
sudo apt-get update

echo "Upgrading installed packages..."
sudo apt-get upgrade -y

echo "Performing a distribution upgrade..."
sudo apt-get dist-upgrade -y

# Step 2: Install required dependencies for Docker
echo "Installing prerequisites for Docker..."
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release

# Step 3: Add Docker's official GPG key
echo "Adding Docker's official GPG key..."
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Step 4: Add the Docker APT repository
echo "Adding Docker's APT repository..."
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: Update the package database again
echo "Updating package lists for Docker..."
sudo apt-get update

# Step 6: Install Docker Engine
echo "Installing Docker..."
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Step 7: Add the current user to the Docker group
echo "Adding user $(whoami) to the docker group..."
sudo usermod -aG docker $(whoami)

# Step 8: Verify Docker installation
echo "Verifying Docker installation..."
docker --version

# Step 9: Print final instructions
echo "Installation complete! Please log out and back in for Docker group changes to take effect."
```

### Steps to Use:
1. Save the script to a file, e.g., `install_wsl2_docker.sh`.
2. Make the script executable:
   ```bash
   chmod +x install_wsl2_docker.sh
   ```
3. Run the script with `sudo`:
   ```bash
   sudo ./install_wsl2_docker.sh
   ```
4. After installation, log out of your WSL2 session and log back in for the changes to take effect.

### Notes:
- Ensure you are using **WSL2** with Ubuntu. You can verify the WSL version with:
  ```bash
  wsl --list --verbose
  ```
- Docker Desktop on Windows also supports WSL2 integration, but this script installs Docker directly within the WSL2 environment.

