**Swapan**: for the following code how do set the non root user password?

#!/usr/bin/bash

# Exit on any error
set -e

FORCE_REINSTALL=false

# Function to display usage and help
usage() {
    echo "========================================="
    echo "Docker, OpenShift CLI, and VS Code Installer Script"
    echo "========================================="
    echo "Usage: $0 [options]"
    echo
    echo "Options:"
    echo "  --update                  Update and upgrade the system."
    echo "  --create-user <username>  Create a non-root user and add them to the 'docker' and 'sudo' groups."
    echo "                            If no username is provided, it will prompt interactively."
    echo "  --install docker          Install Docker and configure permissions for the current user."
    echo "  --install oc              Install the latest OpenShift CLI (oc)."
    echo "  --install vscode          Install or update Visual Studio Code."
    echo "  --all                     Perform all tasks in sequence (update system, create user, install Docker, OpenShift CLI, and VS Code)."
    echo "  --force                   Force reinstall Docker and OpenShift CLI (oc) even if they are already installed."
    echo "  --help                    Show this help message."
    echo
    echo "========================================="
    echo "Examples:"
    echo
    echo "  1. To update and upgrade the system:"
    echo "      $0 --update"
    echo
    echo "  2. To create a non-root user named 'developer':"
    echo "      $0 --create-user developer"
    echo
    echo "  3. To install Docker:"
    echo "      $0 --install docker"
    echo
    echo "  4. To install the latest OpenShift CLI (oc):"
    echo "      $0 --install oc"
    echo
    echo "  5. To install or update Visual Studio Code:"
    echo "      $0 --install vscode"
    echo
    echo "  6. To force reinstall Docker and OpenShift CLI (oc):"
    echo "      $0 --force --install docker --install oc"
    echo
    echo "  7. To update the system, create a user, and install Docker:"
    echo "      $0 --update --create-user developer --install docker"
    echo
    echo "  8. To perform all tasks (update, create a user, and install all tools):"
    echo "      $0 --all"
    echo
    echo "  9. To install all tools with forced reinstall for Docker and OpenShift CLI:"
    echo "      $0 --all --force"
    echo "========================================="
}


# Function to update and upgrade the system
update_system() {
    echo "Updating and upgrading Ubuntu..."
    echo "Checking for available updates..."
    if sudo apt-get -s upgrade | grep -q 'upgraded,'; then
        echo "Updates are available. Proceeding with update and upgrade..."
        sudo apt update
        sudo apt -y upgrade
    else
        echo "No updates available. Skipping update and upgrade."
    fi
}

# Function to uninstall Docker
uninstall_docker() {
    echo "Uninstalling Docker..."
    sudo apt purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo apt autoremove -y
    sudo rm -rf /var/lib/docker
    sudo rm -rf /etc/docker
    echo "Docker has been uninstalled."
}

# Function to install Docker
install_docker_logic() {
    echo "Installing prerequisites for Docker..."
    sudo apt install -y ca-certificates curl gnupg lsb-release tar

    echo "Adding Docker's official GPG key..."
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo "Setting up Docker repository..."
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    echo "Updating package index and installing Docker..."
    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    echo "Verifying Docker installation..."
    if docker --version; then
        echo "Docker installed successfully."
    else
        echo "Docker installation failed."
        exit 1
    fi
}

# Function to enable and start Docker service
enable_docker_service() {
    echo "Enabling and starting Docker service..."
    sudo service docker start
    sudo systemctl enable docker

    TIMEOUT=30
    INTERVAL=2
    ELAPSED=0

    while [ $ELAPSED -lt $TIMEOUT ]; do
        DOCKER_STATUS=$(sudo service docker status)
        if echo "$DOCKER_STATUS" | grep -q "running"; then
            echo "Docker service is running."
            return 0
        fi
        sleep $INTERVAL
        ELAPSED=$((ELAPSED + INTERVAL))
    done

    echo "Docker service not running after $TIMEOUT seconds. Exiting."
    return 1
}

# Function to create a new non-root user or verify group membership
create_non_root_user() {
    if [ -z "$1" ]; then
        read -r -p "Enter the username for the non-root user: " NEW_USER
    else
        NEW_USER="$1"
    fi

    if [ -z "$NEW_USER" ]; then
        echo "Error: Username cannot be empty. Please provide a valid username."
        exit 1
    fi

    if id "$NEW_USER" &>/dev/null; then
        echo "User $NEW_USER already exists."
        
        # Check if the user belongs to the 'docker' group
        if groups "$NEW_USER" | grep -qw "docker"; then
            echo "User $NEW_USER already belongs to the 'docker' group."
        else
            echo "Adding user $NEW_USER to the 'docker' group."
            sudo usermod -aG docker "$NEW_USER"
        fi
        
        # Check if the user belongs to the 'sudo' group
        if groups "$NEW_USER" | grep -qw "sudo"; then
            echo "User $NEW_USER already belongs to the 'sudo' group."
        else
            echo "Adding user $NEW_USER to the 'sudo' group."
            sudo usermod -aG sudo "$NEW_USER"
        fi
    else
        echo "Creating a new non-root user: $NEW_USER"
        # --gecos "" skips the interactive prompts
        sudo adduser --gecos "" "$NEW_USER"
        echo "Adding $NEW_USER to the 'docker' and 'sudo' groups."
        sudo usermod -aG docker "$NEW_USER"
        sudo usermod -aG sudo "$NEW_USER"
    fi
}

# Function to check and optionally reinstall Docker
install_docker() {
    DOCKER_INSTALLED=$(command -v docker &>/dev/null && echo true || echo false)
    if [ "$DOCKER_INSTALLED" = true ]; then
        echo "Docker is already installed."
        if [ "$FORCE_REINSTALL" = true ]; then
            echo "Force reinstall is enabled. Reinstalling Docker..."
            uninstall_docker
            install_docker_logic
            enable_docker_service
            create_non_root_user
        fi
    else
        echo "Docker is not installed. Proceeding with installation..."
        install_docker_logic
        enable_docker_service
        create_non_root_user
    fi
}

# Function to install or reinstall the latest OpenShift CLI (oc)
install_openshift_cli() {
    if command -v oc &>/dev/null && [ "$FORCE_REINSTALL" = false ]; then
        echo "OpenShift CLI (oc) is already installed. Skipping installation."
    else
        echo "Installing or reinstalling OpenShift CLI (oc)..."
        HTTP_STATUS=$(curl -o /dev/null -s -w "%{http_code}\n" https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
        if [ "$HTTP_STATUS" -ne 200 ]; then
            echo "Failed to fetch the latest OpenShift CLI URL. HTTP status code: $HTTP_STATUS. Exiting."
            exit 1
        fi

        OC_PAGE_CONTENT=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
        OC_LATEST_URL=$(echo "$OC_PAGE_CONTENT" | grep -o 'href="openshift-client-linux-.*\.tar\.gz"' | head -n 1 | cut -d '"' -f 2)
        OC_DOWNLOAD_URL="https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$OC_LATEST_URL"

        if [ -z "$OC_LATEST_URL" ]; then
            echo "Failed to fetch the latest OpenShift CLI URL. Exiting."
            exit 1
        fi

        echo "Downloading from $OC_DOWNLOAD_URL"
        curl -LO "$OC_DOWNLOAD_URL"

        echo "Extracting OpenShift CLI..."
        tar -xzf $(basename "$OC_DOWNLOAD_URL")

        echo "Moving oc binary to /usr/local/bin..."
        sudo mv oc /usr/local/bin/
        sudo chmod +x /usr/local/bin/oc


        echo "Verifying oc installation..."
        oc version --client || { echo "Failed to verify OpenShift CLI installation."; exit 1; }

        rm -f $(basename "$OC_LATEST_URL") kubectl

        echo "Verifying oc installation..."
        oc version --client || { echo "Failed to verify OpenShift CLI installation."; exit 1; }
    fi
}

# Function to install or update Visual Studio Code
install_vscode() {
    echo "Checking for existing Visual Studio Code installation..."
    if command -v code >/dev/null 2>&1; then
        echo "Visual Studio Code is already installed. Checking for updates..."
        # Update package lists and upgrade VS Code
        sudo apt update
        sudo apt install --only-upgrade -y code
        echo "Visual Studio Code has been updated to the latest version."
    else
        echo "Visual Studio Code is not installed. Installing..."
        # Import the Microsoft GPG key
        wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
        sudo install -o root -g root -m 644 packages.microsoft.gpg /usr/share/keyrings/
        # Add the VS Code repository
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/vscode stable main" | sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null
        # Update package lists and install VS Code
        sudo apt update
        sudo apt install -y code
        # Clean up
        rm -f packages.microsoft.gpg
        echo "Visual Studio Code installation completed."
    fi
}

# Function to perform all tasks
perform_all_tasks() {
    update_system
    install_docker
    create_non_root_user
    install_openshift_cli
    install_vscode
}


if [ "$#" -eq 0 ]; then
    usage
fi

while [[ "$#" -gt 0 ]]; do
    case "$1" in
        --force) FORCE_REINSTALL=true; shift ;;
        --update) update_system; shift ;;
        --create-user)
            if [ -n "$2" ]; then
                create_non_root_user "$2"
                shift 2
            else
                echo "Error: --create-user requires a username."
                exit 1
            fi
            ;;
        --install)
                if [ -n "$2" ]; then
                    case "$2" in
                        docker) install_docker ;;
                        oc) install_openshift_cli ;;
                        vscode) install_vscode ;;
                        *) echo "Invalid option for --install. Use 'docker', 'oc', or 'vscode'." ;;
                    esac
                    shift 2
                else
                    echo "Error: --install requires an argument ('docker', 'oc', or 'vscode')."
                    exit 1
                fi
                ;;
        --all) perform_all_tasks; shift ;;
        --help) usage; shift ;;
        *) echo "Invalid option. Use --help to see the usage."; usage; exit 1 ;;
    esac
