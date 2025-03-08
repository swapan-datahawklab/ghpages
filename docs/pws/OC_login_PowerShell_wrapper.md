**Swapan**: power shell .ps1 wrapper of openshift oc.exe accept id and password oc login function

**ChatGPT**: Here’s an example PowerShell script (`login-wrapper.ps1`) that wraps the OpenShift `oc.exe` tool to accept a user ID and password for logging in:

```powershell
param(
    [string]$OpenShiftUrl, # OpenShift API server URL
    [string]$Username,     # OpenShift username
    [string]$Password      # OpenShift password
)

# Function to check if `oc.exe` is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Error "The 'oc.exe' CLI tool is not installed or not available in PATH."
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ApiUrl,
        [string]$User,
        [string]$Pass
    )

    # Run the oc login command
    $loginResult = & oc.exe login $ApiUrl --username=$User --password=$Pass --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ApiUrl as $User."
    } else {
        Write-Error "Login failed. Please check the URL, username, or password."
        Write-Output $loginResult
        exit 1
    }
}

# Check if oc.exe exists
Check-OcExecutable

# Validate input parameters
if (-not $OpenShiftUrl -or -not $Username -or -not $Password) {
    Write-Error "All parameters (OpenShiftUrl, Username, Password) are required."
    exit 1
}

# Perform login
Login-ToOpenShift -ApiUrl $OpenShiftUrl -User $Username -Pass $Password
```

### How to Use the Script

1. Save the script as `login-wrapper.ps1`.
2. Make sure `oc.exe` is installed and available in your system PATH.
3. Run the script with the required parameters:

   ```powershell
   .\login-wrapper.ps1 -OpenShiftUrl "https://api.openshift.example.com:6443" -Username "your-username" -Password "your-password"
   ```

### Notes

- **TLS Verification**: The script uses `--insecure-skip-tls-verify` for simplicity. In a production environment, ensure proper certificate management.
- **Security**: Avoid exposing sensitive data (like passwords) in plain text. Consider using secure secrets management solutions.
- You can enhance the script to include parameter validation and additional error handling as needed.

**Swapan**: prompt for allowed set of openshift clusters if not passed in at command line

**ChatGPT**: You can modify the script to include a prompt that allows the user to select from a predefined list of allowed OpenShift clusters if the `OpenShiftUrl` parameter is not provided. Here's the updated script:

### Updated Script

```powershell
param(
    [string]$OpenShiftUrl, # OpenShift API server URL (optional)
    [string]$Username,     # OpenShift username
    [string]$Password      # OpenShift password
)

# Predefined list of allowed OpenShift clusters
$AllowedClusters = @{
    "Cluster1" = "https://api.openshift.cluster1.example.com:6443"
    "Cluster2" = "https://api.openshift.cluster2.example.com:6443"
    "Cluster3" = "https://api.openshift.cluster3.example.com:6443"
}

# Function to prompt for cluster selection if URL is not provided
function Select-Cluster {
    Write-Host "Please select an OpenShift cluster from the allowed list:" -ForegroundColor Cyan
    $index = 1
    foreach ($key in $AllowedClusters.Keys) {
        Write-Host "[$index] $key - $($AllowedClusters[$key])"
        $index++
    }

    $selection = Read-Host "Enter the number corresponding to your cluster"
    if ($selection -notmatch '^\d+$' -or [int]$selection -lt 1 -or [int]$selection -gt $AllowedClusters.Count) {
        Write-Error "Invalid selection. Please run the script again and select a valid option."
        exit 1
    }

    $selectedKey = $AllowedClusters.Keys[$selection - 1]
    return $AllowedClusters[$selectedKey]
}

# Function to check if `oc.exe` is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Error "The 'oc.exe' CLI tool is not installed or not available in PATH."
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ApiUrl,
        [string]$User,
        [string]$Pass
    )

    # Run the oc login command
    $loginResult = & oc.exe login $ApiUrl --username=$User --password=$Pass --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ApiUrl as $User."
    } else {
        Write-Error "Login failed. Please check the URL, username, or password."
        Write-Output $loginResult
        exit 1
    }
}

# Check if oc.exe exists
Check-OcExecutable

# Prompt for cluster if OpenShiftUrl is not provided
if (-not $OpenShiftUrl) {
    $OpenShiftUrl = Select-Cluster
}

# Validate username and password
if (-not $Username -or -not $Password) {
    Write-Error "Both Username and Password parameters are required."
    exit 1
}

# Perform login
Login-ToOpenShift -ApiUrl $OpenShiftUrl -User $Username -Pass $Password
```

### Key Features
1. **Allowed Clusters List**:
   - The `$AllowedClusters` variable contains a predefined list of allowed clusters with user-friendly names and URLs.

2. **Cluster Selection**:
   - If `OpenShiftUrl` is not passed as a parameter, the script will prompt the user to select from the allowed clusters.

3. **Validation**:
   - Ensures valid input during cluster selection.

### How to Use
1. Save the script as `login-wrapper.ps1`.
2. Run the script without the `OpenShiftUrl` parameter to trigger the cluster selection prompt:

   ```powershell
   .\login-wrapper.ps1 -Username "your-username" -Password "your-password"
   ```

3. If you want to pass the `OpenShiftUrl` directly, you can still do so:

   ```powershell
   .\login-wrapper.ps1 -OpenShiftUrl "https://api.openshift.cluster1.example.com:6443" -Username "your-username" -Password "your-password"
   ```

This approach ensures flexibility and security, allowing predefined cluster selection while still supporting direct input.

**Swapan**: only pass in cluster1 or cluster2 or cluster3 concatenate generate the rest of the URL

**ChatGPT**: Here's an updated version of the script where the user only needs to pass the cluster name (`Cluster1`, `Cluster2`, or `Cluster3`). The script will automatically concatenate and generate the rest of the URL:

### Updated Script

```powershell
param(
    [string]$ClusterName,  # Cluster name (Cluster1, Cluster2, or Cluster3)
    [string]$Username,     # OpenShift username
    [string]$Password      # OpenShift password
)

# Base URL format for OpenShift clusters
$BaseUrl = "https://api.openshift.{0}.example.com:6443"

# Allowed cluster names
$AllowedClusters = @("Cluster1", "Cluster2", "Cluster3")

# Function to validate and generate the URL
function Get-ClusterUrl {
    param (
        [string]$ClusterName
    )

    if ($ClusterName -notin $AllowedClusters) {
        Write-Error "Invalid cluster name. Allowed values are: $($AllowedClusters -join ', ')"
        exit 1
    }

    return $BaseUrl -f $ClusterName.ToLower()
}

# Function to check if `oc.exe` is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Error "The 'oc.exe' CLI tool is not installed or not available in PATH."
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ApiUrl,
        [string]$User,
        [string]$Pass
    )

    # Run the oc login command
    $loginResult = & oc.exe login $ApiUrl --username=$User --password=$Pass --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ApiUrl as $User."
    } else {
        Write-Error "Login failed. Please check the URL, username, or password."
        Write-Output $loginResult
        exit 1
    }
}

# Check if oc.exe exists
Check-OcExecutable

# Validate input parameters
if (-not $ClusterName -or -not $Username -or -not $Password) {
    Write-Error "All parameters (ClusterName, Username, Password) are required."
    exit 1
}

# Generate the URL for the selected cluster
$OpenShiftUrl = Get-ClusterUrl -ClusterName $ClusterName

# Perform login
Login-ToOpenShift -ApiUrl $OpenShiftUrl -User $Username -Pass $Password
```

### How to Use
1. Save the script as `login-wrapper.ps1`.
2. Run the script with the required parameters:

   ```powershell
   .\login-wrapper.ps1 -ClusterName "Cluster1" -Username "your-username" -Password "your-password"
   ```

3. The script will automatically generate the URL based on the cluster name and log you in.

### Key Features
1. **Simplified Input**:
   - The user only needs to specify the cluster name (`Cluster1`, `Cluster2`, or `Cluster3`).

2. **Automatic URL Generation**:
   - The `$BaseUrl` format string is used to construct the full OpenShift API URL dynamically.

3. **Validation**:
   - Ensures that the cluster name is one of the allowed values.

4. **Security**:
   - Supports secure input and handles errors gracefully.

This approach reduces the risk of errors in typing the full URL and simplifies the user experience.

**Swapan**: prompt to choose a cluster if clustername is not passed in at the command line

**ChatGPT**: Here's the updated script that prompts the user to choose a cluster from a predefined list if `ClusterName` is not passed in at the command line:

### Updated Script with Cluster Prompt

```powershell
param(
    [string]$ClusterName,  # Cluster name (Cluster1, Cluster2, or Cluster3)
    [string]$Username,     # OpenShift username
    [string]$Password      # OpenShift password
)

# Base URL format for OpenShift clusters
$BaseUrl = "https://api.openshift.{0}.example.com:6443"

# Allowed cluster names
$AllowedClusters = @("Cluster1", "Cluster2", "Cluster3")

# Function to prompt for cluster selection
function Select-Cluster {
    Write-Host "Please select a cluster from the following options:" -ForegroundColor Cyan
    for ($i = 0; $i -lt $AllowedClusters.Count; $i++) {
        Write-Host "[$($i + 1)] $($AllowedClusters[$i])"
    }

    $selection = Read-Host "Enter the number corresponding to your cluster"
    if ($selection -notmatch '^\d+$' -or [int]$selection -lt 1 -or [int]$selection -gt $AllowedClusters.Count) {
        Write-Error "Invalid selection. Please run the script again and select a valid option."
        exit 1
    }

    return $AllowedClusters[[int]$selection - 1]
}

# Function to validate and generate the URL
function Get-ClusterUrl {
    param (
        [string]$ClusterName
    )

    if ($ClusterName -notin $AllowedClusters) {
        Write-Error "Invalid cluster name. Allowed values are: $($AllowedClusters -join ', ')"
        exit 1
    }

    return $BaseUrl -f $ClusterName.ToLower()
}

# Function to check if `oc.exe` is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Error "The 'oc.exe' CLI tool is not installed or not available in PATH."
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ApiUrl,
        [string]$User,
        [string]$Pass
    )

    # Run the oc login command
    $loginResult = & oc.exe login $ApiUrl --username=$User --password=$Pass --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ApiUrl as $User."
    } else {
        Write-Error "Login failed. Please check the URL, username, or password."
        Write-Output $loginResult
        exit 1
    }
}

# Check if oc.exe exists
Check-OcExecutable

# Prompt for cluster name if not provided
if (-not $ClusterName) {
    $ClusterName = Select-Cluster
}

# Validate username and password
if (-not $Username -or -not $Password) {
    Write-Error "Both Username and Password parameters are required."
    exit 1
}

# Generate the URL for the selected cluster
$OpenShiftUrl = Get-ClusterUrl -ClusterName $ClusterName

# Perform login
Login-ToOpenShift -ApiUrl $OpenShiftUrl -User $Username -Pass $Password
```

