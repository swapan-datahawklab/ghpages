**Swapan**: kick off bash script after installing wsl2 ubuntu distribution

**ChatGPT**: To automatically kick off a bash script after installing a WSL2 Ubuntu distribution, you can follow these steps:

### 1. **Ensure the Script is Ready**
- Save your bash script (e.g., `setup.sh`) in a location that will be accessible to the WSL instance, such as your Windows file system or within the WSL file system after installation.

### 2. **Use the `wsl.exe` Command**
The `wsl.exe` command-line tool allows you to execute commands in a WSL distribution from Windows.

### 3. **Run the Script Automatically**
You can use a PowerShell or Batch script to trigger the WSL instance and run your script. Here's how:

#### **Option 1: PowerShell Script**
Create a PowerShell script, e.g., `run-wsl-script.ps1`, with the following content:

```powershell
# Define the WSL distribution name (default is "Ubuntu" unless renamed)
$wslDistro = "Ubuntu"

# Path to the script you want to run (use WSL path or Windows path)
$scriptPath = "/mnt/c/path-to-your-script/setup.sh"

# Launch WSL and execute the script
wsl -d $wslDistro bash -c "$scriptPath"
```

#### **Option 2: Batch File**
If you prefer a batch file, create `run-wsl-script.bat`:

```batch
@echo off
wsl -d Ubuntu bash -c "/mnt/c/path-to-your-script/setup.sh"
```

### 4. **Automate Execution**
You can set up this script to run automatically after WSL is installed.

#### **a. Run Manually After Installation**
After installing WSL2, you can run the PowerShell or Batch script manually to kick off your setup script.

#### **b. Use Scheduled Task or Post-Install Hook**
- If you're automating the installation of WSL2 and the Ubuntu distribution, you can append the script execution step in your installation workflow.
- Example: After installing WSL2 using PowerShell, run your script in the same script.

#### Example:
```powershell
# Install WSL and Ubuntu
wsl --install -d Ubuntu

# Wait for installation to complete
Start-Sleep -Seconds 10

# Run the bash script
wsl -d Ubuntu bash -c "/mnt/c/path-to-your-script/setup.sh"
```

### 5. **Verify Script Execution**
After installation, open your WSL terminal or check the script's log output to ensure it executed correctly.

Let me know if you need help creating the script or automating further!

**Swapan**: get list of available ubuntu distributions with wsl -l -o prompt for which version to install then install with custom name

**ChatGPT**: Here’s a PowerShell script that lists the available Ubuntu distributions, prompts the user to select one, and then installs it with a custom name:

### PowerShell Script

```powershell
# Get the list of available WSL distributions
Write-Host "Fetching available WSL distributions..."
$distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"

# Extract distribution names from the output
$ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }

# Display available distributions
Write-Host "Available Ubuntu distributions:"
for ($i = 0; $i -lt $ubuntuDistros.Count; $i++) {
    Write-Host "$($i + 1). $($ubuntuDistros[$i])"
}

# Prompt user to choose a distribution
$choice = Read-Host "Enter the number of the Ubuntu version you want to install"

# Validate the user's choice
if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $ubuntuDistros.Count) {
    $selectedDistro = $ubuntuDistros[$choice - 1]
    Write-Host "You selected: $selectedDistro"

    # Prompt for a custom name
    $customName = Read-Host "Enter a custom name for the distribution"

    # Install the selected distribution with the custom name
    Write-Host "Installing $selectedDistro with the name '$customName'..."
    wsl --install -d $selectedDistro --name $customName

    Write-Host "Installation initiated for $customName."
} else {
    Write-Host "Invalid selection. Please run the script again and select a valid option."
}
```

### Explanation

1. **Fetch Available Distributions**:
   - The script uses `wsl --list --online` to get a list of available distributions and filters the ones containing "Ubuntu."

2. **User Prompt**:
   - The user is presented with a numbered list of available Ubuntu distributions and is asked to select one.

3. **Custom Name**:
   - The script prompts the user to provide a custom name for the WSL distribution.

4. **Installation**:
   - The selected distribution is installed using `wsl --install -d <distribution>` with the `--name` parameter for the custom name.

### Notes:
- **Administrator Privileges**: Ensure you run the PowerShell script as an administrator.
- **WSL Version**: This script assumes WSL 2 is installed and configured.

