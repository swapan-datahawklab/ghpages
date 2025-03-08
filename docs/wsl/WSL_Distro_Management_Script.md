**Swapan**: for below just have three options create, delete, re-create

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

**ChatGPT**: **Clarifying user instructions**

The user wants a PowerShell script with just three options: "create," "delete," and "re-create." The provided script seems focused on handling WSL distributions, with prompts to delete or recreate a distribution. 

Given the instruction, it seems I need to simplify the script by limiting it to these three actions: create, delete, and re-create. I’ll need to remove any unnecessary prompts or actions that don’t align with these options. Let's refine the code accordingly.

**Simplifying script options**

Okay, the user wants a script with three clear options: "create," "delete," and "re-create." The provided code fetches WSL distributions and asks for deletion and re-creation, but the user seems to want something simpler.

The approach should be: prompt the user for one of three choices, then execute the corresponding option. "Re-create" can be simplified as delete, then create. I'll incorporate a menu or switch statement to organize these options into a neat, easy-to-follow flow. Let's simplify the script based on this understanding!

**Designing a new script**

The user wants a menu with three options: "create," "delete," and "re-create." For "create," it should probably prompt for a tarball URL, installation directory, and WSL version, and then run something like `wsl --import`.

For "delete," it’ll list distributions and let the user delete one, similar to the existing code. 

For "re-create," it should first delete a distro, then ask for details to re-create it. I'll adjust the script to handle these three options clearly, simplifying the flow.

**Designing simplified script**

I’m planning to simplify the script with three options: "Create," "Delete," and "Re-create." 

For "Create," the user will be prompted for necessary details like the distro name, tarball URL, installation directory, and WSL version. This will then create the file similarly to the current re-create logic. 

"Delete" will list distributions, let the user pick one, and then delete it. 

"Re-create" will follow the same flow as delete, but with additional prompts to handle re-importing.

I'll also keep functions like `Check-WSLInstallation` and `Get-WSLDistros` to manage distributions.

**Designing script functions**

I’m creating a PowerShell script that includes three options: "Create," "Delete," and "Re-create." 

For each option, I structure a clear process: the functions will prompt for the necessary details, perform the corresponding action (like importing, deleting, or re-creating distributions), and provide user feedback. 

The main logic checks WSL setup, then presents a menu for the user. I also ensure proper error handling — for example, checking if directories or files exist before acting on them.

Let me finalize the script now!

**ChatGPT**: Below is an updated version of your PowerShell script that presents a simple menu with three options: **Create**, **Delete**, and **Re-create** a WSL distribution. You can run the script, choose the desired option by entering the corresponding number, and follow the prompts. (Remember to run PowerShell as an administrator.)