done

**ChatGPT**: **Setting non-root user password**  

To set a non-root user's password with this script, you can modify the `adduser` command by prompting for a password. For example, after adding the user, you can use:  
`sudo passwd "$NEW_USER"`  
This will prompt for a password for the user. Alternatively, you can modify the `adduser` command to include a `password` argument if you're automating the process further. The current script uses `--gecos ""` to skip prompts, but it still asks for a password interactively.

**Setting password non-interactively**  

If the goal is to set the user's password non-interactively, you can use the `chpasswd` command after creating the user. Normally, `adduser` prompts for a password, but if you'd rather skip that, you can include something like this in your script:  

```  
echo "$NEW_USER:password" | sudo chpasswd  
```  

In your script, this would go right after the `adduser` command to set the password without needing interaction. This ensures the password is set right when the user is created.

**ChatGPT**: In the current script, the non-root user is created with the `adduser` command using the `--gecos ""` option, which skips the interactive prompts for additional information but **still prompts for a password**. If you run the script as-is, when creating a new user the system will ask you to type in and confirm the password manually.

If you prefer to set the password non-interactively (for example, if you want the script to fully automate the process without waiting for user input), you can do so by using the `chpasswd` command immediately after creating the user. Here’s how you can modify the `create_non_root_user` function to set the password automatically:

1. **Define a Password (Not Recommended for Production):**  
   You can either hardcode a password (less secure) or prompt for one once (if you want to avoid interactive confirmation during user creation).

2. **Use `chpasswd` to Set the Password:**  
   After the user is created with `adduser`, pipe a string in the format `username:password` into `chpasswd`.

Below is an example of how you might modify the function:

```bash
create_non_root_user() {
    if [ -z "$1" ]; then
        read -r -p "Enter the username for the non-root user: " NEW_USER
    else
        NEW_USER="$1"
    fi

    if [ -z "$NEW_USER" ]; then
        echo "Error: Username cannot be empty. Please provide a valid username."
        exit 1
    fi

    if id "$NEW_USER" &>/dev/null; then
        echo "User $NEW_USER already exists."
        # Check if the user belongs to the 'docker' group
        if groups "$NEW_USER" | grep -qw "docker"; then
            echo "User $NEW_USER already belongs to the 'docker' group."
        else
            echo "Adding user $NEW_USER to the 'docker' group."
            sudo usermod -aG docker "$NEW_USER"
        fi
        # Check if the user belongs to the 'sudo' group
        if groups "$NEW_USER" | grep -qw "sudo"; then
            echo "User $NEW_USER already belongs to the 'sudo' group."
        else
            echo "Adding user $NEW_USER to the 'sudo' group."
            sudo usermod -aG sudo "$NEW_USER"
        fi
    else
        echo "Creating a new non-root user: $NEW_USER"
        # The adduser command will still prompt for a password unless you disable it.
        # To avoid interactive prompts entirely, you could pass --disabled-password, then set the password via chpasswd.
        sudo adduser --disabled-password --gecos "" "$NEW_USER"

        # Option 1: Hardcode a password (replace 'YourPasswordHere' with your desired password)
        # echo "$NEW_USER:YourPasswordHere" | sudo chpasswd

        # Option 2: Prompt once for the password (hidden input) and then set it
        read -rs -p "Enter password for $NEW_USER: " USER_PASS
        echo
        read -rs -p "Confirm password for $NEW_USER: " USER_PASS_CONFIRM
        echo
        if [ "$USER_PASS" != "$USER_PASS_CONFIRM" ]; then
            echo "Error: Passwords do not match."
            exit 1
        fi
        echo "$NEW_USER:$USER_PASS" | sudo chpasswd

        echo "Adding $NEW_USER to the 'docker' and 'sudo' groups."
        sudo usermod -aG docker "$NEW_USER"
        sudo usermod -aG sudo "$NEW_USER"
    fi
}
```

### Explanation

- **Using `--disabled-password`:**  
  The option `--disabled-password` prevents `adduser` from prompting for a password interactively. This way, you can set the password non-interactively using `chpasswd`.

- **Hardcoding vs. Prompting:**  
  - **Hardcoding (Option 1):** Directly setting the password with a known value (e.g., `YourPasswordHere`). This is simple but can be insecure if the script is shared.
  - **Prompting (Option 2):** Securely prompting the administrator running the script for a password and confirmation, then using that to set the password. This method avoids echoing the password to the terminal and ensures that the administrator confirms the value.

Choose the option that best suits your use case. For production environments, it's usually better to avoid hardcoding passwords and to use secure mechanisms for password management.

By making one of these modifications, you can control the password setup for the non-root user as part of your script.

**Swapan**: powershell script to drop/delete and recreate wsl distrubution

**ChatGPT**: **Creating PowerShell script**

The user's request is for a PowerShell script to drop, delete, and recreate a WSL distribution. To address this, we can use the `wsl --unregister <distro>` command to remove the distribution, followed by `wsl --import` to recreate it. Alternatively, `wsl --install` could be used to reinstall a distribution.

It's important to keep in mind the system's guidelines while providing a clear solution with these commands. I think this approach covers what needs to be done efficiently.

**Clarifying script functionality**

The request to "drop/delete and recreate a WSL distribution" is a bit ambiguous. "Drop" likely means to unregister, and "recreate" means reinstallation. For WSL, I typically use `wsl --unregister <distroName>` to remove a distribution, and `wsl --import` to import it from a tarball.

However, it depends on having the tarball, or using `wsl --install` if available. For a script, I could either download the tarball from an official source and import it or use `wsl --install` if on Windows 11. I’m still considering the exact workflow.

**Designing PowerShell script**

The user's request is to create a PowerShell script to drop/delete and recreate a WSL distribution. My approach is as follows:

- Accept parameters for distribution name, tarball URL (or local file), and install location.
- Unregister the existing distribution with `wsl --unregister <distroName>`.
- Download the tarball if it's not locally available.
- Create the install location folder.
- Import the distribution using `wsl --import`.

I’ll design a flexible script to handle Ubuntu distributions and customize for any rootfs tarball. I’ll ensure that WSL is installed and commands are in the PATH.

**ChatGPT**: Below is an example PowerShell script that un-registers (deletes) an existing WSL distribution and then re-creates it by downloading a root file system tarball and importing it. In this example, you supply the distribution name (for example, `"Ubuntu-20.04"`) and the URL for the tarball (for example, using Microsoft’s shortcut URL for Ubuntu 20.04). You can also specify an installation folder and the desired WSL version (defaulting to version 2).

Save the script (for example, as `Recreate-WSLDistro.ps1`), then run it from an elevated PowerShell session.

```powershell
<#
.SYNOPSIS
    Deletes (unregisters) an existing WSL distribution and recreates it by importing a downloaded tarball.

.DESCRIPTION
    This script will:
      1. Unregister any existing WSL distribution with the specified name.
      2. Create an installation folder (if it doesn’t exist).
      3. Download a tarball (rootfs) from a specified URL.
      4. Import the tarball as a new WSL distribution.
      5. Optionally, set the distribution’s WSL version.

.PARAMETER DistroName
    The name of the WSL distribution to recreate (e.g., "Ubuntu-20.04").

.PARAMETER TarballUrl
    The URL from which to download the Linux root filesystem tarball.
    For example, to download Ubuntu 20.04, you can use: "https://aka.ms/wsl-ubuntu-2004"

.PARAMETER InstallLocation
    The folder where the WSL distribution files will be stored.
    Default: "$env:LOCALAPPDATA\WSL\<DistroName>"

.PARAMETER WSLVersion
    The WSL version to set for the distribution (typically 1 or 2). Default is 2.

.EXAMPLE
    .\Recreate-WSLDistro.ps1 -DistroName "Ubuntu-20.04" `
       -TarballUrl "https://aka.ms/wsl-ubuntu-2004" `
       -InstallLocation "C:\WSL\Ubuntu-20.04" `
       -WSLVersion 2

    This command will unregister any existing "Ubuntu-20.04" distribution, download the tarball from the given URL,
    import it to "C:\WSL\Ubuntu-20.04", and set the WSL version to 2.
#>

[CmdletBinding()]
Param(
    [Parameter(Mandatory = $true)]
    [string]$DistroName,

    [Parameter(Mandatory = $true)]
    [string]$TarballUrl,

    [Parameter(Mandatory = $false)]
    [string]$InstallLocation = "$env:LOCALAPPDATA\WSL\$DistroName",

    [Parameter(Mandatory = $false)]
    [int]$WSLVersion = 2
)

# Check if wsl.exe is available.
function Check-WSLInstallation {
    if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
        Write-Error "WSL (wsl.exe) is not installed or not in PATH. Please install WSL and try again."
        exit 1
    }
}

Check-WSLInstallation