### How It Works
1. **Cluster Prompt**:
   - If `ClusterName` is not provided, the script calls the `Select-Cluster` function, prompting the user to select a cluster from the predefined list.

2. **Automatic URL Generation**:
   - Once a cluster is selected, the `Get-ClusterUrl` function constructs the full API URL.

3. **Validation**:
   - Ensures valid cluster selection and checks other required inputs.

### How to Use the Script
1. Run the script without passing a `ClusterName` to trigger the prompt:

   ```powershell
   .\login-wrapper.ps1 -Username "your-username" -Password "your-password"
   ```

   You'll see a prompt like this:

   ```
   Please select a cluster from the following options:
   [1] Cluster1
   [2] Cluster2
   [3] Cluster3
   Enter the number corresponding to your cluster:
   ```

   Enter the number corresponding to the desired cluster.

2. If you know the cluster name, you can still pass it directly:

   ```powershell
   .\login-wrapper.ps1 -ClusterName "Cluster1" -Username "your-username" -Password "your-password"
   ```

This modification provides a better user experience by letting the user select a cluster interactively if they haven't provided it upfront.

**Swapan**: prompt for the password if not specified at command line

**ChatGPT**: Here’s the updated script that prompts for the password if it’s not specified at the command line:

### Updated Script

```powershell
param(
    [string]$ClusterName,  # Cluster name (Cluster1, Cluster2, or Cluster3)
    [string]$Username,     # OpenShift username
    [string]$Password      # OpenShift password (optional)
)

# Base URL format for OpenShift clusters
$BaseUrl = "https://api.openshift.{0}.example.com:6443"

# Allowed cluster names
$AllowedClusters = @("Cluster1", "Cluster2", "Cluster3")

# Function to prompt for cluster selection
function Select-Cluster {
    Write-Host "Please select a cluster from the following options:" -ForegroundColor Cyan
    for ($i = 0; $i -lt $AllowedClusters.Count; $i++) {
        Write-Host "[$($i + 1)] $($AllowedClusters[$i])"
    }

    $selection = Read-Host "Enter the number corresponding to your cluster"
    if ($selection -notmatch '^\d+$' -or [int]$selection -lt 1 -or [int]$selection -gt $AllowedClusters.Count) {
        Write-Error "Invalid selection. Please run the script again and select a valid option."
        exit 1
    }

    return $AllowedClusters[[int]$selection - 1]
}

# Function to validate and generate the URL
function Get-ClusterUrl {
    param (
        [string]$ClusterName
    )

    if ($ClusterName -notin $AllowedClusters) {
        Write-Error "Invalid cluster name. Allowed values are: $($AllowedClusters -join ', ')"
        exit 1
    }

    return $BaseUrl -f $ClusterName.ToLower()
}

# Function to check if `oc.exe` is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Error "The 'oc.exe' CLI tool is not installed or not available in PATH."
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ApiUrl,
        [string]$User,
        [string]$Pass
    )

    # Run the oc login command
    $loginResult = & oc.exe login $ApiUrl --username=$User --password=$Pass --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ApiUrl as $User."
    } else {
        Write-Error "Login failed. Please check the URL, username, or password."
        Write-Output $loginResult
        exit 1
    }
}

# Check if oc.exe exists
Check-OcExecutable

# Prompt for cluster name if not provided
if (-not $ClusterName) {
    $ClusterName = Select-Cluster
}

# Prompt for password if not provided
if (-not $Password) {
    $Password = Read-Host "Enter your OpenShift password" -AsSecureString
    $Password = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($Password))
}

# Validate username
if (-not $Username) {
    Write-Error "The Username parameter is required."
    exit 1
}

# Generate the URL for the selected cluster
$OpenShiftUrl = Get-ClusterUrl -ClusterName $ClusterName

# Perform login
Login-ToOpenShift -ApiUrl $OpenShiftUrl -User $Username -Pass $Password
```

### What’s New
1. **Password Prompt**:
   - If `Password` is not provided as a parameter, the script prompts the user to enter it securely using `Read-Host` with the `-AsSecureString` option.
   - The password is securely handled, converted to plaintext for use with `oc.exe`, and not displayed on the screen.

2. **Cluster Prompt**:
   - If `ClusterName` is not provided, the user is prompted to select a cluster from a predefined list.

3. **Validation**:
   - Ensures all required parameters are provided or prompted for.

### How to Use
1. Run the script without specifying the password to trigger the secure password prompt:

   ```powershell
   .\login-wrapper.ps1 -ClusterName "Cluster1" -Username "your-username"
   ```

2. If you want to provide all parameters directly:

   ```powershell
   .\login-wrapper.ps1 -ClusterName "Cluster1" -Username "your-username" -Password "your-password"
   ```

This updated script ensures both flexibility and security by securely prompting for the password if it’s not provided on the command line.

**Swapan**: simple UI for the power shell script?

**ChatGPT**: You can create a simple graphical user interface (GUI) for your PowerShell script using Windows Presentation Foundation (WPF). Here’s how to make a simple UI for your OpenShift login script.

### Example: Simple PowerShell GUI

```powershell
# Import necessary assembly for WPF
Add-Type -AssemblyName PresentationFramework

# Create the GUI window
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")

# Create the form
$form = New-Object System.Windows.Forms.Form
$form.Text = "OpenShift Login"
$form.Size = New-Object System.Drawing.Size(400, 250)
$form.StartPosition = "CenterScreen"

# Create labels and input fields
$labelCluster = New-Object System.Windows.Forms.Label
$labelCluster.Text = "Cluster Name:"
$labelCluster.Location = New-Object System.Drawing.Point(10, 20)
$form.Controls.Add($labelCluster)

$comboCluster = New-Object System.Windows.Forms.ComboBox
$comboCluster.Items.AddRange(@("Cluster1", "Cluster2", "Cluster3"))
$comboCluster.Location = New-Object System.Drawing.Point(120, 18)
$comboCluster.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboCluster)

$labelUsername = New-Object System.Windows.Forms.Label
$labelUsername.Text = "Username:"
$labelUsername.Location = New-Object System.Drawing.Point(10, 60)
$form.Controls.Add($labelUsername)

$textUsername = New-Object System.Windows.Forms.TextBox
$textUsername.Location = New-Object System.Drawing.Point(120, 58)
$textUsername.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($textUsername)

$labelPassword = New-Object System.Windows.Forms.Label
$labelPassword.Text = "Password:"
$labelPassword.Location = New-Object System.Drawing.Point(10, 100)
$form.Controls.Add($labelPassword)

$textPassword = New-Object System.Windows.Forms.TextBox
$textPassword.Location = New-Object System.Drawing.Point(120, 98)
$textPassword.Size = New-Object System.Drawing.Size(200, 20)
$textPassword.UseSystemPasswordChar = $true
$form.Controls.Add($textPassword)

# Create buttons
$btnLogin = New-Object System.Windows.Forms.Button
$btnLogin.Text = "Login"
$btnLogin.Location = New-Object System.Drawing.Point(120, 150)
$form.Controls.Add($btnLogin)

$btnCancel = New-Object System.Windows.Forms.Button
$btnCancel.Text = "Cancel"
$btnCancel.Location = New-Object System.Drawing.Point(220, 150)
$form.Controls.Add($btnCancel)

# Define button actions
$btnLogin.Add_Click({
    $ClusterName = $comboCluster.Text
    $Username = $textUsername.Text
    $Password = $textPassword.Text

    if (-not $ClusterName -or -not $Username -or -not $Password) {
        [System.Windows.Forms.MessageBox]::Show("All fields are required!", "Error", "OK", "Error")
    } else {
        $BaseUrl = "https://api.openshift.{0}.example.com:6443"
        $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

        # Call oc.exe with the collected parameters
        $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

        if ($loginResult -match "Login successful") {
            [System.Windows.Forms.MessageBox]::Show("Login successful!", "Success", "OK", "Information")
        } else {
            [System.Windows.Forms.MessageBox]::Show("Login failed!`n`n$loginResult", "Error", "OK", "Error")
        }
    }
})

$btnCancel.Add_Click({
    $form.Close()
})

# Show the form
[void] $form.ShowDialog()
```

### Key Features
1. **Dropdown for Cluster Selection**:
   - Users can select `Cluster1`, `Cluster2`, or `Cluster3` from a dropdown.

2. **Input Fields for Username and Password**:
   - The username is a simple text field, and the password field hides input for security.

3. **Login Button**:
   - Logs the user into the selected OpenShift cluster by invoking the `oc.exe login` command.

4. **Cancel Button**:
   - Closes the GUI window without doing anything.

### How to Run
1. Copy the script into a `.ps1` file (e.g., `OpenShiftLoginGUI.ps1`).
2. Run the script in PowerShell:
   ```powershell
   .\OpenShiftLoginGUI.ps1
   ```
3. The GUI will appear, allowing you to input the required fields and perform the login.

This simple UI improves user experience by removing the need to use the command line directly, making the login process more user-friendly.

**Swapan**: add check to see if oc.exe is in path

**ChatGPT**: Here's the updated script with a check to ensure that `oc.exe` is available in the system's PATH. If it's not found, the user will be alerted, and the script will not proceed.

### Updated Script with `oc.exe` PATH Check