```powershell
<#
.SYNOPSIS
    Manages WSL distributions with three options: Create, Delete, and Re-create.

.DESCRIPTION
    This script will:
      - Check that WSL is available.
      - Present a menu with three options:
          1. Create a new distribution.
          2. Delete an existing distribution.
          3. Re-create an existing distribution (delete then import again).
      - For creation/re-creation, the user is prompted for:
          • Distribution name (or, in re-create, the selected distro’s name is used)
          • Tarball URL (to download a Linux rootfs)
          • Installation directory (with a default value)
          • Desired WSL version (default is 2)
      - The tarball is downloaded, the distro imported via `wsl --import`, and its version set.

.NOTES
    Run PowerShell as an administrator.
    Unregistering (deleting) a distribution deletes its data permanently.
#>

# Function: Check if WSL is available
function Check-WSLInstallation {
    if (-not (Get-Command wsl.exe -ErrorAction SilentlyContinue)) {
        Write-Error "WSL (wsl.exe) is not installed or not in PATH. Please install WSL and try again."
        exit 1
    }
}

Check-WSLInstallation

# Function: Retrieve list of installed WSL distributions
function Get-WSLDistros {
    Write-Host "Fetching list of WSL distributions..."
    $wslListRaw = wsl -l -v | Select-Object -Skip 1 | ForEach-Object { $_.Trim() }
    $distros = @()
    foreach ($line in $wslListRaw) {
        if ([string]::IsNullOrWhiteSpace($line)) { continue }
        $columns = $line -split "\s+"
        # Remove a leading asterisk (*) if present (default distro indicator)
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

# Function: Create a new WSL distribution
function Create-WSLDistro {
    Write-Host "`n=== Create WSL Distribution ==="
    $name = Read-Host "Enter the new distribution name"
    if ([string]::IsNullOrWhiteSpace($name)) {
        Write-Error "Distribution name cannot be empty."
        return
    }
    $tarballUrl = Read-Host "Enter the tarball URL to import the distribution (e.g., https://aka.ms/wsl-ubuntu-2004)"
    if ([string]::IsNullOrWhiteSpace($tarballUrl)) {
        Write-Error "Tarball URL cannot be empty."
        return
    }
    $defaultInstallDir = "$env:LOCALAPPDATA\WSL\$name"
    $installLocation = Read-Host "Enter the installation directory for '$name' (default: $defaultInstallDir)"
    if ([string]::IsNullOrWhiteSpace($installLocation)) {
        $installLocation = $defaultInstallDir
    }
    $wslVersionInput = Read-Host "Enter the WSL version to set for '$name' (default is 2)"
    if ([string]::IsNullOrWhiteSpace($wslVersionInput)) {
        $wslVersion = 2
    }
    else {
        if (-not [int]::TryParse($wslVersionInput, [ref]$null)) {
            Write-Error "Invalid WSL version entered."
            return
        }
        $wslVersion = [int]$wslVersionInput
    }

    # Create installation directory if it doesn't exist
    if (-not (Test-Path -Path $installLocation)) {
        Write-Host "Creating installation directory at '$installLocation'..."
        New-Item -ItemType Directory -Path $installLocation | Out-Null
    }

    # Download tarball to a temporary file
    $tarballFile = Join-Path -Path $env:TEMP -ChildPath "$name.tar.gz"
    if (Test-Path $tarballFile) {
        Write-Host "Deleting existing tarball file at '$tarballFile'..."
        Remove-Item $tarballFile -Force
    }
    Write-Host "Downloading tarball from '$tarballUrl' to '$tarballFile'..."
    try {
        Invoke-WebRequest -Uri $tarballUrl -OutFile $tarballFile -UseBasicParsing
        Write-Host "Tarball downloaded successfully."
    }
    catch {
        Write-Error "Failed to download tarball from '$tarballUrl'."
        return
    }

    # Import the distribution using the downloaded tarball
    Write-Host "Importing distribution '$name' into WSL..."
    try {
        wsl --import $name $installLocation $tarballFile
        Write-Host "Import of '$name' successful."
    }
    catch {
        Write-Error "Failed to import the distribution '$name'."
        return
    }

    # Set the desired WSL version
    Write-Host "Setting WSL version for '$name' to $wslVersion..."
    try {
        wsl --set-version $name $wslVersion
        Write-Host "WSL version set successfully."
    }
    catch {
        Write-Warning "Failed to set WSL version for '$name'."
    }
    Write-Host "`nOperation completed: Distribution '$name' has been created successfully."
}

# Function: Delete an existing WSL distribution
function Delete-WSLDistro {
    Write-Host "`n=== Delete WSL Distribution ==="
    $distros = Get-WSLDistros
    if ($distros.Count -eq 0) {
        Write-Host "No WSL distributions found."
        return
    }
    Write-Host "`nAvailable WSL Distributions:"
    for ($i = 0; $i -lt $distros.Count; $i++) {
        Write-Host ("{0}. {1} - State: {2}, Version: {3}" -f ($i + 1), $distros[$i].Name, $distros[$i].State, $distros[$i].Version)
    }
    $choice = Read-Host "`nEnter the number of the distribution you want to delete"
    if (-not [int]::TryParse($choice, [ref]$null)) {
        Write-Error "Invalid input. Please enter a valid number."
        return
    }
    $index = [int]$choice - 1
    if ($index -lt 0 -or $index -ge $distros.Count) {
        Write-Error "Selection out of range."
        return
    }
    $selectedDistro = $distros[$index]
    Write-Host "`nYou selected: $($selectedDistro.Name) - State: $($selectedDistro.State), Version: $($selectedDistro.Version)"
    $confirm = Read-Host "Are you sure you want to unregister (delete) the distribution '$($selectedDistro.Name)'? (Y/N)"
    if ($confirm -notin @("Y", "y")) {
        Write-Host "Operation cancelled."
        return
    }
    Write-Host "Unregistering distribution '$($selectedDistro.Name)'..."
    try {
        wsl --unregister $selectedDistro.Name
        Write-Host "Distribution '$($selectedDistro.Name)' has been unregistered successfully."
    }
    catch {
        Write-Error "Failed to unregister distribution '$($selectedDistro.Name)'."
    }
}