Write-Host "Attempting to unregister any existing WSL distribution named '$DistroName'..."
try {
    # This command removes the distribution if it exists.
    wsl --unregister $DistroName
    Write-Host "Successfully unregistered '$DistroName'."
}
catch {
    Write-Host "No existing distribution named '$DistroName' found or an error occurred while unregistering. Continuing..."
}

Write-Host "Creating installation directory at '$InstallLocation'..."
if (-not (Test-Path -Path $InstallLocation)) {
    New-Item -ItemType Directory -Path $InstallLocation | Out-Null
}

# Define a temporary path for the tarball download.
$tarballFile = Join-Path -Path $env:TEMP -ChildPath "$DistroName.tar.gz"

if (Test-Path $tarballFile) {
    Write-Host "Tarball file already exists at '$tarballFile'. Deleting it..."
    Remove-Item $tarballFile -Force
}

Write-Host "Downloading tarball from '$TarballUrl' to '$tarballFile'..."
try {
    Invoke-WebRequest -Uri $TarballUrl -OutFile $tarballFile
    Write-Host "Download completed."
}
catch {
    Write-Error "Failed to download tarball from '$TarballUrl'. Exiting."
    exit 1
}

Write-Host "Importing distribution '$DistroName' into WSL..."
try {
    # The --import command creates a new WSL distro using the downloaded tarball.
    wsl --import $DistroName $InstallLocation $tarballFile
    Write-Host "Import successful."
}
catch {
    Write-Error "Failed to import the distribution. Exiting."
    exit 1
}

Write-Host "Setting WSL version for '$DistroName' to $WSLVersion..."
try {
    wsl --set-version $DistroName $WSLVersion
    Write-Host "WSL version set successfully."
}
catch {
    Write-Warning "Failed to set WSL version. You may need to update your WSL installation."
}

Write-Host "WSL distribution '$DistroName' has been recreated successfully."
```

---

### How It Works

1. **Unregistering the Distribution:**  
   The script calls  
   ```powershell
   wsl --unregister $DistroName
   ```  
   to remove the distribution (if it exists).

2. **Creating the Install Directory:**  
   It ensures the directory where the new distribution files will reside exists (or creates it).

3. **Downloading the Tarball:**  
   It downloads the Linux root filesystem tarball from the URL you provide (using `Invoke-WebRequest`) into your TEMP folder.

4. **Importing the Distribution:**  
   The command  
   ```powershell
   wsl --import $DistroName $InstallLocation $tarballFile
   ```  
   imports the tarball as a new WSL distribution with the specified name.

5. **Setting the WSL Version:**  
   Finally, it sets the distribution to use the desired WSL version (e.g., 2) with  
   ```powershell
   wsl --set-version $DistroName $WSLVersion
   ```

---

### Running the Script

1. Open PowerShell as an administrator.
2. Navigate to the folder where you saved the script.
3. Run the script with the required parameters. For example:

   ```powershell
   .\Recreate-WSLDistro.ps1 -DistroName "Ubuntu-20.04" -TarballUrl "https://aka.ms/wsl-ubuntu-2004"
   ```

This script provides an automated way to drop (delete) and recreate a WSL distribution. Adjust parameters as needed for your environment or different distributions.

**Swapan**: get list of running distributions and prompt for the one to delete

**ChatGPT**: **Creating script for WSL distributions**

Okay, I need to create a PowerShell script that lists running distributions and prompts the user for the one to delete. For that, I'd use `wsl -l -v` to get the list — it shows the distributions and their running status.

After listing the distributions, the script will prompt the user to choose one to delete. Then, I guess we may ask if they'd want to recreate it afterward. This seems like a simple update to the first script!

**Filtering running distributions**

So, `wsl -l -v` lists all distributions and their states (e.g., Running, Stopped). The user specifically asked for "running distributions," meaning I need to focus on those where the state is "Running." I'll filter out the "Stopped" ones and ask the user to select which running distribution to delete. Since WSL doesn’t have a command like `-l --running`, parsing the "Running" status from the output of `wsl -l -v` in PowerShell seems like the right approach! Then, we can proceed with the deletion and recreation.

**Modifying script for deletion**

To modify the script, I'll follow these steps: 

1. Run `wsl -l -v` to get all distributions.
2. Filter out the ones with a "Running" state.
3. List those to the user and prompt for the distribution to delete.
4. After selection, confirm deletion with the user.
5. Unregister it with `wsl --unregister`.
6. Optionally, prompt for a tarball URL to recreate the distribution.

It's important that I check for cases where no running distributions exist and inform the user. This should create a flow for deletion, possibly adding the option for recreation if needed.

**Creating interactive script**

I can create a script to list running WSL distributions, letting the user choose one to delete. Here's how I can approach it:

1. Check for WSL availability.
2. Fetch the list using `wsl -l -v`, parse the output, and filter for "Running" distributions.
3. Show them to the user and prompt for a choice.
4. Confirm deletion and unregister the selected distribution.
5. Optionally, prompt to recreate the distribution by providing a tarball URL and installation location.

This script will handle errors such as no running distributions and bad user input, and allow for distribution recreation if desired.

**ChatGPT**: Below is an example PowerShell script that:

1. Retrieves the list of currently running WSL distributions by parsing the output of `wsl -l -v`.
2. Displays a numbered list of running distributions.
3. Prompts you to choose one (by number) to unregister (delete).
4. Confirms the deletion.
5. Optionally prompts you to recreate that distribution by asking for a tarball URL, an installation directory, and a WSL version before importing the tarball back as a new distribution.

> **Note:** Unregistering a distribution will remove all its data. Make sure you have backups or are certain you want to delete the chosen distribution.

Save the script (for example, as `Recreate-WSLDistroInteractive.ps1`), then run it from an elevated PowerShell session.

```powershell
<#
.SYNOPSIS
    Lists running WSL distributions, prompts the user to select one for deletion, and optionally recreates it.

.DESCRIPTION
    This script:
      1. Lists running WSL distributions by parsing the output of `wsl -l -v`.
      2. Prompts the user to select one distribution (by number) to unregister (delete).
      3. Unregisters the selected distribution.
      4. Optionally, if the user wishes to recreate the distribution, prompts for a tarball URL,
         installation directory, and WSL version, then downloads the tarball and imports it.

.NOTES
    Run PowerShell as an administrator. Unregistering a distribution deletes its data permanently.
#>

# Check if WSL is installed and available
function Check-WSLInstallation {
    if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
        Write-Error "WSL (wsl.exe) is not installed or not in PATH. Please install WSL and try again."
        exit 1
    }
}

Check-WSLInstallation

# Retrieve the list of distributions with their state and version.
# The command output looks like:
#
#   NAME                   STATE           VERSION
# * Ubuntu-20.04           Running         2
#   docker-desktop         Stopped         2
#   docker-desktop-data    Stopped         2
#
# We will skip the header and filter lines that indicate the distribution is Running.
Write-Host "Fetching list of running WSL distributions..."
$wslListRaw = wsl -l -v | Select-Object -Skip 1 | ForEach-Object { $_.Trim() }

if (-not $wslListRaw) {
    Write-Host "No WSL distributions found."
    exit
}

$runningDistros = @()
foreach ($line in $wslListRaw) {
    # Split the line into columns by whitespace.
    $columns = $line -split "\s+"
    # Remove a leading asterisk if present.
    if ($columns[0] -eq "*") {
        $columns = $columns[1..($columns.Count - 1)]
    }
    # Expecting at least 3 columns: Name, State, Version.
    if ($columns.Count -ge 3 -and $columns[1] -eq "Running") {
        $runningDistros += $columns[0]
    }
}

if ($runningDistros.Count -eq 0) {
    Write-Host "No running WSL distributions found."
    exit
}

# Display the list of running distributions.
Write-Host "Running WSL Distributions:"
for ($i = 0; $i -lt $runningDistros.Count; $i++) {
    Write-Host "$($i + 1). $($runningDistros[$i])"
}

# Prompt the user to select one distribution to delete.
$choice = Read-Host "Enter the number of the distribution you want to delete"
if (-not [int]::TryParse($choice, [ref]$null)) {
    Write-Error "Invalid input. Please enter a valid number."
    exit 1
}

$index = [int]$choice - 1
if ($index -lt 0 -or $index -ge $runningDistros.Count) {
    Write-Error "Selection out of range. Exiting."
    exit 1
}

$selectedDistro = $runningDistros[$index]
Write-Host "You selected: $selectedDistro"

# Confirm deletion
$confirm = Read-Host "Are you sure you want to unregister (delete) the distribution '$selectedDistro'? (Y/N)"
if ($confirm -notin @("Y", "y")) {
    Write-Host "Operation cancelled by user."
    exit
}

Write-Host "Unregistering distribution '$selectedDistro'..."
try {
    wsl --unregister $selectedDistro
    Write-Host "Distribution '$selectedDistro' has been unregistered successfully."
}
catch {
    Write-Error "Failed to unregister distribution '$selectedDistro'. Exiting."
    exit 1
}

# Ask if the user wants to recreate the distribution
$recreate = Read-Host "Do you want to recreate the distribution '$selectedDistro'? (Y/N)"
if ($recreate -notin @("Y", "y")) {
    Write-Host "Exiting without recreating the distribution."
    exit
}

# Prompt for the tarball URL for the new distribution.
$tarballUrl = Read-Host "Enter the tarball URL to import the distribution (e.g., https://aka.ms/wsl-ubuntu-2004)"
if ([string]::IsNullOrWhiteSpace($tarballUrl)) {
    Write-Error "Tarball URL cannot be empty. Exiting."
    exit 1
}