```powershell
# Import necessary assembly for WPF
Add-Type -AssemblyName PresentationFramework

# Create the GUI window
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")

# Create the form
$form = New-Object System.Windows.Forms.Form
$form.Text = "OpenShift Login"
$form.Size = New-Object System.Drawing.Size(400, 250)
$form.StartPosition = "CenterScreen"

# Check if oc.exe exists in PATH
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        [System.Windows.Forms.MessageBox]::Show(
            "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again.",
            "Error",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        $form.Close()
        exit 1
    }
}

# Run the check for oc.exe
Check-OcExecutable

# Create labels and input fields
$labelCluster = New-Object System.Windows.Forms.Label
$labelCluster.Text = "Cluster Name:"
$labelCluster.Location = New-Object System.Drawing.Point(10, 20)
$form.Controls.Add($labelCluster)

$comboCluster = New-Object System.Windows.Forms.ComboBox
$comboCluster.Items.AddRange(@("Cluster1", "Cluster2", "Cluster3"))
$comboCluster.Location = New-Object System.Drawing.Point(120, 18)
$comboCluster.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboCluster)

$labelUsername = New-Object System.Windows.Forms.Label
$labelUsername.Text = "Username:"
$labelUsername.Location = New-Object System.Drawing.Point(10, 60)
$form.Controls.Add($labelUsername)

$textUsername = New-Object System.Windows.Forms.TextBox
$textUsername.Location = New-Object System.Drawing.Point(120, 58)
$textUsername.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($textUsername)

$labelPassword = New-Object System.Windows.Forms.Label
$labelPassword.Text = "Password:"
$labelPassword.Location = New-Object System.Drawing.Point(10, 100)
$form.Controls.Add($labelPassword)

$textPassword = New-Object System.Windows.Forms.TextBox
$textPassword.Location = New-Object System.Drawing.Point(120, 98)
$textPassword.Size = New-Object System.Drawing.Size(200, 20)
$textPassword.UseSystemPasswordChar = $true
$form.Controls.Add($textPassword)

# Create buttons
$btnLogin = New-Object System.Windows.Forms.Button
$btnLogin.Text = "Login"
$btnLogin.Location = New-Object System.Drawing.Point(120, 150)
$form.Controls.Add($btnLogin)

$btnCancel = New-Object System.Windows.Forms.Button
$btnCancel.Text = "Cancel"
$btnCancel.Location = New-Object System.Drawing.Point(220, 150)
$form.Controls.Add($btnCancel)

# Define button actions
$btnLogin.Add_Click({
    $ClusterName = $comboCluster.Text
    $Username = $textUsername.Text
    $Password = $textPassword.Text

    if (-not $ClusterName -or -not $Username -or -not $Password) {
        [System.Windows.Forms.MessageBox]::Show("All fields are required!", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
    } else {
        $BaseUrl = "https://api.openshift.{0}.example.com:6443"
        $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

        # Call oc.exe with the collected parameters
        $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

        if ($loginResult -match "Login successful") {
            [System.Windows.Forms.MessageBox]::Show("Login successful!", "Success", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Information)
        } else {
            [System.Windows.Forms.MessageBox]::Show("Login failed!`n`n$loginResult", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
        }
    }
})

$btnCancel.Add_Click({
    $form.Close()
})

# Show the form
[void] $form.ShowDialog()
```

### Key Additions
1. **Check for `oc.exe`**:
   - A `Check-OcExecutable` function verifies whether `oc.exe` exists in the system's PATH.
   - If not found, an error message is displayed in a message box, and the script exits.

2. **User-Friendly Error**:
   - If `oc.exe` is not available, the script alerts the user with a message box and does not proceed further.

### How to Use
1. Save the script to a `.ps1` file (e.g., `OpenShiftLoginGUI.ps1`).
2. Ensure `oc.exe` is installed and added to your system PATH.
3. Run the script in PowerShell:
   ```powershell
   .\OpenShiftLoginGUI.ps1
   ```
4. If `oc.exe` is not found, you’ll see an error message like this:
   ```
   The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again.
   ```

This ensures the script only proceeds if the necessary tool is available, providing a better user experience.

**Swapan**: after login execute oc get projects and display in table allow to switch to a project from dropdown or all projects

**ChatGPT**: Here’s an updated version of the PowerShell script. After a successful login, it retrieves the list of OpenShift projects (`oc get projects`) and displays them in a table. The user can then select a project from a dropdown menu or switch to all projects.

### Full Script with Project Selection

```powershell
# Import necessary assembly for WPF
Add-Type -AssemblyName PresentationFramework

# Create the GUI window
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")

# Check if oc.exe exists in PATH
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        [System.Windows.Forms.MessageBox]::Show(
            "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again.",
            "Error",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        exit 1
    }
}

# Run the check for oc.exe
Check-OcExecutable

# Function to get OpenShift projects
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        [System.Windows.Forms.MessageBox]::Show(
            "Failed to retrieve projects. Make sure you are logged in and have the necessary permissions.",
            "Error",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        return @()
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Function to switch project
function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    if ($ProjectName -eq "All Projects") {
        [System.Windows.Forms.MessageBox]::Show(
            "You are now viewing all projects.",
            "Info",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Information
        )
    } else {
        $switchOutput = & oc.exe project $ProjectName 2>&1
        if ($switchOutput -match "Now using project") {
            [System.Windows.Forms.MessageBox]::Show(
                "Switched to project '$ProjectName'.",
                "Success",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Information
            )
        } else {
            [System.Windows.Forms.MessageBox]::Show(
                "Failed to switch to project '$ProjectName'.",
                "Error",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Error
            )
        }
    }
}

# Create the form
$form = New-Object System.Windows.Forms.Form
$form.Text = "OpenShift Login and Projects"
$form.Size = New-Object System.Drawing.Size(500, 350)
$form.StartPosition = "CenterScreen"

# Create labels and input fields
$labelCluster = New-Object System.Windows.Forms.Label
$labelCluster.Text = "Cluster Name:"
$labelCluster.Location = New-Object System.Drawing.Point(10, 20)
$form.Controls.Add($labelCluster)

$comboCluster = New-Object System.Windows.Forms.ComboBox
$comboCluster.Items.AddRange(@("Cluster1", "Cluster2", "Cluster3"))
$comboCluster.Location = New-Object System.Drawing.Point(120, 18)
$comboCluster.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboCluster)

$labelUsername = New-Object System.Windows.Forms.Label
$labelUsername.Text = "Username:"
$labelUsername.Location = New-Object System.Drawing.Point(10, 60)
$form.Controls.Add($labelUsername)

$textUsername = New-Object System.Windows.Forms.TextBox
$textUsername.Location = New-Object System.Drawing.Point(120, 58)
$textUsername.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($textUsername)

$labelPassword = New-Object System.Windows.Forms.Label
$labelPassword.Text = "Password:"
$labelPassword.Location = New-Object System.Drawing.Point(10, 100)
$form.Controls.Add($labelPassword)

$textPassword = New-Object System.Windows.Forms.TextBox
$textPassword.Location = New-Object System.Drawing.Point(120, 98)
$textPassword.Size = New-Object System.Drawing.Size(200, 20)
$textPassword.UseSystemPasswordChar = $true
$form.Controls.Add($textPassword)

# Create buttons
$btnLogin = New-Object System.Windows.Forms.Button
$btnLogin.Text = "Login"
$btnLogin.Location = New-Object System.Drawing.Point(120, 150)
$form.Controls.Add($btnLogin)

$btnCancel = New-Object System.Windows.Forms.Button
$btnCancel.Text = "Cancel"
$btnCancel.Location = New-Object System.Drawing.Point(220, 150)
$form.Controls.Add($btnCancel)

$labelProjects = New-Object System.Windows.Forms.Label
$labelProjects.Text = "Projects:"
$labelProjects.Location = New-Object System.Drawing.Point(10, 200)
$form.Controls.Add($labelProjects)

$comboProjects = New-Object System.Windows.Forms.ComboBox
$comboProjects.Location = New-Object System.Drawing.Point(120, 198)
$comboProjects.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboProjects)

$btnSwitchProject = New-Object System.Windows.Forms.Button
$btnSwitchProject.Text = "Switch Project"
$btnSwitchProject.Location = New-Object System.Drawing.Point(120, 240)
$form.Controls.Add($btnSwitchProject)

# Define button actions
$btnLogin.Add_Click({
    $ClusterName = $comboCluster.Text
    $Username = $textUsername.Text
    $Password = $textPassword.Text

    if (-not $ClusterName -or -not $Username -or -not $Password) {
        [System.Windows.Forms.MessageBox]::Show("All fields are required!", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
    } else {
        $BaseUrl = "https://api.openshift.{0}.example.com:6443"
        $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

        # Call oc.exe with the collected parameters
        $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

        if ($loginResult -match "Login successful") {
            [System.Windows.Forms.MessageBox]::Show("Login successful!", "Success", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Information)

            # Get the list of projects
            $projects = Get-OpenShiftProjects
            $comboProjects.Items.Clear()
            $comboProjects.Items.Add("All Projects")
            $projects | ForEach-Object { $comboProjects.Items.Add($_) }
            $comboProjects.SelectedIndex = 0
        } else {
            [System.Windows.Forms.MessageBox]::Show("Login failed!`n`n$loginResult", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
        }
    }
})

$btnSwitchProject.Add_Click({
    $selectedProject = $comboProjects.Text
    Switch-OpenShiftProject -ProjectName $selectedProject
})

$btnCancel.Add_Click({
    $form.Close()
})

# Show the form
[void] $form.ShowDialog()
```

### Key Features:
1. **Login to OpenShift**:
   - Allows the user to log in with a username, password, and cluster name.
   
2. **Retrieve Projects**:
   - After login, retrieves the list of OpenShift projects using `oc get projects`.

3. **Dropdown for Projects**:
   - Displays all projects in a dropdown list, along with an option to view "All Projects."

4. **Switch Projects**:
   - Allows the user to switch to a selected project from the dropdown.

5. **Error Handling**:
   - Displays appropriate error messages if login or project retrieval fails.

### How to Use:
1. Save this script to a `.ps1` file, such as `OpenShiftLoginAndProjectsGUI.ps1`.
2. Run the script in PowerShell:
   ```powershell
   .\OpenShiftLoginAndProjectsGUI.ps1
   ```
3. Login, view projects, and switch between them as needed.