# Function: Re-create an existing WSL distribution (delete then create)
function Recreate-WSLDistro {
    Write-Host "`n=== Re-create WSL Distribution ==="
    $distros = Get-WSLDistros
    if ($distros.Count -eq 0) {
        Write-Host "No WSL distributions found."
        return
    }
    Write-Host "`nAvailable WSL Distributions:"
    for ($i = 0; $i -lt $distros.Count; $i++) {
        Write-Host ("{0}. {1} - State: {2}, Version: {3}" -f ($i + 1), $distros[$i].Name, $distros[$i].State, $distros[$i].Version)
    }
    $choice = Read-Host "`nEnter the number of the distribution you want to re-create"
    if (-not [int]::TryParse($choice, [ref]$null)) {
        Write-Error "Invalid input. Please enter a valid number."
        return
    }
    $index = [int]$choice - 1
    if ($index -lt 0 -or $index -ge $distros.Count) {
        Write-Error "Selection out of range."
        return
    }
    $selectedDistro = $distros[$index]
    Write-Host "`nYou selected: $($selectedDistro.Name) - State: $($selectedDistro.State), Version: $($selectedDistro.Version)"
    $confirm = Read-Host "Are you sure you want to unregister (delete) the distribution '$($selectedDistro.Name)'? (Y/N)"
    if ($confirm -notin @("Y", "y")) {
        Write-Host "Operation cancelled."
        return
    }
    Write-Host "Unregistering distribution '$($selectedDistro.Name)'..."
    try {
        wsl --unregister $selectedDistro.Name
        Write-Host "Distribution '$($selectedDistro.Name)' has been unregistered successfully."
    }
    catch {
        Write-Error "Failed to unregister distribution '$($selectedDistro.Name)'."
        return
    }

    # Re-create process
    $tarballUrl = Read-Host "Enter the tarball URL to import the distribution (e.g., https://aka.ms/wsl-ubuntu-2004)"
    if ([string]::IsNullOrWhiteSpace($tarballUrl)) {
        Write-Error "Tarball URL cannot be empty."
        return
    }
    $defaultInstallDir = "$env:LOCALAPPDATA\WSL\$($selectedDistro.Name)"
    $installLocation = Read-Host "Enter the installation directory for '$($selectedDistro.Name)' (default: $defaultInstallDir)"
    if ([string]::IsNullOrWhiteSpace($installLocation)) {
        $installLocation = $defaultInstallDir
    }
    $wslVersionInput = Read-Host "Enter the WSL version to set for '$($selectedDistro.Name)' (default is 2)"
    if ([string]::IsNullOrWhiteSpace($wslVersionInput)) {
        $wslVersion = 2
    }
    else {
        if (-not [int]::TryParse($wslVersionInput, [ref]$null)) {
            Write-Error "Invalid WSL version entered."
            return
        }
        $wslVersion = [int]$wslVersionInput
    }
    if (-not (Test-Path -Path $installLocation)) {
        Write-Host "Creating installation directory at '$installLocation'..."
        New-Item -ItemType Directory -Path $installLocation | Out-Null
    }
    $tarballFile = Join-Path -Path $env:TEMP -ChildPath "$($selectedDistro.Name).tar.gz"
    if (Test-Path $tarballFile) {
        Write-Host "Deleting existing tarball file at '$tarballFile'..."
        Remove-Item $tarballFile -Force
    }
    Write-Host "Downloading tarball from '$tarballUrl' to '$tarballFile'..."
    try {
        Invoke-WebRequest -Uri $tarballUrl -OutFile $tarballFile -UseBasicParsing
        Write-Host "Tarball downloaded successfully."
    }
    catch {
        Write-Error "Failed to download tarball from '$tarballUrl'."
        return
    }
    Write-Host "Importing distribution '$($selectedDistro.Name)' into WSL..."
    try {
        wsl --import $selectedDistro.Name $installLocation $tarballFile
        Write-Host "Import of '$($selectedDistro.Name)' successful."
    }
    catch {
        Write-Error "Failed to import the distribution '$($selectedDistro.Name)'."
        return
    }
    Write-Host "Setting WSL version for '$($selectedDistro.Name)' to $wslVersion..."
    try {
        wsl --set-version $selectedDistro.Name $wslVersion
        Write-Host "WSL version set successfully."
    }
    catch {
        Write-Warning "Failed to set WSL version for '$($selectedDistro.Name)'."
    }
    Write-Host "`nOperation completed: Distribution '$($selectedDistro.Name)' has been re-created successfully."
}