# Prompt for the installation directory.
$defaultInstallDir = "$env:LOCALAPPDATA\WSL\$selectedDistro"
$installLocation = Read-Host "Enter the installation directory for '$selectedDistro' (default: $defaultInstallDir)"
if ([string]::IsNullOrWhiteSpace($installLocation)) {
    $installLocation = $defaultInstallDir
}

# Prompt for the desired WSL version.
$wslVersionInput = Read-Host "Enter the WSL version to set for '$selectedDistro' (default is 2)"
if ([string]::IsNullOrWhiteSpace($wslVersionInput)) {
    $wslVersion = 2
} else {
    if (-not [int]::TryParse($wslVersionInput, [ref]$null)) {
        Write-Error "Invalid WSL version entered. Exiting."
        exit 1
    }
    $wslVersion = [int]$wslVersionInput
}

# Create the installation directory if it does not exist.
if (-not (Test-Path -Path $installLocation)) {
    Write-Host "Creating installation directory at '$installLocation'..."
    New-Item -ItemType Directory -Path $installLocation | Out-Null
}

# Define a temporary file path to download the tarball.
$tarballFile = Join-Path -Path $env:TEMP -ChildPath "$selectedDistro.tar.gz"
if (Test-Path $tarballFile) {
    Write-Host "Deleting existing tarball file at '$tarballFile'..."
    Remove-Item $tarballFile -Force
}

# Download the tarball.
Write-Host "Downloading tarball from '$tarballUrl' to '$tarballFile'..."
try {
    Invoke-WebRequest -Uri $tarballUrl -OutFile $tarballFile
    Write-Host "Tarball downloaded successfully."
}
catch {
    Write-Error "Failed to download tarball from '$tarballUrl'. Exiting."
    exit 1
}

# Import the distribution using the downloaded tarball.
Write-Host "Importing distribution '$selectedDistro' into WSL..."
try {
    wsl --import $selectedDistro $installLocation $tarballFile
    Write-Host "Import of '$selectedDistro' successful."
}
catch {
    Write-Error "Failed to import the distribution. Exiting."
    exit 1
}

# Set the desired WSL version.
Write-Host "Setting WSL version for '$selectedDistro' to $wslVersion..."
try {
    wsl --set-version $selectedDistro $wslVersion
    Write-Host "WSL version set successfully."
}
catch {
    Write-Warning "Failed to set WSL version for '$selectedDistro'. You may need to update your WSL installation."
}

Write-Host "Operation completed: Distribution '$selectedDistro' has been deleted and recreated successfully."
```

---

### How It Works

1. **Listing Running Distributions:**  
   The script runs `wsl -l -v` and skips the header line. It then splits each remaining line into columns and collects only those distributions whose state is `"Running"`.

2. **Prompting for Selection:**  
   It displays the running distributions with an index (number) and prompts you to enter the number corresponding to the distribution you want to delete.

3. **Unregistering the Distribution:**  
   After confirming, the script calls `wsl --unregister <DistributionName>` to delete the selected distribution.

4. **Optional Recreation:**  
   If you choose to recreate it, the script asks for:
   - The tarball URL (for example, a download link for a Linux root file system).
   - The installation directory (with a default location provided).
   - The desired WSL version (defaulting to 2).  
   It then downloads the tarball to a temporary location, imports it using `wsl --import`, and sets the WSL version.

5. **Execution:**  
   Run the script in an elevated PowerShell prompt. When prompted, follow the on-screen instructions.

This script provides an interactive way to select a running WSL distribution for deletion and optionally re-import it using a provided tarball.

**Swapan**: instead of just running get list both running and stopped distributions

**ChatGPT**: **Modifying script for user request**

The user wants the script to list all WSL distributions, whether they're running or stopped. I’ll need to adjust the code to remove the filter that only shows "Running" distributions and instead display all distributions, regardless of their state. Then, I’ll prompt the user to select a distribution to delete. Even if a distribution is running, it won't be a problem—I'll just ask for the user’s selection and proceed with the deletion.

**Revised script for user request**

The user wants the script to list both running and stopped distributions. I’ll adjust the script by removing the filter that only collects running distributions. To do this, I’ll use the `wsl -l -v` command, which lists all distributions along with their state and version. I’ll clean the output—removing any header lines and asterisks from the default distribution. Then, I’ll present the list with indices and prompt the user to select one for deletion, including a confirmation step. I'll modify the script accordingly.

**ChatGPT**: Below is an updated PowerShell script that lists **all** WSL distributions (both running and stopped), displays them in a numbered list (including their state and version), and then prompts you to select one to unregister (delete). After confirmation, it unregisters the selected distribution. You can also choose to recreate it by providing a tarball URL, an installation directory, and a desired WSL version.

> **Warning:** Unregistering a WSL distribution will permanently remove all its data. Use this script with caution and ensure you have backups if needed.

Save the script (for example, as `Manage-WSLDistro.ps1`) and run it in an elevated PowerShell session.

```powershell
<#
.SYNOPSIS
    Lists all WSL distributions (running and stopped), prompts the user to select one for deletion,
    and optionally recreates it.

.DESCRIPTION
    This script will:
      1. Retrieve all WSL distributions by parsing the output of `wsl -l -v`.
      2. Display a numbered list showing the distribution name, state, and version.
      3. Prompt the user to select one distribution (by number) to unregister (delete).
      4. Confirm the deletion.
      5. Optionally prompt to recreate the distribution by asking for:
         - Tarball URL (to download a Linux rootfs)
         - Installation directory
         - Desired WSL version
         The script then downloads the tarball, imports it, and sets the WSL version.

.NOTES
    Run PowerShell as an administrator.
    Unregistering a distribution deletes its data permanently.
#>

# Function: Check if WSL is available
function Check-WSLInstallation {
    if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
        Write-Error "WSL (wsl.exe) is not installed or not in PATH. Please install WSL and try again."
        exit 1
    }
}

Check-WSLInstallation

# Retrieve the list of all distributions (running and stopped)
Write-Host "Fetching list of all WSL distributions..."
$wslListRaw = wsl -l -v | Select-Object -Skip 1 | ForEach-Object { $_.Trim() } 

if (-not $wslListRaw) {
    Write-Host "No WSL distributions found."
    exit
}

# Parse the output and build an array of objects
$distros = @()
foreach ($line in $wslListRaw) {
    if ([string]::IsNullOrWhiteSpace($line)) { continue }
    
    # Split the line by whitespace
    $columns = $line -split "\s+"
    
    # Remove the leading asterisk (*) if present (indicates default distro)
    if ($columns[0] -eq "*") {
        $columns = $columns[1..($columns.Count - 1)]
    }
    
    # Expecting at least three columns: Name, State, Version.
    if ($columns.Count -ge 3) {
        $distros += [PSCustomObject]@{
            Name    = $columns[0]
            State   = $columns[1]
            Version = $columns[2]
        }
    }
}

if ($distros.Count -eq 0) {
    Write-Host "No WSL distributions found."
    exit
}

# Display the list of distributions with index numbers
Write-Host "`nWSL Distributions:"
for ($i = 0; $i -lt $distros.Count; $i++) {
    Write-Host ("{0}. {1} - State: {2}, Version: {3}" -f ($i + 1), $distros[$i].Name, $distros[$i].State, $distros[$i].Version)
}

# Prompt the user to select one distribution to delete
$choice = Read-Host "`nEnter the number of the distribution you want to delete"
if (-not [int]::TryParse($choice, [ref]$null)) {
    Write-Error "Invalid input. Please enter a valid number."
    exit 1
}

$index = [int]$choice - 1
if ($index -lt 0 -or $index -ge $distros.Count) {
    Write-Error "Selection out of range. Exiting."
    exit 1
}

$selectedDistro = $distros[$index]
Write-Host "`nYou selected: $($selectedDistro.Name) - State: $($selectedDistro.State), Version: $($selectedDistro.Version)"

# Confirm deletion
$confirm = Read-Host "Are you sure you want to unregister (delete) the distribution '$($selectedDistro.Name)'? (Y/N)"
if ($confirm -notin @("Y", "y")) {
    Write-Host "Operation cancelled by user."
    exit
}

# Unregister the selected distribution
Write-Host "Unregistering distribution '$($selectedDistro.Name)'..."
try {
    wsl --unregister $selectedDistro.Name
    Write-Host "Distribution '$($selectedDistro.Name)' has been unregistered successfully."
}
catch {
    Write-Error "Failed to unregister distribution '$($selectedDistro.Name)'. Exiting."
    exit 1
}

# Ask if the user wants to recreate the distribution
$recreate = Read-Host "Do you want to recreate the distribution '$($selectedDistro.Name)'? (Y/N)"
if ($recreate -notin @("Y", "y")) {
    Write-Host "Exiting without recreating the distribution."
    exit
}

# Prompt for the tarball URL for the new distribution
$tarballUrl = Read-Host "Enter the tarball URL to import the distribution (e.g., https://aka.ms/wsl-ubuntu-2004)"
if ([string]::IsNullOrWhiteSpace($tarballUrl)) {
    Write-Error "Tarball URL cannot be empty. Exiting."
    exit 1
}