Let me know if you need further adjustments or enhancements!

**Swapan**: check out a bash script from github run it as part of wsl -d $customName bash -c "/mnt/c/path-to-your-script/setup.sh"

**ChatGPT**: To create a PowerShell script that clones a Bash script from GitHub, runs it as part of a WSL distribution with a custom name, and executes the script, follow these steps:

### PowerShell Script

```powershell
# Prompt user to enter the custom WSL distribution name
$customName = Read-Host "Enter the name of the WSL distribution to use"

# Prompt user for the GitHub repository URL
$githubRepo = Read-Host "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git)"

# Prompt user for the path of the Bash script inside the repository
$scriptPathInRepo = Read-Host "Enter the relative path of the Bash script inside the repository (e.g., setup.sh)"

# Define a temporary folder to clone the GitHub repository
$tempFolder = "$env:USERPROFILE\wsl-temp-repo"
if (Test-Path $tempFolder) {
    Remove-Item -Recurse -Force $tempFolder
}
New-Item -ItemType Directory -Path $tempFolder | Out-Null

# Clone the GitHub repository
Write-Host "Cloning GitHub repository..."
git clone $githubRepo $tempFolder

if (!(Test-Path "$tempFolder\$scriptPathInRepo")) {
    Write-Host "Error: Script not found at the specified path in the repository." -ForegroundColor Red
    Exit
}

# Convert Windows path to WSL-compatible path
$scriptPathWSL = "/mnt/" + ($tempFolder -replace ":", "").Replace("\", "/") + "/" + $scriptPathInRepo

# Run the Bash script in the specified WSL distribution
Write-Host "Running the script in WSL ($customName)..."
wsl -d $customName bash -c "$scriptPathWSL"

# Clean up temporary folder
Write-Host "Cleaning up temporary files..."
Remove-Item -Recurse -Force $tempFolder

Write-Host "Script execution completed."
```

### Steps the Script Performs:
1. **Prompt for Custom WSL Name**:
   - The user enters the name of the WSL distribution (e.g., `MyUbuntu`).

2. **Prompt for GitHub Repository and Script Path**:
   - The user provides the URL of the GitHub repository and the relative path to the Bash script (e.g., `setup.sh`).

3. **Clone Repository**:
   - The script clones the GitHub repository into a temporary folder on the host Windows file system.

4. **Run Bash Script in WSL**:
   - It converts the local Windows path to the script into a WSL-compatible path and runs it inside the specified WSL distribution.

5. **Cleanup**:
   - After execution, the temporary folder is deleted.

### Prerequisites:
1. **Git**: Ensure Git is installed on your Windows system.
2. **WSL**: Verify that WSL and the specified distribution are installed and configured.

### Example Usage:
- If the custom WSL name is `MyUbuntu`, the GitHub repository is `https://github.com/username/repo.git`, and the script is located at `scripts/setup.sh`, the script will:
   1. Clone the repository to a temporary folder.
   2. Run the `setup.sh` script in the `MyUbuntu` WSL instance.

Let me know if you need any further assistance!

**Swapan**: convert below into functions

# Get the list of available WSL distributions
Write-Host "Fetching available WSL distributions..."
$distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"

# Extract distribution names from the output
$ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }

# Display available distributions
Write-Host "Available Ubuntu distributions:"
for ($i = 0; $i -lt $ubuntuDistros.Count; $i++) {
    Write-Host "$($i + 1). $($ubuntuDistros[$i])"
}

# Prompt user to choose a distribution
$choice = Read-Host "Enter the number of the Ubuntu version you want to install"

# Validate the user's choice
if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $ubuntuDistros.Count) {
    $selectedDistro = $ubuntuDistros[$choice - 1]
    Write-Host "You selected: $selectedDistro"

    # Prompt for a custom name
    $customName = Read-Host "Enter a custom name for the distribution"

    # Install the selected distribution with the custom name
    Write-Host "Installing $selectedDistro with the name '$customName'..."
    wsl --install -d $selectedDistro --name $customName

    Write-Host "Installation initiated for $customName."
} else {
    Write-Host "Invalid selection. Please run the script again and select a valid option."
}

**ChatGPT**: Here is the code refactored into functions to improve readability and reusability:

```powershell
# Function to get the list of available WSL distributions
function Get-AvailableDistributions {
    Write-Host "Fetching available WSL distributions..."
    $distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"
    $ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }
    return $ubuntuDistros
}

# Function to display available distributions
function Display-Distributions {
    param (
        [array]$distributions
    )
    Write-Host "Available Ubuntu distributions:"
    for ($i = 0; $i -lt $distributions.Count; $i++) {
        Write-Host "$($i + 1). $($distributions[$i])"
    }
}

# Function to prompt the user to select a distribution
function Get-UserSelection {
    param (
        [array]$distributions
    )
    $choice = Read-Host "Enter the number of the Ubuntu version you want to install"
    if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $distributions.Count) {
        return $distributions[$choice - 1]
    } else {
        Write-Host "Invalid selection. Please run the script again and select a valid option." -ForegroundColor Red
        return $null
    }
}

# Function to install the selected distribution with a custom name
function Install-Distribution {
    param (
        [string]$distribution,
        [string]$customName
    )
    Write-Host "Installing $distribution with the name '$customName'..."
    wsl --install -d $distribution --name $customName
    Write-Host "Installation initiated for $customName."
}

# Main script logic
$ubuntuDistros = Get-AvailableDistributions
if ($ubuntuDistros.Count -eq 0) {
    Write-Host "No available Ubuntu distributions found." -ForegroundColor Red
    exit
}

Display-Distributions -distributions $ubuntuDistros
$selectedDistro = Get-UserSelection -distributions $ubuntuDistros

if ($selectedDistro) {
    Write-Host "You selected: $selectedDistro"
    $customName = Read-Host "Enter a custom name for the distribution"
    Install-Distribution -distribution $selectedDistro -customName $customName
}
```

### How It Works:
1. **Functions**:
   - **`Get-AvailableDistributions`**: Fetches and returns the list of Ubuntu distributions available for WSL installation.
   - **`Display-Distributions`**: Displays the list of available distributions in a numbered format.
   - **`Get-UserSelection`**: Prompts the user to select a distribution and validates their input.
   - **`Install-Distribution`**: Installs the selected distribution with the custom name.

2. **Main Script**:
   - Calls the functions in sequence to fetch, display, and process the user's selection, and installs the chosen distribution.

### Benefits:
- **Modularity**: Each function handles a specific task, making the code easier to read and maintain.
- **Reusability**: Functions can be reused in other scripts if needed.
- **Error Handling**: Invalid input is handled gracefully.

Let me know if you'd like any further enhancements!

**Swapan**: convert this into functions

# Prompt user to enter the custom WSL distribution name
$customName = Read-Host "Enter the name of the WSL distribution to use"

# Prompt user for the GitHub repository URL
$githubRepo = Read-Host "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git)"

# Prompt user for the path of the Bash script inside the repository
$scriptPathInRepo = Read-Host "Enter the relative path of the Bash script inside the repository (e.g., setup.sh)"

# Define a temporary folder to clone the GitHub repository
$tempFolder = "$env:USERPROFILE\wsl-temp-repo"
if (Test-Path $tempFolder) {
    Remove-Item -Recurse -Force $tempFolder
}
New-Item -ItemType Directory -Path $tempFolder | Out-Null

# Clone the GitHub repository
Write-Host "Cloning GitHub repository..."
git clone $githubRepo $tempFolder

if (!(Test-Path "$tempFolder\$scriptPathInRepo")) {
    Write-Host "Error: Script not found at the specified path in the repository." -ForegroundColor Red
    Exit
}