# Main Menu
Write-Host "=== WSL Distribution Manager ==="
Write-Host "1. Create a new distribution"
Write-Host "2. Delete an existing distribution"
Write-Host "3. Re-create an existing distribution"
$option = Read-Host "Enter the number of your choice"

switch ($option) {
    "1" { Create-WSLDistro }
    "2" { Delete-WSLDistro }
    "3" { Recreate-WSLDistro }
    default { Write-Host "Invalid option selected. Exiting." }
}
```

### How It Works

1. **Create (Option 1):**  
   - Prompts for a new distribution name, tarball URL, installation directory (default is under your `%LOCALAPPDATA%\WSL\` folder), and WSL version.  
   - Downloads the tarball, imports the distro, and sets the version.

2. **Delete (Option 2):**  
   - Retrieves and displays your installed WSL distributions.  
   - Lets you select one by number and confirms before calling `wsl --unregister` to delete it.

3. **Re-create (Option 3):**  
   - Lists the existing distributions and allows you to select one.  
   - After confirmation, it deletes the selected distro and then asks for tarball URL, installation directory, and WSL version to import it back.

Feel free to adjust any of the prompts or default values to suit your needs. Enjoy managing your WSL distributions with these three straightforward options!

**Swapan**: wsl tarball url for direct download

**ChatGPT**: **Looking into WSL tarball download**

The user might be asking for a direct download link for a WSL tarball to use in a script. From what I know, the URL like `https://aka.ms/wsl-ubuntu-2004` redirects to a tarball file that can be downloaded directly. Other WSL distributions, such as Ubuntu 18.04, can be found at links like `https://aka.ms/wsl-ubuntu-1804`. It seems WSL provides these images on the official site directly. So, I think this could answer the user's need!

**Listing WSL tarball links**

I might need to provide a list of known WSL tarball URLs for the user. Some examples I recall are:

- Ubuntu 20.04: https://aka.ms/wsl-ubuntu-2004 (direct download for the tarball)
- Ubuntu 18.04: https://aka.ms/wsl-ubuntu-1804
- Ubuntu 22.04: possibly https://aka.ms/wsl-ubuntu-2204
- Debian: https://aka.ms/wsl-debian-gnu-linux
- Kali: https://aka.ms/wsl-kali-linux-new
- openSUSE-42: https://aka.ms/wsl-opensuse-42
- SLES: https://aka.ms/wsl-sles-12

It seems like https://aka.ms/wsl-ubuntu-2004 is already the direct link for Ubuntu 20.04’s tarball.

**ChatGPT**: Below are some example direct-download tarball URLs provided by Microsoft for popular WSL distributions. You can use these URLs directly with commands like `Invoke-WebRequest` or as a parameter to the `wsl --import` command:

- **Ubuntu 20.04:**  
  `https://aka.ms/wsl-ubuntu-2004`

- **Ubuntu 18.04:**  
  `https://aka.ms/wsl-ubuntu-1804`

- **Debian:**  
  `https://aka.ms/wsl-debian-gnu-linux`

- **Kali Linux:**  
  `https://aka.ms/wsl-kali-linux-new`

> **Note:**  
> These URLs are maintained by Microsoft and may change over time. Be sure to verify the current download link from the official [WSL documentation](https://docs.microsoft.com/windows/wsl/install-manual) or the distribution’s page if you run into issues.

You can use any of these URLs in your script or download process to directly retrieve the WSL tarball.