This script provides a simple yet effective GUI for managing OpenShift clusters and projects.

**Swapan**: how to run by double clicking on script

**ChatGPT**: To run a PowerShell script by double-clicking, you need to associate `.ps1` files with PowerShell or create a shortcut that directly executes the script. Here's how to do it:

---

### Method 1: Create a Shortcut
1. **Save the Script**:
   Save your script with the `.ps1` extension (e.g., `OpenShiftLoginGUI.ps1`).

2. **Create a Shortcut**:
   - Right-click on the desktop or in a folder.
   - Select **New > Shortcut**.
   - In the location field, enter the following command:
     ```plaintext
     powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Path\To\OpenShiftLoginGUI.ps1"
     ```
     Replace `C:\Path\To\OpenShiftLoginGUI.ps1` with the full path to your script.

3. **Name the Shortcut**:
   Give the shortcut a meaningful name (e.g., "OpenShift Login GUI").

4. **Double-Click the Shortcut**:
   Double-clicking the shortcut will run your script.

---

### Method 2: Associate `.ps1` Files with PowerShell
1. **Change Default Program for `.ps1` Files**:
   - Right-click on any `.ps1` file and select **Open with > Choose another app**.
   - Select **PowerShell** from the list. If it's not listed, click **More apps > Look for another app on this PC**, navigate to:
     ```
     C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
     ```
   - Check **Always use this app to open .ps1 files**, then click **OK**.

2. **Enable Script Execution**:
   Ensure the script execution policy allows the script to run:
   - Open PowerShell as an Administrator and run:
     ```powershell
     Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Bypass
     ```

3. **Double-Click the Script**:
   Double-clicking the `.ps1` file will now execute it in PowerShell.

---

### Method 3: Use a Batch File
1. **Create a Batch File**:
   - Open Notepad and enter the following code:
     ```batch
     @echo off
     powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Path\To\OpenShiftLoginGUI.ps1"
     pause
     ```
     Replace `C:\Path\To\OpenShiftLoginGUI.ps1` with the full path to your script.

2. **Save the Batch File**:
   Save the file with a `.bat` extension (e.g., `OpenShiftLoginGUI.bat`).

3. **Double-Click the Batch File**:
   Double-clicking the batch file will run your PowerShell script.

---

### Notes:
- **Script Permissions**: Ensure the script has the necessary permissions and is not blocked. Right-click the `.ps1` file, select **Properties**, and click **Unblock** if the option appears.
- **Execution Policy**: If the execution policy is restrictive, you may encounter errors. The `-ExecutionPolicy Bypass` option in the methods above temporarily overrides the restriction for the script.

Using any of these methods, you can execute your PowerShell script with a simple double-click.

**Swapan**: bat file to create shorcut and run dynamically every time

**ChatGPT**: Here’s a `.bat` file that dynamically creates a shortcut to your PowerShell script and executes it each time you run the `.bat`. This ensures the shortcut is recreated and up-to-date every time.

### Batch Script to Create and Run Shortcut

Save the following script as a `.bat` file (e.g., `RunOpenShiftScript.bat`):

```batch
@echo off
:: Set variables for the paths
set SCRIPT_PATH="C:\Path\To\Your\OpenShiftLoginGUI.ps1"
set SHORTCUT_NAME="OpenShift Login GUI"
set SHORTCUT_PATH="%TEMP%\%SHORTCUT_NAME%.lnk"

:: Check if PowerShell script exists
if not exist %SCRIPT_PATH% (
    echo PowerShell script not found at %SCRIPT_PATH%.
    pause
    exit /b
)

:: Create the PowerShell shortcut dynamically
powershell.exe -Command ^
    $ws = New-Object -ComObject WScript.Shell; ^
    $shortcut = $ws.CreateShortcut("%SHORTCUT_PATH%"); ^
    $shortcut.TargetPath = "powershell.exe"; ^
    $shortcut.Arguments = "-NoProfile -ExecutionPolicy Bypass -File %SCRIPT_PATH%"; ^
    $shortcut.WorkingDirectory = "%~dp0"; ^
    $shortcut.WindowStyle = 1; ^
    $shortcut.Save()

:: Run the shortcut
start "" "%SHORTCUT_PATH%"

:: Clean up (optional: remove the shortcut after use)
:: timeout /t 1 >nul
:: del "%SHORTCUT_PATH%"
```

---

### How It Works
1. **Dynamic Shortcut Creation**:
   - The script uses PowerShell's `WScript.Shell` COM object to dynamically create a shortcut in the `%TEMP%` directory.
   - The shortcut points to `powershell.exe` and includes the `-NoProfile -ExecutionPolicy Bypass -File` arguments to execute the `.ps1` script.

2. **Temporary Shortcut**:
   - The shortcut is created in the `%TEMP%` folder and run immediately.

3. **Run the PowerShell Script**:
   - The batch file ensures the script is executed with the required permissions and settings.

4. **Optional Cleanup**:
   - The shortcut can be deleted after execution by uncommenting the cleanup section (`del "%SHORTCUT_PATH%"`).

---

### How to Use
1. **Edit the Script Path**:
   - Replace `C:\Path\To\Your\OpenShiftLoginGUI.ps1` with the full path to your PowerShell script.

2. **Save the Batch File**:
   - Save the `.bat` file (e.g., `RunOpenShiftScript.bat`) in a convenient location.

3. **Run the Batch File**:
   - Double-click the `.bat` file to create and run the shortcut dynamically.

---

### Advantages
- **Dynamic**: The shortcut is recreated each time, ensuring it's always up-to-date.
- **No Manual Shortcut Management**: You don't need to manually create or maintain the shortcut.
- **Portable**: The batch file can be moved to another system with minimal changes (just update the PowerShell script path).

This method ensures a clean and efficient way to manage and execute your PowerShell script.

**Swapan**: can i do this powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Path\To\OpenShiftLoginGUI.ps1" with relative path to .ps1 script?

**ChatGPT**: Yes, you can use a relative path to the `.ps1` script when running it with `powershell.exe`. To make it relative to the location of the `.bat` file, you can use `%~dp0`, which represents the directory of the batch file. Here's how you can modify the command to use a relative path:

### Example with Relative Path
Assume the `.ps1` script (`OpenShiftLoginGUI.ps1`) is in the same directory as the `.bat` file. The `.bat` file would look like this:

```batch
@echo off
:: Run the PowerShell script with a relative path
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%~dp0OpenShiftLoginGUI.ps1"
```

### Explanation:
1. **`%~dp0`**:
   - This expands to the full path of the directory containing the `.bat` file, including the trailing backslash (`\`).
   - For example, if your `.bat` file is in `C:\Scripts`, `%~dp0` will expand to `C:\Scripts\`.

2. **Relative Path**:
   - Appending `OpenShiftLoginGUI.ps1` to `%~dp0` creates the full path to the `.ps1` script, assuming it's in the same directory as the `.bat`.

### How to Use:
1. Place the `.ps1` script (e.g., `OpenShiftLoginGUI.ps1`) and the `.bat` file in the same directory.
2. Double-click the `.bat` file to execute the script.

---

### If the `.ps1` Script is in a Subdirectory
If the `.ps1` script is in a subdirectory relative to the `.bat` file, modify the path accordingly. For example:

```batch
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%~dp0subfolder\OpenShiftLoginGUI.ps1"
```

---

### Additional Tip: Error Handling
To add error handling if the `.ps1` script is not found, you can include a check like this:

```batch
@echo off
:: Check if the PowerShell script exists
if not exist "%~dp0OpenShiftLoginGUI.ps1" (
    echo PowerShell script not found: %~dp0OpenShiftLoginGUI.ps1
    pause
    exit /b
)

:: Run the PowerShell script
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%~dp0OpenShiftLoginGUI.ps1"
```

This ensures the script only runs if the `.ps1` file is present.

**Swapan**: on run below if run in interactive mode otherwise also allow login to openshift and namespace info at command line 

# Import necessary assembly for WPF
Add-Type -AssemblyName PresentationFramework

# Create the GUI window
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")

# Check if oc.exe exists in PATH
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        [System.Windows.Forms.MessageBox]::Show(
            "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again.",
            "Error",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        exit 1
    }
}

# Run the check for oc.exe
Check-OcExecutable

# Function to get OpenShift projects
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        [System.Windows.Forms.MessageBox]::Show(
            "Failed to retrieve projects. Make sure you are logged in and have the necessary permissions.",
            "Error",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        return @()
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Function to switch project
function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    if ($ProjectName -eq "All Projects") {
        [System.Windows.Forms.MessageBox]::Show(
            "You are now viewing all projects.",
            "Info",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Information
        )
    } else {
        $switchOutput = & oc.exe project $ProjectName 2>&1
        if ($switchOutput -match "Now using project") {
            [System.Windows.Forms.MessageBox]::Show(
                "Switched to project '$ProjectName'.",
                "Success",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Information
            )
        } else {
            [System.Windows.Forms.MessageBox]::Show(
                "Failed to switch to project '$ProjectName'.",
                "Error",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Error
            )
        }
    }
}

# Create the form
$form = New-Object System.Windows.Forms.Form
$form.Text = "OpenShift Login and Projects"
$form.Size = New-Object System.Drawing.Size(500, 350)
$form.StartPosition = "CenterScreen"

# Create labels and input fields
$labelCluster = New-Object System.Windows.Forms.Label
$labelCluster.Text = "Cluster Name:"
$labelCluster.Location = New-Object System.Drawing.Point(10, 20)
$form.Controls.Add($labelCluster)

$comboCluster = New-Object System.Windows.Forms.ComboBox
$comboCluster.Items.AddRange(@("Cluster1", "Cluster2", "Cluster3"))
$comboCluster.Location = New-Object System.Drawing.Point(120, 18)
$comboCluster.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboCluster)

$labelUsername = New-Object System.Windows.Forms.Label
$labelUsername.Text = "Username:"
$labelUsername.Location = New-Object System.Drawing.Point(10, 60)
$form.Controls.Add($labelUsername)

$textUsername = New-Object System.Windows.Forms.TextBox
$textUsername.Location = New-Object System.Drawing.Point(120, 58)
$textUsername.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($textUsername)

$labelPassword = New-Object System.Windows.Forms.Label
$labelPassword.Text = "Password:"
$labelPassword.Location = New-Object System.Drawing.Point(10, 100)
$form.Controls.Add($labelPassword)

$textPassword = New-Object System.Windows.Forms.TextBox
$textPassword.Location = New-Object System.Drawing.Point(120, 98)
$textPassword.Size = New-Object System.Drawing.Size(200, 20)
$textPassword.UseSystemPasswordChar = $true
$form.Controls.Add($textPassword)

# Create buttons
$btnLogin = New-Object System.Windows.Forms.Button
$btnLogin.Text = "Login"
$btnLogin.Location = New-Object System.Drawing.Point(120, 150)
$form.Controls.Add($btnLogin)

$btnCancel = New-Object System.Windows.Forms.Button
$btnCancel.Text = "Cancel"
$btnCancel.Location = New-Object System.Drawing.Point(220, 150)
$form.Controls.Add($btnCancel)

$labelProjects = New-Object System.Windows.Forms.Label
$labelProjects.Text = "Projects:"
$labelProjects.Location = New-Object System.Drawing.Point(10, 200)
$form.Controls.Add($labelProjects)

$comboProjects = New-Object System.Windows.Forms.ComboBox
$comboProjects.Location = New-Object System.Drawing.Point(120, 198)
$comboProjects.Size = New-Object System.Drawing.Size(200, 20)
$form.Controls.Add($comboProjects)

$btnSwitchProject = New-Object System.Windows.Forms.Button
$btnSwitchProject.Text = "Switch Project"
$btnSwitchProject.Location = New-Object System.Drawing.Point(120, 240)
$form.Controls.Add($btnSwitchProject)

# Define button actions
$btnLogin.Add_Click({
    $ClusterName = $comboCluster.Text
    $Username = $textUsername.Text
    $Password = $textPassword.Text

    if (-not $ClusterName -or -not $Username -or -not $Password) {
        [System.Windows.Forms.MessageBox]::Show("All fields are required!", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
    } else {
        $BaseUrl = "https://api.openshift.{0}.example.com:6443"
        $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

        # Call oc.exe with the collected parameters
        $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

        if ($loginResult -match "Login successful") {
            [System.Windows.Forms.MessageBox]::Show("Login successful!", "Success", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Information)

            # Get the list of projects
            $projects = Get-OpenShiftProjects
            $comboProjects.Items.Clear()
            $comboProjects.Items.Add("All Projects")
            $projects | ForEach-Object { $comboProjects.Items.Add($_) }
            $comboProjects.SelectedIndex = 0
        } else {
            [System.Windows.Forms.MessageBox]::Show("Login failed!`n`n$loginResult", "Error", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Error)
        }
    }
})

$btnSwitchProject.Add_Click({
    $selectedProject = $comboProjects.Text
    Switch-OpenShiftProject -ProjectName $selectedProject
})

$btnCancel.Add_Click({
    $form.Close()
})

# Show the form
[void] $form.ShowDialog()

**ChatGPT**: To make the script detect whether it should run in interactive mode (GUI) or as a command-line tool, you can check whether any arguments are passed to the script. If arguments are provided, run the login process and namespace operations in the command-line mode. Otherwise, launch the GUI.

Here's an updated version of the script:

### Updated Script

```powershell
# Check if oc.exe exists in PATH
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Host "Error: The 'oc.exe' CLI tool is not installed or not available in the system PATH." -ForegroundColor Red
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    # Execute login command
    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        Write-Host "Login successful to $ClusterName as $Username." -ForegroundColor Green
    } else {
        Write-Host "Login failed. Details: $loginResult" -ForegroundColor Red
        exit 1
    }
}