# Convert Windows path to WSL-compatible path
$scriptPathWSL = "/mnt/" + ($tempFolder -replace ":", "").Replace("\", "/") + "/" + $scriptPathInRepo

# Run the Bash script in the specified WSL distribution
Write-Host "Running the script in WSL ($customName)..."
wsl -d $customName bash -c "$scriptPathWSL"

# Clean up temporary folder
Write-Host "Cleaning up temporary files..."
Remove-Item -Recurse -Force $tempFolder

Write-Host "Script execution completed."

**ChatGPT**: Here's the refactored version of your code with functions for each task:

```powershell
# Function to prompt the user for input
function Prompt-UserInput {
    param (
        [string]$promptMessage
    )
    return Read-Host $promptMessage
}

# Function to create a temporary folder
function Create-TemporaryFolder {
    param (
        [string]$folderPath
    )
    if (Test-Path $folderPath) {
        Remove-Item -Recurse -Force $folderPath
    }
    New-Item -ItemType Directory -Path $folderPath | Out-Null
    return $folderPath
}

# Function to clone the GitHub repository
function Clone-GitHubRepository {
    param (
        [string]$repositoryUrl,
        [string]$destinationFolder
    )
    Write-Host "Cloning GitHub repository..."
    git clone $repositoryUrl $destinationFolder
}

# Function to validate the presence of the script in the cloned repository
function Validate-ScriptPath {
    param (
        [string]$folderPath,
        [string]$scriptPath
    )
    $fullPath = Join-Path -Path $folderPath -ChildPath $scriptPath
    if (!(Test-Path $fullPath)) {
        Write-Host "Error: Script not found at the specified path in the repository." -ForegroundColor Red
        Exit
    }
    return $fullPath
}

# Function to convert a Windows path to a WSL-compatible path
function Convert-ToWSLPath {
    param (
        [string]$windowsPath
    )
    return "/mnt/" + ($windowsPath -replace ":", "").Replace("\", "/")
}

# Function to run the Bash script in the specified WSL distribution
function Run-ScriptInWSL {
    param (
        [string]$distroName,
        [string]$scriptPath
    )
    Write-Host "Running the script in WSL ($distroName)..."
    wsl -d $distroName bash -c "$scriptPath"
}

# Function to clean up temporary files
function Clean-UpTemporaryFiles {
    param (
        [string]$folderPath
    )
    Write-Host "Cleaning up temporary files..."
    Remove-Item -Recurse -Force $folderPath
}

# Main script logic
$customName = Prompt-UserInput -promptMessage "Enter the name of the WSL distribution to use"
$githubRepo = Prompt-UserInput -promptMessage "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git)"
$scriptPathInRepo = Prompt-UserInput -promptMessage "Enter the relative path of the Bash script inside the repository (e.g., setup.sh)"

$tempFolder = Create-TemporaryFolder -folderPath "$env:USERPROFILE\wsl-temp-repo"

Clone-GitHubRepository -repositoryUrl $githubRepo -destinationFolder $tempFolder

$fullScriptPath = Validate-ScriptPath -folderPath $tempFolder -scriptPath $scriptPathInRepo

$wslScriptPath = Convert-ToWSLPath -windowsPath $fullScriptPath

Run-ScriptInWSL -distroName $customName -scriptPath $wslScriptPath

Clean-UpTemporaryFiles -folderPath $tempFolder

Write-Host "Script execution completed."
```

### Explanation of Functions:
1. **`Prompt-UserInput`**:
   - A generic function for prompting user input.
2. **`Create-TemporaryFolder`**:
   - Creates a temporary folder or removes the existing one.
3. **`Clone-GitHubRepository`**:
   - Clones the specified GitHub repository to the given destination folder.
4. **`Validate-ScriptPath`**:
   - Ensures the specified script exists in the cloned repository.
5. **`Convert-ToWSLPath`**:
   - Converts a Windows file path to a WSL-compatible path.
6. **`Run-ScriptInWSL`**:
   - Runs the script in the specified WSL distribution.
7. **`Clean-UpTemporaryFiles`**:
   - Cleans up the temporary folder created during the process.

### Benefits:
- **Reusability**: Individual functions can be reused in other scripts.
- **Readability**: The main script is cleaner and focuses on the overall workflow.
- **Modularity**: Tasks are separated into their own functions, making debugging and testing easier.

Let me know if you need further enhancements!

**Swapan**: convert below to a main function

# Main script logic
$customName = Prompt-UserInput -promptMessage "Enter the name of the WSL distribution to use"
$githubRepo = Prompt-UserInput -promptMessage "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git)"
$scriptPathInRepo = Prompt-UserInput -promptMessage "Enter the relative path of the Bash script inside the repository (e.g., setup.sh)"

$tempFolder = Create-TemporaryFolder -folderPath "$env:USERPROFILE\wsl-temp-repo"

Clone-GitHubRepository -repositoryUrl $githubRepo -destinationFolder $tempFolder

$fullScriptPath = Validate-ScriptPath -folderPath $tempFolder -scriptPath $scriptPathInRepo

$wslScriptPath = Convert-ToWSLPath -windowsPath $fullScriptPath

Run-ScriptInWSL -distroName $customName -scriptPath $wslScriptPath

Clean-UpTemporaryFiles -folderPath $tempFolder

Write-Host "Script execution completed."

**ChatGPT**: Here’s the updated script with the main logic encapsulated in a `Main` function:

```powershell
# Main function
function Main {
    # Prompt user for inputs
    $customName = Prompt-UserInput -promptMessage "Enter the name of the WSL distribution to use"
    $githubRepo = Prompt-UserInput -promptMessage "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git)"
    $scriptPathInRepo = Prompt-UserInput -promptMessage "Enter the relative path of the Bash script inside the repository (e.g., setup.sh)"
    
    # Create a temporary folder for cloning the repository
    $tempFolder = Create-TemporaryFolder -folderPath "$env:USERPROFILE\wsl-temp-repo"
    
    # Clone the GitHub repository
    Clone-GitHubRepository -repositoryUrl $githubRepo -destinationFolder $tempFolder
    
    # Validate the presence of the script in the cloned repository
    $fullScriptPath = Validate-ScriptPath -folderPath $tempFolder -scriptPath $scriptPathInRepo
    
    # Convert the Windows path of the script to a WSL-compatible path
    $wslScriptPath = Convert-ToWSLPath -windowsPath $fullScriptPath
    
    # Run the Bash script in the specified WSL distribution
    Run-ScriptInWSL -distroName $customName -scriptPath $wslScriptPath
    
    # Clean up the temporary folder
    Clean-UpTemporaryFiles -folderPath $tempFolder
    
    Write-Host "Script execution completed."
}

# Execute the main function
Main
```

### Key Updates:
1. **Encapsulation**:
   - The script logic is wrapped inside a `Main` function, which organizes the code and makes it easier to call or modify.
   
2. **Function Execution**:
   - The script now runs by simply calling `Main` at the end, allowing for easy testing and potential expansion.

3. **Benefits**:
   - **Readability**: All auxiliary tasks (e.g., input prompts, cleanup, validations) are now functions, and the main logic is clearly defined.
   - **Modularity**: You can now reuse or integrate this script into larger scripts or workflows by calling `Main`.

Feel free to modify the `Main` function as needed to add additional steps or handle edge cases! Let me know if you need any further refinements.

**Swapan**: merge the following: 

# Function to get the list of available WSL distributions
function Get-AvailableDistributions {
    Write-Host "Fetching available WSL distributions..."
    $distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"
    $ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }
    return $ubuntuDistros
}