# Prompt for the installation directory (with a default value)
$defaultInstallDir = "$env:LOCALAPPDATA\WSL\$($selectedDistro.Name)"
$installLocation = Read-Host "Enter the installation directory for '$($selectedDistro.Name)' (default: $defaultInstallDir)"
if ([string]::IsNullOrWhiteSpace($installLocation)) {
    $installLocation = $defaultInstallDir
}

# Prompt for the desired WSL version
$wslVersionInput = Read-Host "Enter the WSL version to set for '$($selectedDistro.Name)' (default is 2)"
if ([string]::IsNullOrWhiteSpace($wslVersionInput)) {
    $wslVersion = 2
} else {
    if (-not [int]::TryParse($wslVersionInput, [ref]$null)) {
        Write-Error "Invalid WSL version entered. Exiting."
        exit 1
    }
    $wslVersion = [int]$wslVersionInput
}

# Create the installation directory if it does not exist
if (-not (Test-Path -Path $installLocation)) {
    Write-Host "Creating installation directory at '$installLocation'..."
    New-Item -ItemType Directory -Path $installLocation | Out-Null
}

# Define a temporary file path to download the tarball
$tarballFile = Join-Path -Path $env:TEMP -ChildPath "$($selectedDistro.Name).tar.gz"
if (Test-Path $tarballFile) {
    Write-Host "Deleting existing tarball file at '$tarballFile'..."
    Remove-Item $tarballFile -Force
}

# Download the tarball
Write-Host "Downloading tarball from '$tarballUrl' to '$tarballFile'..."
try {
    Invoke-WebRequest -Uri $tarballUrl -OutFile $tarballFile
    Write-Host "Tarball downloaded successfully."
}
catch {
    Write-Error "Failed to download tarball from '$tarballUrl'. Exiting."
    exit 1
}

# Import the distribution using the downloaded tarball
Write-Host "Importing distribution '$($selectedDistro.Name)' into WSL..."
try {
    wsl --import $selectedDistro.Name $installLocation $tarballFile
    Write-Host "Import of '$($selectedDistro.Name)' successful."
}
catch {
    Write-Error "Failed to import the distribution. Exiting."
    exit 1
}

# Set the desired WSL version for the imported distribution
Write-Host "Setting WSL version for '$($selectedDistro.Name)' to $wslVersion..."
try {
    wsl --set-version $selectedDistro.Name $wslVersion
    Write-Host "WSL version set successfully."
}
catch {
    Write-Warning "Failed to set WSL version for '$($selectedDistro.Name)'. You may need to update your WSL installation."
}

Write-Host "`nOperation completed: Distribution '$($selectedDistro.Name)' has been deleted and recreated successfully."
```

---

### How It Works

1. **List All Distributions:**  
   The script calls `wsl -l -v`, skips the header, and then parses each line. It removes any leading asterisk (used to denote the default distribution) and captures the distribution's **Name**, **State**, and **Version** in an array of objects.

2. **Display the List:**  
   Each distribution is displayed in a numbered list so you can see both running and stopped distributions along with their details.

3. **User Selection:**  
   You are prompted to enter the number corresponding to the distribution you wish to delete. The script validates the input to ensure it is within range.

4. **Confirmation and Unregister:**  
   Once you confirm, the script calls `wsl --unregister <DistroName>` to delete the selected distribution.

5. **Optional Recreation:**  
   If you choose to recreate the distribution, the script will prompt for:
   - A tarball URL to download the Linux root filesystem.
   - An installation directory (with a default suggestion).
   - The WSL version to set (defaulting to version 2).  
   It then downloads the tarball, imports the distribution with `wsl --import`, and sets the specified WSL version.

Run the script in an elevated PowerShell window, follow the prompts, and manage your WSL distributions interactively.

**Swapan**: seperate everythign into functions

**ChatGPT**: **Separating script into functions**

Okay, so we're working on breaking the script into individual functions instead of keeping it all integrated in one piece. The functions would include: 

- Check-WSLInstallation (checks if WSL is installed)
- Get-WSLDistributions (returns distribution list)
- Display-WSLDistributions (shows distributions in numbered list)
- Prompt-UserSelection (prompts for user selection of distribution)
- Unregister-WSLDistro (unregisters a distribution)
- Prompt-RecreateOption (asks if user wants to recreate)
  
I’ll organize the script by putting each of these into separate functions, then have the main code call them as needed.

**Breaking down the script functions**

Let’s break this down: the script consists of a **main function** that calls all these smaller functions. I'll produce a final version where everything is separated into clearly named functions with comments.

For example, **Get-WSLDistributions** will run the command `wsl -l -v` to list active distributions, then parse the output into an array of objects representing each distribution.

Next, the **Display-WSLDistributions** function will output the list of distributions with an index.

Then, **Prompt-UserSelection** will ask the user to choose a distribution by number.

I’ll continue from here and finalize this code.




**ChatGPT**: Below is a complete PowerShell script that separates each task into its own function. The script will:

1. **Check for WSL installation.**  
2. **Retrieve and display all WSL distributions** (both running and stopped).  
3. **Prompt the user to select a distribution** (by number) to unregister (delete).  
4. **Confirm and unregister the selected distribution.**  
5. **Optionally prompt to recreate the distribution** by asking for a tarball URL, installation directory, and WSL version, then download, import, and configure it.

> **Warning:** Unregistering a WSL distribution permanently removes its data. Use this script with caution.

Save the script (e.g., as `Manage-WSLDistro_Functions.ps1`), then run it from an elevated PowerShell session.

```powershell
<#
.SYNOPSIS
    Manage WSL distributions: list all distributions, delete one, and optionally recreate it.

.DESCRIPTION
    This script:
      - Checks if WSL is installed.
      - Retrieves all WSL distributions (running and stopped).
      - Displays a numbered list of distributions with their state and version.
      - Prompts the user to select a distribution to delete.
      - Unregisters (deletes) the selected distribution.
      - Optionally, if the user chooses, prompts for details to recreate the distribution:
          • Tarball URL (to download a Linux root filesystem)
          • Installation directory
          • Desired WSL version
        It then downloads the tarball, imports the distribution, and sets the specified WSL version.

.NOTES
    Run this script as an administrator.
    Unregistering a distribution deletes its data permanently.
#>

#region Functions

function Check-WSLInstallation {
    <#
    .SYNOPSIS
        Checks if WSL (wsl.exe) is available in PATH.
    #>
    if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
        Write-Error "WSL (wsl.exe) is not installed or not in PATH. Please install WSL and try again."
        exit 1
    }
}

function Get-WSLDistributions {
    <#
    .SYNOPSIS
        Retrieves all WSL distributions (running and stopped) and parses them into objects.
    .OUTPUTS
        Array of PSCustomObjects with properties: Name, State, and Version.
    #>
    $wslListRaw = wsl -l -v | Select-Object -Skip 1 | ForEach-Object { $_.Trim() }
    $distros = @()
    foreach ($line in $wslListRaw) {
        if ([string]::IsNullOrWhiteSpace($line)) { continue }
        $columns = $line -split "\s+"
        # Remove the leading asterisk if present (indicates the default distro)
        if ($columns[0] -eq "*") {
            $columns = $columns[1..($columns.Count - 1)]
        }
        if ($columns.Count -ge 3) {
            $distros += [PSCustomObject]@{
                Name    = $columns[0]
                State   = $columns[1]
                Version = $columns[2]
            }
        }
    }
    return $distros
}

function Display-WSLDistributions {
    <#
    .SYNOPSIS
        Displays a numbered list of WSL distributions.
    .PARAMETER Distros
        Array of distribution objects.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [array]$Distros
    )
    Write-Host "`nWSL Distributions:"
    for ($i = 0; $i -lt $Distros.Count; $i++) {
        Write-Host ("{0}. {1} - State: {2}, Version: {3}" -f ($i + 1), $Distros[$i].Name, $Distros[$i].State, $Distros[$i].Version)
    }
}

function Prompt-UserSelection {
    <#
    .SYNOPSIS
        Prompts the user to select a distribution by number.
    .PARAMETER Distros
        Array of distribution objects.
    .OUTPUTS
        The selected distribution object.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [array]$Distros
    )
    $choice = Read-Host "`nEnter the number of the distribution you want to delete"
    if (-not [int]::TryParse($choice, [ref]$null)) {
        Write-Error "Invalid input. Please enter a valid number."
        return $null
    }
    $index = [int]$choice - 1
    if ($index -lt 0 -or $index -ge $Distros.Count) {
        Write-Error "Selection out of range."
        return $null
    }
    return $Distros[$index]
}

function Unregister-WSLDistro {
    <#
    .SYNOPSIS
        Unregisters (deletes) the specified WSL distribution.
    .PARAMETER DistroName
        The name of the distribution to unregister.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$DistroName
    )
    try {
        wsl --unregister $DistroName
        Write-Host "Distribution '$DistroName' has been unregistered successfully."
    }
    catch {
        Write-Error "Failed to unregister distribution '$DistroName'."
        exit 1
    }
}

function Prompt-RecreateOption {
    <#
    .SYNOPSIS
        Prompts the user if they want to recreate the deleted distribution.
    .PARAMETER DistroName
        The name of the distribution.
    .OUTPUTS
        Returns $true if the user chooses to recreate, otherwise $false.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$DistroName
    )
    $recreate = Read-Host "Do you want to recreate the distribution '$DistroName'? (Y/N)"
    return $recreate -in @("Y", "y")
}