# Function to list OpenShift projects
function List-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Failed to retrieve projects. Make sure you are logged in and have the necessary permissions." -ForegroundColor Red
        exit 1
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Function to switch OpenShift projects
function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        Write-Host "Switched to project '$ProjectName'." -ForegroundColor Green
    } else {
        Write-Host "Failed to switch to project '$ProjectName'. Details: $switchOutput" -ForegroundColor Red
        exit 1
    }
}

# Function to launch GUI
function Launch-GUI {
    # Import necessary assemblies for WPF
    Add-Type -AssemblyName PresentationFramework
    [void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
    [void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")

    # Create GUI components (as per your provided script)
    $form = New-Object System.Windows.Forms.Form
    $form.Text = "OpenShift Login and Projects"
    $form.Size = New-Object System.Drawing.Size(500, 350)
    $form.StartPosition = "CenterScreen"

    # Add GUI elements (as per your original script)
    # ...

    # Add GUI functionality (e.g., login, project list, switch project actions)
    # ...

    # Show the form
    [void] $form.ShowDialog()
}

# Entry point of the script
if ($args.Count -eq 0) {
    # No arguments passed, run in GUI mode
    Launch-GUI
} else {
    # Arguments provided, run in command-line mode
    if ($args.Count -lt 3) {
        Write-Host "Usage: .\script.ps1 <ClusterName> <Username> <Password>" -ForegroundColor Yellow
        exit 1
    }

    # Assign arguments to variables
    $ClusterName = $args[0]
    $Username = $args[1]
    $Password = $args[2]

    # Perform operations
    Check-OcExecutable
    Login-ToOpenShift -ClusterName $ClusterName -Username $Username -Password $Password

    # List projects and display them
    $projects = List-OpenShiftProjects
    Write-Host "Available Projects:" -ForegroundColor Cyan
    $projects | ForEach-Object { Write-Host $_ }

    # Prompt to switch project
    Write-Host "`nEnter a project name to switch to (or press Enter to skip):" -ForegroundColor Yellow
    $selectedProject = Read-Host "Project Name"
    if ($selectedProject -and ($selectedProject -in $projects)) {
        Switch-OpenShiftProject -ProjectName $selectedProject
    } else {
        Write-Host "No project switched. Exiting..." -ForegroundColor Yellow
    }
}
```

---

### How It Works:
1. **Interactive Mode (GUI)**:
   - If no arguments are passed, the script launches the GUI using `Launch-GUI`.

2. **Command-Line Mode**:
   - If arguments are passed, it performs the login and retrieves the project list.
   - Displays available projects in the terminal.
   - Prompts the user to optionally switch to a project by entering its name.

3. **Command-Line Arguments**:
   - The script expects the following arguments for command-line mode:
     - `<ClusterName>`: The OpenShift cluster name (e.g., `Cluster1`).
     - `<Username>`: The OpenShift username.
     - `<Password>`: The OpenShift password.

4. **Checks**:
   - Validates the presence of `oc.exe` in the PATH.
   - Ensures valid input and displays error messages if issues occur.

---

### How to Use:
1. **GUI Mode**:
   - Run the script without any arguments:
     ```powershell
     .\OpenShiftScript.ps1
     ```

2. **Command-Line Mode**:
   - Provide cluster name, username, and password as arguments:
     ```powershell
     .\OpenShiftScript.ps1 Cluster1 your-username your-password
     ```

This dual-mode functionality allows the script to cater to both interactive and non-interactive use cases.

**Swapan**: can this be converted to using WPF?

**ChatGPT**: Yes, the script can be converted to use **Windows Presentation Foundation (WPF)** for the graphical interface, which allows for more modern and customizable UI components compared to Windows Forms. Below is the updated version of the script using **WPF** for the GUI.

---

### Full Script Using WPF

```powershell
# Check if oc.exe exists in PATH
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        Write-Host "Error: The 'oc.exe' CLI tool is not installed or not available in the system PATH." -ForegroundColor Red
        exit 1
    }
}

# Function to log in to OpenShift
function Login-ToOpenShift {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    # Execute login command
    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        return "Login successful to $ClusterName as $Username."
    } else {
        throw "Login failed: $loginResult"
    }
}

# Function to get OpenShift projects
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Function to switch OpenShift projects
function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        return "Switched to project '$ProjectName'."
    } else {
        throw "Failed to switch to project '$ProjectName': $switchOutput"
    }
}

# Function to launch WPF-based GUI
function Launch-GUI {
    # Import required .NET assemblies
    Add-Type -AssemblyName PresentationCore, PresentationFramework, WindowsBase

    # Create XAML for the GUI
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5">
            <ComboBoxItem>Cluster1</ComboBoxItem>
            <ComboBoxItem>Cluster2</ComboBoxItem>
            <ComboBoxItem>Cluster3</ComboBoxItem>
        </ComboBox>

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    # Parse the XAML
    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    # Get references to GUI elements
    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    # Event handlers
    $loginButton.Add_Click({
        try {
            $clusterName = $clusterComboBox.SelectedItem.Content
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Perform login
            $loginMessage = Login-ToOpenShift -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            # Fetch projects
            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProjects
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Switch project
            $switchMessage = Switch-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $cancelButton.Add_Click({
        $window.Close()
    })

    # Show the window
    $window.ShowDialog() | Out-Null
}

# Main entry point
if ($args.Count -eq 0) {
    Launch-GUI
} else {
    Write-Host "This script currently only supports the GUI mode."
}
```

---

### Key Changes for WPF:
1. **XAML for Layout**:
   - The UI layout is defined using **XAML** for a modern and flexible design.
   - WPF supports advanced layouts, styles, and animations if needed.

2. **Dynamic Events**:
   - Event handlers (e.g., button clicks) are added dynamically to interact with the GUI elements.

3. **Controls**:
   - **ComboBox**: For selecting a cluster.
   - **TextBox** and **PasswordBox**: For username and password input.
   - **ListBox**: To display OpenShift projects.
   - **Buttons**: For login, project switching, and cancel actions.

4. **Error Handling**:
   - WPF message boxes are used for error messages and notifications.

---

### How to Run:
1. Save this script as `OpenShiftLoginWPF.ps1`.
2. Run the script in PowerShell:
   ```powershell
   .\OpenShiftLoginWPF.ps1
   ```

This WPF-based version provides a more modern and visually appealing UI while maintaining functionality for OpenShift login and project management.

**Swapan**: seperate funcotinality into different modules

**ChatGPT**: To separate the functionality into different modules for better maintainability and modularity, you can split the script into multiple PowerShell modules (`.psm1`) and a main script (`.ps1`). Here's how you can organize the functionality:

---

### Project Structure
```
OpenShiftScript/
│
├── Modules/
│   ├── CheckEnvironment.psm1
│   ├── OpenShiftLogin.psm1
│   ├── ProjectManagement.psm1
│
├── Main.ps1
```

---

### 1. **`CheckEnvironment.psm1`** (Check environment and dependencies)
This module ensures that `oc.exe` exists and is available in the system's PATH.

```powershell
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        throw "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again."
    }
}

Export-ModuleMember -Function Check-OcExecutable
```

---

### 2. **`OpenShiftLogin.psm1`** (Login functionality)
This module contains functions for logging into OpenShift.

```powershell
function Login-ToOpenShift {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        return "Login successful to $ClusterName as $Username."
    } else {
        throw "Login failed: $loginResult"
    }
}