# Function to display available distributions
function Show-Distributions {
    param (
        [array]$distributions
    )
    Write-Host "Available Ubuntu distributions:"
    for ($i = 0; $i -lt $distributions.Count; $i++) {
        Write-Host "$($i + 1). $($distributions[$i])"
    }
}

# Function to prompt the user to select a distribution
function Get-UserSelection {
    param (
        [array]$distributions
    )
    $choice = Read-Host "Enter the number of the Ubuntu version you want to install"
    if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $distributions.Count) {
        return $distributions[$choice - 1]
    } else {
        Write-Host "Invalid selection. Please run the script again and select a valid option." -ForegroundColor Red
        return $null
    }
}

# Function to install the selected distribution with a custom name
function Install-Distribution {
    param (
        [string]$distribution,
        [string]$customName
    )
    Write-Host "Installing $distribution with the name '$customName'..."
    wsl --install -d $distribution --name $customName
    Write-Host "Installation initiated for $customName."
}

# Main script logic
$ubuntuDistros = Get-AvailableDistributions
if ($ubuntuDistros.Count -eq 0) {
    Write-Host "No available Ubuntu distributions found." -ForegroundColor Red
    exit
}

Show-Distributions -distributions $ubuntuDistros
$selectedDistro = Get-UserSelection -distributions $ubuntuDistros

if ($selectedDistro) {
    Write-Host "You selected: $selectedDistro"
    $customName = Read-Host "Enter a custom name for the distribution"
    Install-Distribution -distribution $selectedDistro -customName $customName
}

**ChatGPT**: Here's the merged version of the script, combining all the related logic into a single, well-structured function:

```powershell
# Function to manage WSL distribution selection and installation
function Manage-WSLDistribution {
    # Fetch available WSL distributions
    Write-Host "Fetching available WSL distributions..."
    $distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"
    $ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }

    # Check if any distributions are available
    if ($ubuntuDistros.Count -eq 0) {
        Write-Host "No available Ubuntu distributions found." -ForegroundColor Red
        return
    }

    # Display available distributions
    Write-Host "Available Ubuntu distributions:"
    for ($i = 0; $i -lt $ubuntuDistros.Count; $i++) {
        Write-Host "$($i + 1). $($ubuntuDistros[$i])"
    }

    # Prompt user for selection
    $choice = Read-Host "Enter the number of the Ubuntu version you want to install"
    if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $ubuntuDistros.Count) {
        $selectedDistro = $ubuntuDistros[$choice - 1]
        Write-Host "You selected: $selectedDistro"

        # Prompt for a custom name
        $customName = Read-Host "Enter a custom name for the distribution"

        # Install the selected distribution with the custom name
        Write-Host "Installing $selectedDistro with the name '$customName'..."
        wsl --install -d $selectedDistro --name $customName
        Write-Host "Installation initiated for $customName."
    } else {
        Write-Host "Invalid selection. Please run the script again and select a valid option." -ForegroundColor Red
    }
}

# Execute the function
Manage-WSLDistribution
```

### Key Changes:
1. **Single Function**:
   - All logic is now encapsulated in the `Manage-WSLDistribution` function, making it easier to call and manage.
   
2. **Integrated Workflow**:
   - The individual steps (fetching, displaying, selecting, and installing distributions) are streamlined in the function, eliminating redundancy.

3. **Error Handling**:
   - The script gracefully handles cases where no distributions are available or the user provides invalid input.

### Usage:
Run the script, and it will:
1. Fetch and display available Ubuntu distributions.
2. Prompt the user to select a distribution.
3. Ask for a custom name and install the selected distribution with that name.

This approach ensures everything is logically grouped while maintaining modularity. Let me know if you need further adjustments!

**Swapan**: merge the following:

# Function to get the list of available WSL distributions
function Get-AvailableDistributions {
    Write-Host "Fetching available WSL distributions..."
    $distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"
    $ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }
    return $ubuntuDistros
}

# Function to display available distributions
function Show-Distributions {
    param (
        [array]$distributions
    )
    Write-Host "Available Ubuntu distributions:"
    for ($i = 0; $i -lt $distributions.Count; $i++) {
        Write-Host "$($i + 1). $($distributions[$i])"
    }
}

# Function to prompt the user to select a distribution
function Get-UserSelection {
    param (
        [array]$distributions
    )
    $choice = Read-Host "Enter the number of the Ubuntu version you want to install"
    if ($choice -match "^\d+$" -and $choice -ge 1 -and $choice -le $distributions.Count) {
        return $distributions[$choice - 1]
    } else {
        Write-Host "Invalid selection. Please run the script again and select a valid option." -ForegroundColor Red
        return $null
    }
}

# Function to install the selected distribution with a custom name
function Install-Distribution {
    param (
        [string]$distribution,
        [string]$customName
    )
    Write-Host "Installing $distribution with the name '$customName'..."
    wsl --install -d $distribution --name $customName
    Write-Host "Installation initiated for $customName."
}

# Main script logic
$ubuntuDistros = Get-AvailableDistributions
if ($ubuntuDistros.Count -eq 0) {
    Write-Host "No available Ubuntu distributions found." -ForegroundColor Red
    exit
}

Show-Distributions -distributions $ubuntuDistros
$selectedDistro = Get-UserSelection -distributions $ubuntuDistros

if ($selectedDistro) {
    Write-Host "You selected: $selectedDistro"
    $customName = Read-Host "Enter a custom name for the distribution"
    Install-Distribution -distribution $selectedDistro -customName $customName
}

with: 

# Function to prompt the user for input
function Read-UserInput {
    param (
        [string]$promptMessage
    )
    return Read-Host $promptMessage
}

# Function to create a temporary folder
function New-TemporaryFolder {
    param (
        [string]$folderPath
    )
    if (Test-Path $folderPath) {
        Remove-Item -Recurse -Force $folderPath
    }
    New-Item -ItemType Directory -Path $folderPath | Out-Null
    return $folderPath
}

# Function to clone the GitHub repository
function Get-GitHubRepository {
    param (
        [string]$repositoryUrl,
        [string]$destinationFolder
    )
    Write-Host "Cloning GitHub repository..."
    git clone $repositoryUrl $destinationFolder
}