function Prompt-ImportDetails {
    <#
    .SYNOPSIS
        Prompts for details required to import (recreate) a distribution.
    .PARAMETER DistroName
        The name of the distribution.
    .OUTPUTS
        A PSCustomObject containing TarballUrl, InstallLocation, and WSLVersion.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$DistroName
    )
    $tarballUrl = Read-Host "Enter the tarball URL to import the distribution (e.g., https://aka.ms/wsl-ubuntu-2004)"
    if ([string]::IsNullOrWhiteSpace($tarballUrl)) {
        Write-Error "Tarball URL cannot be empty. Exiting."
        exit 1
    }
    $defaultInstallDir = "$env:LOCALAPPDATA\WSL\$DistroName"
    $installLocation = Read-Host "Enter the installation directory for '$DistroName' (default: $defaultInstallDir)"
    if ([string]::IsNullOrWhiteSpace($installLocation)) {
        $installLocation = $defaultInstallDir
    }
    $wslVersionInput = Read-Host "Enter the WSL version to set for '$DistroName' (default is 2)"
    if ([string]::IsNullOrWhiteSpace($wslVersionInput)) {
        $wslVersion = 2
    } else {
        if (-not [int]::TryParse($wslVersionInput, [ref]$null)) {
            Write-Error "Invalid WSL version entered. Exiting."
            exit 1
        }
        $wslVersion = [int]$wslVersionInput
    }
    return [PSCustomObject]@{
        TarballUrl     = $tarballUrl
        InstallLocation = $installLocation
        WSLVersion     = $wslVersion
    }
}

function Download-Tarball {
    <#
    .SYNOPSIS
        Downloads the tarball from the specified URL.
    .PARAMETER TarballUrl
        The URL from which to download the tarball.
    .PARAMETER DistroName
        The name of the distribution (used to name the tarball file).
    .OUTPUTS
        The path to the downloaded tarball file.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$TarballUrl,
        [Parameter(Mandatory = $true)]
        [string]$DistroName
    )
    $tarballFile = Join-Path -Path $env:TEMP -ChildPath "$DistroName.tar.gz"
    if (Test-Path $tarballFile) {
        Write-Host "Deleting existing tarball file at '$tarballFile'..."
        Remove-Item $tarballFile -Force
    }
    Write-Host "Downloading tarball from '$TarballUrl' to '$tarballFile'..."
    try {
        Invoke-WebRequest -Uri $TarballUrl -OutFile $tarballFile
        Write-Host "Tarball downloaded successfully."
    }
    catch {
        Write-Error "Failed to download tarball from '$TarballUrl'. Exiting."
        exit 1
    }
    return $tarballFile
}

function Import-Distribution {
    <#
    .SYNOPSIS
        Imports the distribution using the downloaded tarball.
    .PARAMETER DistroName
        The name of the distribution.
    .PARAMETER InstallLocation
        The directory where the distribution will be installed.
    .PARAMETER TarballFile
        The path to the downloaded tarball.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$DistroName,
        [Parameter(Mandatory = $true)]
        [string]$InstallLocation,
        [Parameter(Mandatory = $true)]
        [string]$TarballFile
    )
    # Create the installation directory if it does not exist.
    if (-not (Test-Path -Path $InstallLocation)) {
        Write-Host "Creating installation directory at '$InstallLocation'..."
        New-Item -ItemType Directory -Path $InstallLocation | Out-Null
    }
    Write-Host "Importing distribution '$DistroName' into WSL..."
    try {
        wsl --import $DistroName $InstallLocation $TarballFile
        Write-Host "Import of '$DistroName' successful."
    }
    catch {
        Write-Error "Failed to import the distribution. Exiting."
        exit 1
    }
}

function Set-WSLVersion {
    <#
    .SYNOPSIS
        Sets the WSL version for the specified distribution.
    .PARAMETER DistroName
        The name of the distribution.
    .PARAMETER WSLVersion
        The desired WSL version.
    #>
    param(
        [Parameter(Mandatory = $true)]
        [string]$DistroName,
        [Parameter(Mandatory = $true)]
        [int]$WSLVersion
    )
    Write-Host "Setting WSL version for '$DistroName' to $WSLVersion..."
    try {
        wsl --set-version $DistroName $WSLVersion
        Write-Host "WSL version set successfully."
    }
    catch {
        Write-Warning "Failed to set WSL version for '$DistroName'. You may need to update your WSL installation."
    }
}

#endregion

#region Main Execution

# 1. Ensure WSL is installed.
Check-WSLInstallation

# 2. Retrieve all WSL distributions.
$distros = Get-WSLDistributions
if ($distros.Count -eq 0) {
    Write-Host "No WSL distributions found."
    exit
}

# 3. Display the distributions.
Display-WSLDistributions -Distros $distros

# 4. Prompt the user for selection.
$selectedDistro = Prompt-UserSelection -Distros $distros
if ($null -eq $selectedDistro) { exit 1 }

Write-Host "`nYou selected: $($selectedDistro.Name) - State: $($selectedDistro.State), Version: $($selectedDistro.Version)"

# 5. Confirm deletion.
$confirm = Read-Host "Are you sure you want to unregister (delete) the distribution '$($selectedDistro.Name)'? (Y/N)"
if ($confirm -notin @("Y", "y")) {
    Write-Host "Operation cancelled by user."
    exit
}

# 6. Unregister (delete) the selected distribution.
Unregister-WSLDistro -DistroName $selectedDistro.Name

# 7. Ask if the user wants to recreate the distribution.
if (Prompt-RecreateOption -DistroName $selectedDistro.Name) {
    # 8. Prompt for import details.
    $details = Prompt-ImportDetails -DistroName $selectedDistro.Name
    # 9. Download the tarball.
    $tarballFile = Download-Tarball -TarballUrl $details.TarballUrl -DistroName $selectedDistro.Name
    # 10. Import the distribution.
    Import-Distribution -DistroName $selectedDistro.Name -InstallLocation $details.InstallLocation -TarballFile $tarballFile
    # 11. Set the WSL version.
    Set-WSLVersion -DistroName $selectedDistro.Name -WSLVersion $details.WSLVersion
    Write-Host "`nOperation completed: Distribution '$($selectedDistro.Name)' has been deleted and recreated successfully."
}
else {
    Write-Host "Exiting without recreating the distribution."
}

#endregion
```

---

### How It Works

- **Functions Section:**  
  Each function encapsulates a specific task (e.g., checking for WSL, listing distributions, prompting the user, unregistering, downloading a tarball, importing a distribution, and setting the WSL version).

- **Main Execution:**  
  The main block calls these functions in sequence:
  1. **Check WSL Installation**
  2. **Retrieve and Display Distributions**
  3. **Prompt for Selection and Confirm Deletion**
  4. **Unregister the Selected Distribution**
  5. **Prompt to Recreate and If Chosen, Gather Details, Download, Import, and Set WSL Version**

Run the script in an elevated PowerShell session and follow the on-screen prompts to manage your WSL distributions.

**Swapan**: wsl ubuntu cloud-init docker install with non root user

**ChatGPT**: **Considering WSL Ubuntu setup**

User is asking for instructions or a script to set up WSL Ubuntu with Docker and a non-root user using cloud-init. The challenge here is that WSL doesn't natively support cloud-init, as it's typically not used in WSL environments. However, I might suggest alternatives to set up Docker and create a non-root user. One possible solution is providing a cloud-init configuration file to install Docker, create a non-root user, and potentially customize the environment further.

**Exploring cloud-init in WSL**

I need to address the challenge that WSL doesn't automatically support cloud-init. Well, I can work around this by using a custom script that mimics cloud-init functionality. The user seems to want a script or configuration file to install Docker and create a non-root user in WSL Ubuntu. I'll create a cloud-init YAML file for this, including commands to update apt, install Docker, and set up a non-root user with sudo privileges and Docker group membership. Some recent WSL versions support cloud-init, but it's often disabled.

**Creating cloud-init config for WSL**

The user likely wants a cloud-init configuration for WSL Ubuntu to install Docker and create a non-root user. I can create something like the YAML file that updates and installs Docker, creates the user, and adds them to necessary groups. However, I need to remember that WSL doesn’t support systemd by default, requiring a workaround for Docker when not using systemd. I’ll also note that cloud-init is experimental in WSL, so I need to clarify using a manual script for setup if necessary.

**ChatGPT**: Below is an example cloud‑init configuration file that you can use with a custom WSL‑Ubuntu image (or during first‑boot provisioning) to do the following:

1. **Update and upgrade packages.**  
2. **Install Docker** (using the Ubuntu “docker.io” package plus its dependencies).  
3. **Create a non‑root user** (named “nonroot” in this example) with sudo privileges and membership in the docker group.  
4. **Set the user’s password.**  
5. **Attempt to start the Docker service.**

> **Important Notes:**  
> - **WSL & Systemd:** By default, WSL does not run systemd. Newer versions of WSL offer experimental systemd support when configured (for example, by adding a `[boot] systemd=true` section in `/etc/wsl.conf`). Without systemd, service management (like automatically starting Docker) must be handled manually or by alternative means.  
> - **Password Security:** In this example, the password is set in plaintext via the `chpasswd` module. In a production scenario you may wish to provide a pre‑hashed password (using the `passwd:` key for the user) or use more secure methods to supply credentials.  
> - **Cloud‑init on WSL:** Cloud‑init is not enabled by default on standard WSL‑Ubuntu installations. To use cloud‑init you’ll need to build or customize your image accordingly (or run the cloud‑init stages manually).  

---

### Example: cloud‑init Configuration (YAML)

Save the following content (for example, as `cloud-init.yaml`):

```yaml
#cloud-config
# This configuration performs the following:
#   1. Updates and upgrades packages.
#   2. Installs Docker (using the docker.io package).
#   3. Creates a non-root user "nonroot" with sudo and docker privileges.
#   4. Sets a password for "nonroot" (plaintext in this example).
#   5. Attempts to start the Docker service.
apt_update: true
apt_upgrade: true

packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg-agent
  - software-properties-common
  - docker.io

users:
  - name: nonroot
    gecos: "Non-root User"
    groups: [sudo, docker]
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    lock_passwd: false
    # The following is a placeholder for a pre-hashed password.
    # (If you want to use a pre-hashed password, replace the value below with your hashed string.)
    passwd: "$6$rounds=4096$EXAMPLEsALT$EXAMPLEhashedpasswordvalue"

chpasswd:
  list: |
    nonroot:mysecurepassword
  expire: False

runcmd:
  # Ensure the non-root user is in the docker group.
  - [ usermod, -aG, docker, nonroot ]
  # Attempt to start the Docker service.
  # (This will only work if systemd is available and running in your WSL instance.)
  - [ service, docker, start ]
```

---

### How to Use This Configuration

1. **Customizing the File:**  
   - Change the username (here `"nonroot"`) if desired.  
   - Adjust the password in the `chpasswd` section (or provide a pre‑hashed password in the `passwd:` key).  
   - If you prefer to install the latest Docker CE rather than Ubuntu’s docker.io package, you can modify the `packages:` section and add the appropriate repository configuration commands in an additional `runcmd:` block.

2. **Deploying in WSL‑Ubuntu:**  
   - If you are building a custom WSL‑Ubuntu image or otherwise enabling cloud‑init on your WSL instance, include this file in your image (for example, by placing it in `/etc/cloud/cloud.cfg.d/50_custom.cfg`).  
   - Alternatively, you can run cloud‑init manually (if it’s installed) to test the configuration:
     ```bash
     sudo cloud-init clean
     sudo cloud-init init
     sudo cloud-init modules --mode=config
     sudo cloud-init modules --mode=final
     ```
   - Remember that if you want Docker to start automatically under WSL, you may need to enable systemd by creating or editing `/etc/wsl.conf` as follows:
     ```ini
     [boot]
     systemd=true
     ```

3. **Re‑Start WSL:**  
   After making these changes, restart your WSL instance:
   ```powershell
   wsl --shutdown
   ```
   Then re‑launch your Ubuntu distribution.

---

This cloud‑init configuration provides an automated way to provision a WSL‑Ubuntu environment with Docker installed and a non‑root user set up. Adjust and extend the configuration to suit your environment and security requirements.

**Swapan**: what does the following do?

chpasswd:
  list: |
    nonroot:mysecurepassword
  expire: False

**ChatGPT**: **Explaining chpasswd configuration**

This cloud-init snippet is using the "chpasswd" module to set a password. The `list: |` part signifies a block of text, and the snippet specifies that the “nonroot” user will get the password "mysecurepassword." The `expire: False` tells cloud-init not to force the password to expire after the first login. Essentially, when this configuration is processed, it updates the password for "nonroot" without requiring a password change at login.

**ChatGPT**: This snippet is part of a cloud‑init configuration and uses the `chpasswd` module to manage user passwords. Here's what each part does:

- **`list: |`**  
  The pipe (`|`) indicates that a multi‑line string is being provided. In this case, the string contains a line with a username and password in the format `username:password`.

- **`nonroot:mysecurepassword`**  
  This line tells cloud‑init to set the password for the user `nonroot` to `mysecurepassword`.

- **`expire: False`**  
  This setting ensures that the password is not marked as expired. As a result, when the user logs in for the first time, they will not be forced to change the password immediately.

---

### Summary

When cloud‑init processes this configuration, it will:

1. Set the password for the user `nonroot` to `mysecurepassword`.
2. Ensure that the password does not expire immediately, so the user is not prompted to change it on first login.

**Swapan**: what is the purpose of having both 
passwd: "$6$rounds=4096$EXAMPLEsALT$EXAMPLEhashedpasswordvalue"

and 
chpasswd:
  list: |
    nonroot:mysecurepassword
  expire: False

**ChatGPT**: **Explaining password configuration**

The user is asking about the combination of `passwd` and `chpasswd` in cloud-init config files. 

The `passwd:` field is for pre-hashed passwords, like `"$6$rounds=4096$...hashedpasswordvalue"`. It’s usually a SHA-512 hash. Meanwhile, `chpasswd:` is used for plain text passwords via `username:password` format, which the system processes with a command like "chpasswd." 

Having both options provides redundancy or for compatibility with different usage scenarios, ensuring both hashed and plain text passwords can be handled appropriately.

**Clarifying password configuration order**

The user may be wondering why both a hashed password in the user definition (`passwd:`) and a plain-text password in `chpasswd:` exist together. It seems like there might be some redundancy or mismatch in processing order — my guess is the `passwd:` setting is processed first, then `chpasswd:` processes later. 

Actually, if both are set, `chpasswd:` could overwrite the earlier password. Having both could allow for pre-setting a hashed value that gets overwritten by a plain text password.

**ChatGPT**: In a cloud‑init configuration you can set a user’s password in more than one way. Here’s what each method does and why you might see both:

1. **Using `passwd:` in the user definition**  
   ```yaml
   users:
     - name: nonroot
       # ... other user settings ...
       passwd: "$6$rounds=4096$EXAMPLEsALT$EXAMPLEhashedpasswordvalue"
   ```  
   This line supplies a **pre‑hashed password** for the user. When cloud‑init creates the user, it writes this hash directly into the system’s shadow file. This method is secure because it does not expose the plain‑text password in the configuration file (aside from when you generate the hash), and it ensures that the user is created with that password.

2. **Using the `chpasswd:` module**  
   ```yaml
   chpasswd:
     list: |
       nonroot:mysecurepassword
     expire: False
   ```  
   The `chpasswd` module takes a plain‑text password (here, `mysecurepassword`) and sets it for the user. The `expire: False` setting means the password will not be forced to change at first login. This module runs after the initial user creation and can override the password set in the `passwd:` field.

---

### Why Have Both?

Including **both** entries can be done for several reasons:

- **Redundancy or Fallback:**  
  The hashed password in the `passwd:` field ensures that the user is created with a known password even if later modules are skipped or fail. Then, the `chpasswd` module can update the password to a plain‑text value you specify (which might be easier to change or configure in some environments).

- **Different Stages of Provisioning:**  
  Cloud‑init processes different modules at different stages. The `passwd:` field is part of the initial user creation, while the `chpasswd:` module runs later. In some setups, you might want to create the user with a safe default hash and then explicitly set or update the password afterward using `chpasswd`.

- **Compatibility:**  
  Some deployments or images may require one method or the other. Including both can help ensure that, regardless of the order in which cloud‑init modules run or which one is supported in a given environment, the user’s password ends up as desired.

---

### In Summary

- **`passwd:`** provides a secure, pre‑hashed password at the time the user is created.
- **`chpasswd:`** then (usually later) sets or updates the password using a plain‑text value and can also control whether the password is forced to expire.

Having both entries ensures that the user is created with a password and then explicitly updated to the intended value (and configuration such as non‑expiration), covering different provisioning stages and compatibility scenarios.

**Swapan**: how to generate the hashed password?

**ChatGPT**: **Generating hashed password**

To generate a hashed password for cloud-init, you can use either `openssl passwd -6` or `mkpasswd` commands. In Ubuntu, the `mkpasswd` command comes from the "whois" package, or you can use `openssl passwd -6`. For example, to use `openssl`, just type `openssl passwd -6 "mysecurepassword"`, and it will return a SHA-512 hashed password string. Python is another option, but `openssl` is probably the easiest. This hashed password can then be used in your cloud-init configuration.

**ChatGPT**: There are a couple of common ways to generate a SHA‑512 hashed password (in the format expected by cloud‑init) on a Linux system:

---

### Using OpenSSL

You can use the `openssl` command with the `-6` option (which specifies the SHA‑512 algorithm):

```bash
openssl passwd -6 "mysecurepassword"
```

This command will output a hashed password string that looks similar to:

```
$6$randomsalt$hashedvalue...
```

You can then copy this output and use it in your cloud‑init configuration under the `passwd:` key.

*Optional:* To specify a particular number of rounds (for example, 4096 rounds), you can use the `-rounds` option (note that not all OpenSSL versions may support this):

```bash
openssl passwd -6 -rounds 4096 "mysecurepassword"
```

---

### Using Python

If you have Python installed, you can generate the hash with a one‑liner using Python’s built‑in `crypt` module:

```bash
python3 -c "import crypt; print(crypt.crypt('mysecurepassword', crypt.mksalt(crypt.METHOD_SHA512)))"
```

This command will produce output in the same format as the OpenSSL method.

---

### When to Use These Hashed Passwords

- **In cloud‑init:**  
  Place the generated hash in your cloud‑init configuration under the user definition’s `passwd:` field. For example:

  ```yaml
  users:
    - name: nonroot
      shell: /bin/bash
      groups: [sudo, docker]
      passwd: "$6$randomsalt$hashedvalue..."
  ```

- **Security Note:**  
  Generating and using pre‑hashed passwords avoids storing plain‑text passwords in your configuration. Always be cautious about where you store and how you share such sensitive information.

Using either of these methods, you can generate the hashed password needed for secure, non‑interactive user creation in cloud‑init.

**Swapan**: how to generate a unique password save it in hashed form  all inside cloud-init then print out the generated password

**ChatGPT**: **Considering cloud-init password generation**

The user wants to generate a unique password via cloud-init, save it in hashed form, and then print the plain text password. Cloud-init, which configures Linux instances, uses YAML files in its modules. It'll generate a random password on the first boot, hash it for storage, and then display it in plain text. The question is interesting because it requires both secure password handling (hashing) and revealing the original password within a cloud-init config.

**Exploring cloud-init password generation**

Cloud-init uses "random_seed" to generate randomness, but I’m not sure it can generate a full random password. A potential workaround could be using a "runcmd" section to run a shell command that generates a password, hashes it, and then prints it. I could consider using tools like `openssl rand` to create the password, then hash it and print the result. Writing it to a file like `/var/log/generated_password.txt` might make sense too, especially since cloud-init runs only once. But the question specifies generating the password for a non-root user, which I'm still thinking through.

**Generating password in cloud-init**

To generate a random password in cloud-init, we can’t directly do it in the "users:" section, but we can use "runcmd" to create a shell script. I can generate a password using `openssl rand -base64 12`, then hash it with `openssl passwd -6`. After that, I’ll use `usermod` to assign the password to a non-root user. Finally, I’ll print the password using `tee` to a file, like `/var/log/generated_password.txt`. This approach achieves the user’s goal while remembering security concerns about storing passwords in plain text.

**ChatGPT**: Below is an example of a cloud‑init configuration that, during first boot, generates a random password for a non‑root user (named `nonroot`), computes its hashed version, sets the password for that user, and then prints (and saves) the generated plain‑text password. (Keep in mind that printing the plain‑text password can be a security risk, so use this technique only in a controlled environment.)

```yaml
#cloud-config
# Basic package update/upgrade
apt_update: true
apt_upgrade: true