Export-ModuleMember -Function Login-ToOpenShift
```

---

### 3. **`ProjectManagement.psm1`** (Project management functionality)
This module includes functions for listing and switching OpenShift projects.

```powershell
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        return "Switched to project '$ProjectName'."
    } else {
        throw "Failed to switch to project '$ProjectName': $switchOutput"
    }
}

Export-ModuleMember -Function Get-OpenShiftProjects, Switch-OpenShiftProject
```

---

### 4. **`Main.ps1`** (Main entry point with WPF GUI)
This is the main script that initializes the WPF GUI and uses the modules to perform specific tasks.

```powershell
# Import modules
Import-Module -Name "$PSScriptRoot/Modules/CheckEnvironment.psm1"
Import-Module -Name "$PSScriptRoot/Modules/OpenShiftLogin.psm1"
Import-Module -Name "$PSScriptRoot/Modules/ProjectManagement.psm1"

# Import required .NET assemblies for WPF
Add-Type -AssemblyName PresentationCore, PresentationFramework

# Function to launch WPF-based GUI
function Launch-GUI {
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5">
            <ComboBoxItem>Cluster1</ComboBoxItem>
            <ComboBoxItem>Cluster2</ComboBoxItem>
            <ComboBoxItem>Cluster3</ComboBoxItem>
        </ComboBox>

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    $loginButton.Add_Click({
        try {
            $clusterName = $clusterComboBox.SelectedItem.Content
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Perform login
            $loginMessage = Login-ToOpenShift -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            # Fetch projects
            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProjects
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $switchMessage = Switch-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $cancelButton.Add_Click({
        $window.Close()
    })

    $window.ShowDialog() | Out-Null
}

# Main entry point
Check-OcExecutable
Launch-GUI
```

---

### Advantages of Modularization:
1. **Code Reusability**: Each module focuses on a specific functionality, making it easier to reuse and test.
2. **Better Maintenance**: Updates can be made in individual modules without affecting the rest of the code.
3. **Improved Readability**: The main script becomes simpler to read and understand.

Now, your script is modular and easier to manage!

**Swapan**: seperate further into GUI and Console

**ChatGPT**: To separate the functionality into **GUI** and **Console** modes, you can further organize the code into a more modular structure where you have distinct logic for the GUI and Console modes. Here’s how you can structure it:

---

### Project Structure

```
OpenShiftScript/
│
├── Modules/
│   ├── CheckEnvironment.psm1
│   ├── OpenShiftLogin.psm1
│   ├── ProjectManagement.psm1
│
├── Modes/
│   ├── ConsoleMode.psm1
│   ├── GuiMode.psm1
│
├── Main.ps1
```

---

### 1. **`CheckEnvironment.psm1`** (Environment checks)

```powershell
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        throw "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again."
    }
}

Export-ModuleMember -Function Check-OcExecutable
```

---

### 2. **`OpenShiftLogin.psm1`** (Login logic)

```powershell
function Login-ToOpenShift {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        return "Login successful to $ClusterName as $Username."
    } else {
        throw "Login failed: $loginResult"
    }
}

Export-ModuleMember -Function Login-ToOpenShift
```

---

### 3. **`ProjectManagement.psm1`** (Project operations)

```powershell
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

function Switch-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        return "Switched to project '$ProjectName'."
    } else {
        throw "Failed to switch to project '$ProjectName': $switchOutput"
    }
}