# Function to validate the presence of the script in the cloned repository
function Test-ScriptPath {
    param (
        [string]$folderPath,
        [string]$scriptPath
    )
    $fullPath = Join-Path -Path $folderPath -ChildPath $scriptPath
    if (!(Test-Path $fullPath)) {
        Write-Host "Error: Script not found at the specified path in the repository." -ForegroundColor Red
        throw "Script not found at the specified path in the repository."
    }
    return $fullPath
}

# Function to convert a Windows path to a WSL-compatible path
function Convert-ToWSLPath {
    param (
        [string]$windowsPath
    )
    return "/mnt/" + ($windowsPath -replace ":", "").Replace("\", "/").Replace(" ", "\ ")
}

# Function to run the Bash script in the specified WSL distribution
function Invoke-ScriptInWSL {
    param (
        [string]$distroName,
        [string]$scriptPath
    )
    Write-Host "Running the script in WSL ($distroName)..."
    wsl -d $distroName bash -c "`"$scriptPath`""
}

# Function to clean up temporary files
function Remove-TemporaryFiles {
    param (
        [string]$folderPath
    )
    Write-Host "Cleaning up temporary files..."
    if (Test-Path $folderPath) {
        Remove-Item -Recurse -Force $folderPath
    }
}

# Main function
function Main {
    # Prompt user for inputs
    $customName = Read-UserInput -promptMessage "Enter the name of the WSL distribution to use (e.g., Ubuntu-20.04)"
    $githubRepo = Read-UserInput -promptMessage "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git):"
    $scriptPathInRepo = Read-UserInput -promptMessage "Enter the relative path of the Bash script inside the repository (e.g., scripts/setup.sh)"
    
    # Create a temporary folder for cloning the repository
    $tempFolder = Create-TemporaryFolder -folderPath "$env:USERPROFILE\wsl-temp-repo"
    
    # Clone the GitHub repository
    Clone-GitHubRepository -repositoryUrl $githubRepo -destinationFolder $tempFolder
    
    # Validate the presence of the script in the cloned repository
    $fullScriptPath = Validate-ScriptPath -folderPath $tempFolder -scriptPath $scriptPathInRepo
    
    # Convert the Windows path of the script to a WSL-compatible path
    $wslScriptPath = Convert-ToWSLPath -windowsPath $fullScriptPath
    
    # Run the Bash script in the specified WSL distribution
    Run-ScriptInWSL -distroName $customName -scriptPath $wslScriptPath
    
    # Clean up the temporary folder
    Clean-UpTemporaryFiles -folderPath $tempFolder
    
    Write-Host "Script execution completed."
}

# Execute the main function
Main

**ChatGPT**: Here’s the merged and streamlined version of the code combining all relevant logic into a unified structure while maintaining modularity and readability:

```powershell
# Function to get the list of available WSL distributions
function Get-AvailableDistributions {
    Write-Host "Fetching available WSL distributions..."
    $distros = wsl --list --online | ForEach-Object { $_.Trim() } | Select-String -Pattern "^Ubuntu"
    $ubuntuDistros = $distros -replace "\s{2,}", " " | ForEach-Object { ($_ -split '\s+')[0] }
    return $ubuntuDistros
}

# Function to display available distributions
function Show-Distributions {
    param (
        [array]$distributions
    )
    Write-Host "Available Ubuntu distributions:"
    for ($i = 0; $i -lt $distributions.Count; $i++) {
        Write-Host "$($i + 1). $($distributions[$i])"
    }
}

# Function to prompt the user for input
function Read-UserInput {
    param (
        [string]$promptMessage
    )
    return Read-Host $promptMessage
}

# Function to create a temporary folder
function Create-TemporaryFolder {
    param (
        [string]$folderPath
    )
    if (Test-Path $folderPath) {
        Remove-Item -Recurse -Force $folderPath
    }
    New-Item -ItemType Directory -Path $folderPath | Out-Null
    return $folderPath
}

# Function to clone the GitHub repository
function Clone-GitHubRepository {
    param (
        [string]$repositoryUrl,
        [string]$destinationFolder
    )
    Write-Host "Cloning GitHub repository..."
    git clone $repositoryUrl $destinationFolder
}

# Function to validate the presence of the script in the cloned repository
function Validate-ScriptPath {
    param (
        [string]$folderPath,
        [string]$scriptPath
    )
    $fullPath = Join-Path -Path $folderPath -ChildPath $scriptPath
    if (!(Test-Path $fullPath)) {
        Write-Host "Error: Script not found at the specified path in the repository." -ForegroundColor Red
        throw "Script not found at the specified path in the repository."
    }
    return $fullPath
}

# Function to convert a Windows path to a WSL-compatible path
function Convert-ToWSLPath {
    param (
        [string]$windowsPath
    )
    return "/mnt/" + ($windowsPath -replace ":", "").Replace("\", "/").Replace(" ", "\ ")
}

# Function to run the Bash script in the specified WSL distribution
function Run-ScriptInWSL {
    param (
        [string]$distroName,
        [string]$scriptPath
    )
    Write-Host "Running the script in WSL ($distroName)..."
    wsl -d $distroName bash -c "`"$scriptPath`""
}

# Function to install the selected distribution with a custom name
function Install-Distribution {
    param (
        [string]$distribution,
        [string]$customName
    )
    Write-Host "Installing $distribution with the name '$customName'..."
    wsl --install -d $distribution --name $customName
    Write-Host "Installation initiated for $customName."
}

# Function to clean up temporary files
function Clean-UpTemporaryFiles {
    param (
        [string]$folderPath
    )
    Write-Host "Cleaning up temporary files..."
    if (Test-Path $folderPath) {
        Remove-Item -Recurse -Force $folderPath
    }
}

# Main function
function Main {
    # Fetch available WSL distributions
    $ubuntuDistros = Get-AvailableDistributions
    if ($ubuntuDistros.Count -eq 0) {
        Write-Host "No available Ubuntu distributions found." -ForegroundColor Red
        return
    }

    # Display available distributions and prompt user to select one
    Show-Distributions -distributions $ubuntuDistros
    $selectedDistro = Read-UserInput -promptMessage "Enter the number of the Ubuntu version you want to install"
    if ($selectedDistro -match "^\d+$" -and $selectedDistro -ge 1 -and $selectedDistro -le $ubuntuDistros.Count) {
        $selectedDistro = $ubuntuDistros[$selectedDistro - 1]
        Write-Host "You selected: $selectedDistro"
    } else {
        Write-Host "Invalid selection. Exiting." -ForegroundColor Red
        return
    }

    # Prompt for custom WSL distribution name
    $customName = Read-UserInput -promptMessage "Enter a custom name for the distribution"
    
    # Install the selected WSL distribution
    Install-Distribution -distribution $selectedDistro -customName $customName

    # Prompt for GitHub repository and script path
    $githubRepo = Read-UserInput -promptMessage "Enter the GitHub repository URL (e.g., https://github.com/username/repository.git):"
    $scriptPathInRepo = Read-UserInput -promptMessage "Enter the relative path of the Bash script inside the repository (e.g., scripts/setup.sh)"
    
    # Create a temporary folder for cloning the repository
    $tempFolder = Create-TemporaryFolder -folderPath "$env:USERPROFILE\wsl-temp-repo"

    # Clone the GitHub repository and validate the script path
    Clone-GitHubRepository -repositoryUrl $githubRepo -destinationFolder $tempFolder
    $fullScriptPath = Validate-ScriptPath -folderPath $tempFolder -scriptPath $scriptPathInRepo

    # Convert the script path to WSL-compatible format
    $wslScriptPath = Convert-ToWSLPath -windowsPath $fullScriptPath

    # Run the script in the specified WSL distribution
    Run-ScriptInWSL -distroName $customName -scriptPath $wslScriptPath

    # Clean up temporary files
    Clean-UpTemporaryFiles -folderPath $tempFolder

    Write-Host "Script execution completed."
}

# Execute the main function
Main
```

### What’s Merged:
1. **WSL Distribution Management**:
   - Fetch, display, and install distributions with custom names.
2. **GitHub Repository Script Execution**:
   - Clone a repository, validate the script path, and run it in WSL.
3. **Unified Input Handling**:
   - `Read-UserInput` simplifies all user prompts.

### Benefits:
- **Single Entry Point**: The `Main` function handles all operations seamlessly.
- **Reusable Functions**: Each task is modular for easy maintenance and testing.
- **Error Handling**: Ensures invalid selections and missing files are handled gracefully.

Let me know if you need further refinements!