# Create the non-root user without an initial password
users:
  - name: nonroot
    gecos: "Non-root User"
    groups: [sudo, docker]
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    lock_passwd: false

runcmd:
  - |
    #!/bin/bash
    # Generate a unique random password (12 bytes, base64 encoded)
    GENERATED_PASSWORD=$(openssl rand -base64 12)
    
    # Generate a SHA-512 hashed version of the password.
    # The -6 option tells openssl to use SHA-512.
    HASHED_PASSWORD=$(openssl passwd -6 "$GENERATED_PASSWORD")
    
    # Set the password for the nonroot user using the hashed value.
    usermod --password "$HASHED_PASSWORD" nonroot
    
    # Output the generated plain-text password.
    # This prints it to the console and saves it to a file for later reference.
    echo "Generated password for nonroot is: $GENERATED_PASSWORD" | tee /var/log/initial_nonroot_password.txt
```

---

### How It Works

1. **User Creation:**  
   The `users:` section creates the user `nonroot` (if not already present) with sudo and docker privileges but does not set a password yet.

2. **Random Password Generation:**  
   In the `runcmd:` section (which runs at the end of the cloud‑init process), a shell script is executed that uses:
   - `openssl rand -base64 12`  
     to generate a 12‑byte random value encoded in base64 (which is typically around 16 characters long). This serves as the unique plain‑text password.

3. **Password Hashing:**  
   The script then pipes the generated password into `openssl passwd -6` to create a hashed version using the SHA‑512 algorithm. The resulting hash is in the format expected by Linux (usually starting with `$6$`).

4. **Setting the Password:**  
   The command  
   ```bash
   usermod --password "$HASHED_PASSWORD" nonroot
   ```  
   sets the `nonroot` user’s password to the hashed version.

5. **Printing and Saving the Password:**  
   Finally, the generated plain‑text password is printed to the console and saved into `/var/log/initial_nonroot_password.txt` using `tee`.

---

### Security Considerations

- **Exposure of Plain‑Text Password:**  
  The generated password is printed and stored in a file. In a production environment, you should avoid exposing plain‑text passwords. This example is best suited for controlled environments (for example, temporary test instances) or when you have another secure mechanism to retrieve the generated password.

- **Cloud‑init and Re-runs:**  
  Note that cloud‑init typically runs only once (on first boot). If you rebuild your image or instance repeatedly, a new password will be generated each time.

This configuration lets you fully automate the creation of a non‑root user with a unique password generated, hashed, applied, and then reported—all within a single cloud‑init configuration file.

**Swapan**: add functionality to install and configure the docker daemon

**ChatGPT**: **Configuring docker daemon**

The user wants to add functionality to install and configure Docker to an existing cloud-init config, which already generates a unique hashed password and configures a user. To achieve the user's goal, I'll extend the cloud-init config by adding steps to update and upgrade the system (`apt_update`, `apt_upgrade: true`), then install Docker packages. The question is whether to use `docker-ce` or `docker.io`. Since the previous config used `docker.io`, it seems safer to stick with that for now. That’s where I’ll start.

**Configuring Docker Daemon**

The user specifically mentioned the "docker daemon," so I should focus on installing the Docker engine and configuring it. However, we also have to consider whether to use `docker.io` or `docker-ce`—`docker.io` seems like the safer option for simplicity, since it's available in the Ubuntu repositories. After installing Docker, I’ll need to configure the daemon by either editing `/etc/docker/daemon.json` or adding start commands for the service. Ideally, I’ll focus on adding appropriate configuration flags, such as `"experimental": true`, into the file.

**ChatGPT**: Below is an example cloud‑init configuration that does the following:

1. **Updates and upgrades packages.**
2. **Installs Docker** (using the `docker.io` package and its prerequisites).
3. **Creates a non‑root user** (`nonroot`) without an initial password.
4. **Generates a unique random password, hashes it, applies it to the user, and prints the plain‑text password** (saving it to a log file).
5. **Configures the Docker daemon** by writing a sample configuration file (`/etc/docker/daemon.json`).
6. **Restarts the Docker daemon** to apply the new configuration.

> **Security Note:**  
> - Storing and printing plain‑text passwords carries risks. Use such configurations only in controlled or testing environments.  
> - In WSL, Docker daemon management can be challenging since systemd is not enabled by default. Newer versions of WSL can enable systemd via `/etc/wsl.conf`.  
> - Adjust the Docker daemon configuration as needed for your environment.

Save the following content (for example, as `cloud-init.yaml`), and use it with your custom WSL‑Ubuntu image or during first‑boot provisioning.

---

```yaml
#cloud-config
# 1. Update and upgrade packages
apt_update: true
apt_upgrade: true

# 2. Install necessary packages including Docker
packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg-agent
  - software-properties-common
  - docker.io

# 3. Create a non-root user (without an initial password)
users:
  - name: nonroot
    gecos: "Non-root User"
    groups: [sudo, docker]
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    lock_passwd: false

runcmd:
  # 4. Generate a random password, hash it, apply it, and output it.
  - |
    #!/bin/bash
    # Generate a unique random password (12 bytes, base64 encoded)
    GENERATED_PASSWORD=$(openssl rand -base64 12)
    # Generate a SHA-512 hashed version of the password (-6 specifies SHA-512)
    HASHED_PASSWORD=$(openssl passwd -6 "$GENERATED_PASSWORD")
    # Set the password for the 'nonroot' user
    usermod --password "$HASHED_PASSWORD" nonroot
    # Output the generated password to console and save it in a log file
    echo "Generated password for nonroot is: $GENERATED_PASSWORD" | tee /var/log/initial_nonroot_password.txt

  # 5. Configure the Docker daemon by writing a configuration file
  - |
    #!/bin/bash
    # Create /etc/docker directory if it doesn't exist
    mkdir -p /etc/docker
    # Write a sample Docker daemon configuration file
    cat <<EOF > /etc/docker/daemon.json
{
  "experimental": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

  # 6. Restart Docker daemon to apply configuration changes
  - |
    #!/bin/bash
    # Try to restart Docker. If it isn't running, start it.
    service docker restart || service docker start
```

---

### How It Works

1. **Package Management:**  
   The `apt_update` and `apt_upgrade` keys ensure the system is updated before installing packages. The `packages` list installs Docker (using `docker.io`) and additional required utilities.

2. **User Creation:**  
   The `users:` section creates a non‑root user named `nonroot` (with sudo and Docker group membership) but does not assign a password immediately.

3. **Random Password Generation:**  
   In the first `runcmd` block, a Bash script generates a random password using `openssl rand -base64 12`, hashes it with `openssl passwd -6`, and applies it to the `nonroot` account. The plain‑text password is printed to the console and saved to `/var/log/initial_nonroot_password.txt`.

4. **Docker Daemon Configuration:**  
   The second `runcmd` block creates (if needed) the `/etc/docker` directory and writes a sample JSON configuration to `/etc/docker/daemon.json`. You can customize this file to suit your requirements.

5. **Restarting Docker:**  
   The third `runcmd` block restarts the Docker service so that the new configuration is loaded. If Docker isn’t running, it attempts to start it.

This configuration fully automates the provisioning of a WSL‑Ubuntu instance with Docker installed and configured, along with a non‑root user secured by a unique generated password.