Export-ModuleMember -Function Get-OpenShiftProjects, Switch-OpenShiftProject
```

---

### 4. **`GuiMode.psm1`** (GUI-specific functionality)

```powershell
function Launch-GUI {
    # Import required .NET assemblies
    Add-Type -AssemblyName PresentationCore, PresentationFramework

    # Create XAML for the GUI
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5">
            <ComboBoxItem>Cluster1</ComboBoxItem>
            <ComboBoxItem>Cluster2</ComboBoxItem>
            <ComboBoxItem>Cluster3</ComboBoxItem>
        </ComboBox>

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    $loginButton.Add_Click({
        try {
            $clusterName = $clusterComboBox.SelectedItem.Content
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $loginMessage = Login-ToOpenShift -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProjects
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $switchMessage = Switch-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    $cancelButton.Add_Click({
        $window.Close()
    })

    $window.ShowDialog() | Out-Null
}

Export-ModuleMember -Function Launch-GUI
```

---

### 5. **`ConsoleMode.psm1`** (Console-specific functionality)

```powershell
function Launch-Console {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    try {
        Check-OcExecutable
        $loginMessage = Login-ToOpenShift -ClusterName $ClusterName -Username $Username -Password $Password
        Write-Host $loginMessage -ForegroundColor Green

        $projects = Get-OpenShiftProjects
        Write-Host "Available Projects:" -ForegroundColor Cyan
        $projects | ForEach-Object { Write-Host $_ }

        Write-Host "Enter a project name to switch to (or press Enter to skip):" -ForegroundColor Yellow
        $selectedProject = Read-Host "Project Name"
        if ($selectedProject -and ($selectedProject -in $projects)) {
            $switchMessage = Switch-OpenShiftProject -ProjectName $selectedProject
            Write-Host $switchMessage -ForegroundColor Green
        } else {
            Write-Host "No project switched." -ForegroundColor Yellow
        }
    } catch {
        Write-Host $_.Exception

**ChatGPT**: .Message -ForegroundColor Red
    }
}

Export-ModuleMember -Function Launch-Console
```

---

### 6. **`Main.ps1`** (Main script to switch between GUI and Console)

```powershell
# Import modules
Import-Module -Name "$PSScriptRoot/Modules/CheckEnvironment.psm1"
Import-Module -Name "$PSScriptRoot/Modules/OpenShiftLogin.psm1"
Import-Module -Name "$PSScriptRoot/Modules/ProjectManagement.psm1"
Import-Module -Name "$PSScriptRoot/Modes/ConsoleMode.psm1"
Import-Module -Name "$PSScriptRoot/Modes/GuiMode.psm1"

# Entry point
if ($args.Count -eq 0) {
    Launch-GUI
} else {
    if ($args.Count -lt 3) {
        Write-Host "Usage: .\Main.ps1 <ClusterName> <Username> <Password>" -ForegroundColor Yellow
        exit 1
    }

    $ClusterName = $args[0]
    $Username = $args[1]
    $Password = $args[2]

    Launch-Console -ClusterName $ClusterName -Username $Username -Password $Password
}
```

---

### Advantages of This Structure:
1. **Clear Separation**:
   - GUI logic is handled in `GuiMode.psm1`.
   - Console logic is handled in `ConsoleMode.psm1`.

2. **Reusability**:
   - Modules like `CheckEnvironment.psm1` and `ProjectManagement.psm1` can be reused across different projects.

3. **Ease of Maintenance**:
   - Each module focuses on a specific functionality, making it easier to test and update.

Now you have a fully modular script with separate GUI and Console modes!

**Swapan**: can i call power scripts to get  oc project list?

**ChatGPT**: Yes, you can call a PowerShell script to retrieve the list of OpenShift projects (`oc get projects`) and pass the result back to the calling script or GUI. Here's an example of how you can structure it:

---

### Script to Retrieve OpenShift Projects (`GetProjects.ps1`)

Save this script as `GetProjects.ps1`. It retrieves the list of OpenShift projects and outputs them.

```powershell
# Check if oc.exe is available
function Check-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        throw "The 'oc.exe' CLI tool is not installed or not available in the system PATH."
    }
}

# Get OpenShift projects
function Get-OpenShiftProjects {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Main Execution
try {
    Check-OcExecutable
    $projects = Get-OpenShiftProjects
    $projects | ForEach-Object { Write-Output $_ }
} catch {
    Write-Output $_.Exception.Message
    exit 1
}
```

---

### Script to Call `GetProjects.ps1` (Caller Script)

Here’s an example script (e.g., `Main.ps1`) that calls the `GetProjects.ps1` script and retrieves the project list:

```powershell
# Path to the GetProjects.ps1 script
$GetProjectsScript = "C:\Path\To\GetProjects.ps1"

# Call the script and capture the output
try {
    $projectList = & powershell.exe -NoProfile -ExecutionPolicy Bypass -File $GetProjectsScript
    if ($LASTEXITCODE -eq 0) {
        Write-Host "Available Projects:" -ForegroundColor Cyan
        $projectList | ForEach-Object { Write-Host $_ }
    } else {
        Write-Host "Error fetching projects:" -ForegroundColor Red
        Write-Host $projectList
    }
} catch {
    Write-Host "An error occurred: $_" -ForegroundColor Red
}
```

---

### Explanation

1. **`GetProjects.ps1`**:
   - Contains the logic to check for `oc.exe`, retrieve the list of projects, and return them as output.
   - If an error occurs, it throws an exception or writes an error message.

2. **Calling Script (`Main.ps1`)**:
   - Uses `powershell.exe` to invoke the `GetProjects.ps1` script.
   - Captures the output from the called script using `&`.
   - Checks `$LASTEXITCODE` to determine if the script ran successfully.

3. **Integration with GUI**:
   - You can use the output of the `GetProjects.ps1` script to populate a dropdown or list in your GUI.

---

### Example GUI Integration

Here's how you can integrate this logic into a WPF-based GUI:

```powershell
# Function to get OpenShift projects and update the ListBox in the GUI
function Update-ProjectList {
    param ($ListBox)

    try {
        $GetProjectsScript = "C:\Path\To\GetProjects.ps1"
        $projectList = & powershell.exe -NoProfile -ExecutionPolicy Bypass -File $GetProjectsScript

        if ($LASTEXITCODE -eq 0) {
            $ListBox.Items.Clear()
            $projectList | ForEach-Object { $ListBox.Items.Add($_) }
        } else {
            [System.Windows.MessageBox]::Show("Error fetching projects: $projectList", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    } catch {
        [System.Windows.MessageBox]::Show("An error occurred: $_", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
    }
}

# Example usage in GUI:
# Call Update-ProjectList and pass the ListBox control as a parameter
Update-ProjectList -ListBox $projectsListBox
```

---

### Benefits of this Approach:
1. **Modularity**:
   - The `GetProjects.ps1` script is reusable in other projects or contexts.
2. **Error Handling**:
   - Any errors in retrieving the project list are handled cleanly.
3. **Integration**:
   - Can be called directly from other scripts or GUIs using PowerShell.

Now you can efficiently call PowerShell scripts to retrieve the OpenShift project list and integrate the results into your application!

**Swapan**: for all Powershell modules for this project convert to all approved verbs

**ChatGPT**: In PowerShell, **approved verbs** ensure consistent and predictable naming for functions and cmdlets. Below is how the existing modules and their functions can be updated to use **approved verbs**:

---

### 1. **CheckEnvironment Module (`CheckEnvironment.psm1`)**

**Original Function**:
```powershell
function Check-OcExecutable { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Test-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        throw "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again."
    }
}

Export-ModuleMember -Function Test-OcExecutable
```

**Explanation**:
- Changed `Check-` to `Test-`, which is an approved verb that reflects verifying the presence of something.

---

### 2. **OpenShiftLogin Module (`OpenShiftLogin.psm1`)**

**Original Function**:
```powershell
function Login-ToOpenShift { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Connect-OpenShiftCluster {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        return "Login successful to $ClusterName as $Username."
    } else {
        throw "Login failed: $loginResult"
    }
}

Export-ModuleMember -Function Connect-OpenShiftCluster
```

**Explanation**:
- Changed `Login-` to `Connect-`, which is the approved verb for establishing a connection to a resource.

---

### 3. **ProjectManagement Module (`ProjectManagement.psm1`)**

#### a) Function to Get Projects

**Original Function**:
```powershell
function Get-OpenShiftProjects { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Get-OpenShiftProject {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

Export-ModuleMember -Function Get-OpenShiftProject
```

**Explanation**:
- No change is required here as `Get-` is already an approved verb for retrieving data.

---

#### b) Function to Switch Projects

**Original Function**:
```powershell
function Switch-OpenShiftProject { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Set-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        return "Switched to project '$ProjectName'."
    } else {
        throw "Failed to switch to project '$ProjectName': $switchOutput"
    }
}

Export-ModuleMember -Function Set-OpenShiftProject
```

**Explanation**:
- Changed `Switch-` to `Set-`, which is the approved verb for modifying a state or configuration.

---

### 4. **GuiMode Module (`GuiMode.psm1`)**

#### a) Function to Launch GUI

**Original Function**:
```powershell
function Launch-GUI { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Show-OpenShiftGui {
    # GUI implementation remains the same
}

Export-ModuleMember -Function Show-OpenShiftGui
```

**Explanation**:
- Changed `Launch-` to `Show-`, which is the approved verb for displaying something to the user.

---

### 5. **ConsoleMode Module (`ConsoleMode.psm1`)**

#### a) Function to Launch Console Mode

**Original Function**:
```powershell
function Launch-Console { ... }
```

**Updated Function with Approved Verb**:
```powershell
function Invoke-OpenShiftConsole {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    try {
        Test-OcExecutable
        $loginMessage = Connect-OpenShiftCluster -ClusterName $ClusterName -Username $Username -Password $Password
        Write-Host $loginMessage -ForegroundColor Green

        $projects = Get-OpenShiftProject
        Write-Host "Available Projects:" -ForegroundColor Cyan
        $projects | ForEach-Object { Write-Host $_ }

        Write-Host "Enter a project name to switch to (or press Enter to skip):" -ForegroundColor Yellow
        $selectedProject = Read-Host "Project Name"
        if ($selectedProject -and ($selectedProject -in $projects)) {
            $switchMessage = Set-OpenShiftProject -ProjectName $selectedProject
            Write-Host $switchMessage -ForegroundColor Green
        } else {
            Write-Host "No project switched." -ForegroundColor Yellow
        }
    } catch {
        Write-Host $_.Exception.Message -ForegroundColor Red
    }
}

Export-ModuleMember -Function Invoke-OpenShiftConsole
```

**Explanation**:
- Changed `Launch-` to `Invoke-`, which is the approved verb for performing an operation or action.

---

### Updated `Main.ps1`

The `Main.ps1` script integrates the updated functions. Example:

```powershell
# Import modules
Import-Module -Name "$PSScriptRoot/Modules/CheckEnvironment.psm1"
Import-Module -Name "$PSScriptRoot/Modules/OpenShiftLogin.psm1"
Import-Module -Name "$PSScriptRoot/Modules/ProjectManagement.psm1"
Import-Module -Name "$PSScriptRoot/Modes/ConsoleMode.psm1"
Import-Module -Name "$PSScriptRoot/Modes/GuiMode.psm1"

# Entry point
if ($args.Count -eq 0) {
    Show-OpenShiftGui
} else {
    if ($args.Count -lt 3) {
        Write-Host "Usage: .\Main.ps1 <ClusterName> <Username> <Password>" -ForegroundColor Yellow
        exit 1
    }

    $ClusterName = $args[0]
    $Username = $args[1]
    $Password = $args[2]

    Invoke-OpenShiftConsole -ClusterName $ClusterName -Username $Username -Password $Password
}
```

---

### Summary of Updated Function Names

| Original Name               | Updated Name                   |
|-----------------------------|--------------------------------|
| `Check-OcExecutable`        | `Test-OcExecutable`           |
| `Login-ToOpenShift`         | `Connect-OpenShiftCluster`    |
| `Get-OpenShiftProjects`     | `Get-OpenShiftProject`        |
| `Switch-OpenShiftProject`   | `Set-OpenShiftProject`        |
| `Launch-GUI`                | `Show-OpenShiftGui`           |
| `Launch-Console`            | `Invoke-OpenShiftConsole`     |

By using approved verbs, the PowerShell modules align with industry best practices, improving readability and consistency.

**Swapan**: project listbox should only  become visible after login is successful 

**ChatGPT**: To make the **project ListBox** visible only after a successful login in the WPF-based GUI, you can modify the `Launch-GUI` function to initially hide the `ListBox` and then show it once the login is successful.

Here’s how you can update your GUI logic:

---

### Updated `Launch-GUI` Function

```powershell
function Show-OpenShiftGui {
    # Import required .NET assemblies
    Add-Type -AssemblyName PresentationCore, PresentationFramework

    # Create XAML for the GUI
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5">
            <ComboBoxItem>Cluster1</ComboBoxItem>
            <ComboBoxItem>Cluster2</ComboBoxItem>
            <ComboBoxItem>Cluster3</ComboBoxItem>
        </ComboBox>

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List (initially hidden) -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" Visibility="Collapsed" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" IsEnabled="False" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    # Retrieve GUI elements
    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    # Event handler for Login button
    $loginButton.Add_Click({
        try {
            $clusterName = $clusterComboBox.SelectedItem.Content
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Perform login
            $loginMessage = Connect-OpenShiftCluster -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            # Fetch projects
            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProject
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }

            # Show the project ListBox and enable the switch button
            $projectsListBox.Visibility = "Visible"
            $switchProjectButton.IsEnabled = $true
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Switch Project button
    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $switchMessage = Set-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Cancel button
    $cancelButton.Add_Click({
        $window.Close()
    })

    # Show the window
    $window.ShowDialog() | Out-Null
}
```

---

### Key Updates:

1. **Initial Visibility of Project ListBox**:
   - The `ProjectsListBox` is initially set to `Visibility="Collapsed"` in the XAML definition.
   - It becomes visible only after a successful login.

2. **Enabling the Switch Project Button**:
   - The `SwitchProjectButton` is initially `IsEnabled="False"`.
   - It becomes enabled after the project list is populated.

3. **Steps in the Login Process**:
   - Perform the login using the `Connect-OpenShiftCluster` function.
   - Retrieve the list of projects using the `Get-OpenShiftProject` function.
   - Populate the `ProjectsListBox` and make it visible.

4. **Error Handling**:
   - Proper error handling ensures the GUI doesn't crash if login or project fetching fails.

---

### Behavior Flow:

1. **Before Login**:
   - The `ProjectsListBox` is hidden.
   - The `Switch Project` button is disabled.

2. **After Successful Login**:
   - The `ProjectsListBox` is populated with the project names and becomes visible.
   - The `Switch Project` button is enabled.

3. **Switch Project**:
   - Allows the user to select a project from the `ProjectsListBox` and switch to it.

---

### Example Integration:

- Ensure the `Connect-OpenShiftCluster`, `Get-OpenShiftProject`, and `Set-OpenShiftProject` functions are implemented in their respective modules and imported correctly.
- Use `Show-OpenShiftGui` in your main entry point to display the GUI.

This implementation improves user experience by dynamically showing the project list only when it is relevant.

**Swapan**: get cluster list from property file for ClusterComboBox

**ChatGPT**: To dynamically populate the `ClusterComboBox` in the WPF GUI with cluster names from a property file, you can read the file at runtime and bind its contents to the `ClusterComboBox`.

Here’s how you can implement it:

---

### Example Property File (`clusters.properties`)

Create a file named `clusters.properties` with the following content:

```
Cluster1=https://api.openshift.cluster1.example.com
Cluster2=https://api.openshift.cluster2.example.com
Cluster3=https://api.openshift.cluster3.example.com
```

---

### Updated `Show-OpenShiftGui` Function

This version reads the cluster names from the property file and populates the `ClusterComboBox`.

```powershell
function Show-OpenShiftGui {
    # Import required .NET assemblies
    Add-Type -AssemblyName PresentationCore, PresentationFramework

    # Create XAML for the GUI
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List (initially hidden) -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" Visibility="Collapsed" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" IsEnabled="False" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    # Retrieve GUI elements
    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    # Load clusters from the property file
    function Load-ClustersFromFile {
        param ([string]$FilePath)

        if (-not (Test-Path $FilePath)) {
            throw "Cluster property file not found: $FilePath"
        }

        $clusters = @{}
        Get-Content $FilePath | ForEach-Object {
            if ($_ -match "^(?<key>[^=]+)=(?<value>.+)$") {
                $clusters[$matches.key.Trim()] = $matches.value.Trim()
            }
        }
        return $clusters
    }

    # Path to the cluster property file
    $propertyFilePath = "C:\Path\To\clusters.properties"
    try {
        $clusters = Load-ClustersFromFile -FilePath $propertyFilePath
        foreach ($cluster in $clusters.Keys) {
            $comboBoxItem = New-Object System.Windows.Controls.ComboBoxItem
            $comboBoxItem.Content = $cluster
            $comboBoxItem.Tag = $clusters[$cluster]
            $clusterComboBox.Items.Add($comboBoxItem)
        }
    } catch {
        [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        return
    }

    # Event handler for Login button
    $loginButton.Add_Click({
        try {
            $selectedItem = $clusterComboBox.SelectedItem
            if (-not $selectedItem) {
                [System.Windows.MessageBox]::Show("Please select a cluster.", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $clusterName = $selectedItem.Content
            $clusterUrl = $selectedItem.Tag
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Perform login
            $loginMessage = Connect-OpenShiftCluster -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            # Fetch projects
            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProject
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }

            # Show the project ListBox and enable the switch button
            $projectsListBox.Visibility = "Visible"
            $switchProjectButton.IsEnabled = $true
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Switch Project button
    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $switchMessage = Set-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Cancel button
    $cancelButton.Add_Click({
        $window.Close()
    })

    # Show the window
    $window.ShowDialog() | Out-Null
}
```

---

### Key Updates:

1. **`Load-ClustersFromFile` Function**:
   - Reads the `clusters.properties` file and parses each line as a key-value pair (`ClusterName=ClusterURL`).
   - Returns a hashtable where the key is the cluster name, and the value is the URL.

2. **Binding Clusters to ComboBox**:
   - For each cluster in the property file, a `ComboBoxItem` is created with:
     - `Content`: Cluster name.
     - `Tag`: Cluster URL (can be used later for login).

3. **Dynamic Cluster List**:
   - The clusters in the property file dynamically populate the `ClusterComboBox` when the GUI launches.

4. **Error Handling**:
   - Displays an error message if the property file is missing or incorrectly formatted.

---

### Example Usage:

1. Save the `clusters.properties` file in the desired path.
2. Update the `$propertyFilePath` in the script with the correct file path.
3. Run the script:
   ```powershell
   .\Main.ps1
   ```
4. The `ClusterComboBox` will be dynamically populated with clusters from the `clusters.properties` file.

This approach allows you to manage cluster configurations externally without hardcoding them into the script.

**Swapan**: merge all functions and main into one .ps1 script

**ChatGPT**: Here's the consolidated script where all functions and the main entry logic are combined into a single `.ps1` file. This script includes everything: cluster property file loading, GUI logic, OpenShift login, project fetching, and switching functionality.

---

### Consolidated Script (`OpenShiftManager.ps1`)

```powershell
# Import required .NET assemblies
Add-Type -AssemblyName PresentationCore, PresentationFramework

# Function to check if oc.exe is available
function Test-OcExecutable {
    if (-not (Get-Command "oc.exe" -ErrorAction SilentlyContinue)) {
        throw "The 'oc.exe' CLI tool is not installed or not available in the system PATH. Please install it and try again."
    }
}

# Function to connect to OpenShift cluster
function Connect-OpenShiftCluster {
    param (
        [string]$ClusterName,
        [string]$Username,
        [string]$Password
    )

    $BaseUrl = "https://api.openshift.{0}.example.com:6443"
    $OpenShiftUrl = $BaseUrl -f $ClusterName.ToLower()

    $loginResult = & oc.exe login $OpenShiftUrl --username=$Username --password=$Password --insecure-skip-tls-verify 2>&1

    if ($loginResult -match "Login successful") {
        return "Login successful to $ClusterName as $Username."
    } else {
        throw "Login failed: $loginResult"
    }
}

# Function to get OpenShift projects
function Get-OpenShiftProject {
    $projectsOutput = & oc.exe get projects -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}" 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve projects: $projectsOutput"
    }
    return $projectsOutput -split "`n" | Where-Object { $_ -ne "" }
}

# Function to switch OpenShift projects
function Set-OpenShiftProject {
    param (
        [string]$ProjectName
    )
    $switchOutput = & oc.exe project $ProjectName 2>&1
    if ($switchOutput -match "Now using project") {
        return "Switched to project '$ProjectName'."
    } else {
        throw "Failed to switch to project '$ProjectName': $switchOutput"
    }
}

# Function to load clusters from a property file
function Load-ClustersFromFile {
    param ([string]$FilePath)

    if (-not (Test-Path $FilePath)) {
        throw "Cluster property file not found: $FilePath"
    }

    $clusters = @{}
    Get-Content $FilePath | ForEach-Object {
        if ($_ -match "^(?<key>[^=]+)=(?<value>.+)$") {
            $clusters[$matches.key.Trim()] = $matches.value.Trim()
        }
    }
    return $clusters
}

# Function to launch WPF GUI
function Show-OpenShiftGui {
    # Create XAML for the GUI
    $xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="OpenShift Login and Projects" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>

        <!-- Labels and Inputs -->
        <TextBlock Grid.Row="0" Grid.Column="0" Margin="5" VerticalAlignment="Center">Cluster Name:</TextBlock>
        <ComboBox x:Name="ClusterComboBox" Grid.Row="0" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="1" Grid.Column="0" Margin="5" VerticalAlignment="Center">Username:</TextBlock>
        <TextBox x:Name="UsernameTextBox" Grid.Row="1" Grid.Column="1" Margin="5" />

        <TextBlock Grid.Row="2" Grid.Column="0" Margin="5" VerticalAlignment="Center">Password:</TextBlock>
        <PasswordBox x:Name="PasswordBox" Grid.Row="2" Grid.Column="1" Margin="5" />

        <!-- Project List (initially hidden) -->
        <ListBox x:Name="ProjectsListBox" Grid.Row="3" Grid.ColumnSpan="2" Margin="5" Visibility="Collapsed" />

        <!-- Buttons -->
        <StackPanel Orientation="Horizontal" Grid.Row="4" Grid.ColumnSpan="2" HorizontalAlignment="Center" Margin="5">
            <Button x:Name="LoginButton" Content="Login" Width="100" Margin="5" />
            <Button x:Name="SwitchProjectButton" Content="Switch Project" Width="150" Margin="5" IsEnabled="False" />
            <Button x:Name="CancelButton" Content="Cancel" Width="100" Margin="5" />
        </StackPanel>
    </Grid>
</Window>
"@

    $reader = New-Object System.Xml.XmlNodeReader([xml]$xaml)
    $window = [Windows.Markup.XamlReader]::Load($reader)

    # Retrieve GUI elements
    $clusterComboBox = $window.FindName("ClusterComboBox")
    $usernameTextBox = $window.FindName("UsernameTextBox")
    $passwordBox = $window.FindName("PasswordBox")
    $projectsListBox = $window.FindName("ProjectsListBox")
    $loginButton = $window.FindName("LoginButton")
    $switchProjectButton = $window.FindName("SwitchProjectButton")
    $cancelButton = $window.FindName("CancelButton")

    # Load clusters from the property file
    $propertyFilePath = "C:\Path\To\clusters.properties"
    try {
        $clusters = Load-ClustersFromFile -FilePath $propertyFilePath
        foreach ($cluster in $clusters.Keys) {
            $comboBoxItem = New-Object System.Windows.Controls.ComboBoxItem
            $comboBoxItem.Content = $cluster
            $comboBoxItem.Tag = $clusters[$cluster]
            $clusterComboBox.Items.Add($comboBoxItem)
        }
    } catch {
        [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        return
    }

    # Event handler for Login button
    $loginButton.Add_Click({
        try {
            $selectedItem = $clusterComboBox.SelectedItem
            if (-not $selectedItem) {
                [System.Windows.MessageBox]::Show("Please select a cluster.", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $clusterName = $selectedItem.Content
            $username = $usernameTextBox.Text
            $password = $passwordBox.Password

            if (-not $clusterName -or -not $username -or -not $password) {
                [System.Windows.MessageBox]::Show("All fields are required!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            # Perform login
            $loginMessage = Connect-OpenShiftCluster -ClusterName $clusterName -Username $username -Password $password
            [System.Windows.MessageBox]::Show($loginMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)

            # Fetch projects
            $projectsListBox.Items.Clear()
            $projects = Get-OpenShiftProject
            foreach ($project in $projects) {
                $projectsListBox.Items.Add($project)
            }

            # Show the project ListBox and enable the switch button
            $projectsListBox.Visibility = "Visible"
            $switchProjectButton.IsEnabled = $true
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Switch Project button
    $switchProjectButton.Add_Click({
        try {
            $selectedProject = $projectsListBox.SelectedItem
            if (-not $selectedProject) {
                [System.Windows.MessageBox]::Show("Please select a project first!", "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
                return
            }

            $switchMessage = Set-OpenShiftProject -ProjectName $selectedProject
            [System.Windows.MessageBox]::Show($switchMessage, "Success", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Information)
        } catch {
            [System.Windows.MessageBox]::Show($_.Exception.Message, "Error", [System.Windows.MessageBoxButton]::OK, [System.Windows.MessageBoxImage]::Error)
        }
    })

    # Event handler for Cancel button
    $cancelButton.Add_Click({
        $window.Close()
    })

    # Show the window
    $window.ShowDialog() | Out-Null
}

# Entry Point
try {
    Test-OcExecutable
    Show-OpenShiftGui
} catch {
    Write-Host "Error: $_" -ForegroundColor Red
}
``

