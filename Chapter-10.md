# Chapter 10: Automation and Scripting with PowerShell

Clicking through Hyper-V Manager to manage a handful of VMs is fine. Clicking through it to manage fifty VMs is a slow, error-prone nightmare. PowerShell is the answer: it lets you perform any Hyper-V operation from the command line, script repetitive tasks, manage VMs remotely, and build automation that runs itself.

---

*An IT team decided to standardize their VM naming convention across 20 VMs. Renaming each one through Hyper-V Manager, one at a time, took over three hours -- including the time spent navigating menus, confirming dialogs, and waiting for each rename to complete. The same operation as a PowerShell one-liner: about 45 seconds.*

---

## 10.1 Which PowerShell?

There are two versions of PowerShell in active use:

**Windows PowerShell (version 5.1):** Built into Windows. The older version. The Hyper-V module works fully here. You'll find it by searching "PowerShell" in the Start menu.

**PowerShell (version 7+):** The modern, cross-platform successor. Also works with the Hyper-V module on Windows. Recommended for new scripts -- better error handling, faster in many cases, and actively developed. Download from [github.com/PowerShell/PowerShell](https://github.com/PowerShell/PowerShell).

For Hyper-V management, both work. When you see `pwsh` in command examples, that's PowerShell 7. `powershell` refers to Windows PowerShell 5.1.

> **Recommendation:** Install PowerShell 7 alongside Windows PowerShell. Use PowerShell 7 for new scripts and daily work. Windows PowerShell 5.1 remains on the system and is used by some older tools.

## 10.2 Your Scripting Environment

**VS Code with the PowerShell Extension** is the recommended environment for writing PowerShell scripts. It provides syntax highlighting, IntelliSense, inline documentation, debugging, and a built-in terminal -- everything you need.

Download VS Code: [code.visualstudio.com](https://code.visualstudio.com)
Install the PowerShell extension: Search "PowerShell" in the VS Code Extensions panel.

> **Note on Windows PowerShell ISE:** The original PowerShell scripting environment that shipped with Windows. It was deprecated by Microsoft in 2017 and is no longer actively developed. New scripts should be written in VS Code, not ISE. ISE is still present in Windows but won't receive new features or improvements.

## 10.3 Getting Started with the Hyper-V Module

The Hyper-V PowerShell module provides cmdlets for every Hyper-V operation. On Windows Server with the Hyper-V role installed, it's available automatically. On Windows 10/11, it's available when Hyper-V is enabled.

```powershell
# Verify the module is available
Get-Module -ListAvailable -Name Hyper-V

# View all available Hyper-V cmdlets
Get-Command -Module Hyper-V

# Get help for a specific cmdlet
Get-Help New-VM -Detailed
Get-Help New-VM -Examples
```

The module loads automatically when you run any Hyper-V cmdlet. You can also explicitly import it:
```powershell
Import-Module Hyper-V
```

> **Remote management note:** If you want to run Hyper-V cmdlets from a management workstation (not the Hyper-V host itself), you need either the Hyper-V Management Tools installed locally (via Windows Features) or PowerShell Remoting. You do not need the Hyper-V role installed on your management machine.

## 10.4 Managing VMs with PowerShell

### Creating a VM

```powershell
# Create a Generation 2 VM with a new 80GB VHDX
New-VM -Name "WebServer01" `
       -MemoryStartupBytes 4GB `
       -NewVHDPath "D:\VMs\WebServer01\WebServer01.vhdx" `
       -NewVHDSizeBytes 80GB `
       -Generation 2 `
       -SwitchName "External-LAN"

# Configure Dynamic Memory
Set-VMMemory -VMName "WebServer01" `
             -DynamicMemoryEnabled $true `
             -MinimumBytes 1GB `
             -MaximumBytes 8GB `
             -Buffer 20

# Set vCPU count
Set-VMProcessor -VMName "WebServer01" -Count 2

# Attach a DVD drive with an ISO for OS installation
Add-VMDvdDrive -VMName "WebServer01" `
               -Path "D:\ISOs\WindowsServer2022.iso"
```

### Managing VM State

```powershell
# Start, stop, and manage VMs
Start-VM -Name "WebServer01"
Stop-VM -Name "WebServer01" -Force          # Graceful shutdown (requests OS shutdown)
Stop-VM -Name "WebServer01" -TurnOff        # Force off (power cut)
Restart-VM -Name "WebServer01" -Force
Suspend-VM -Name "WebServer01"              # Pause execution
Resume-VM -Name "WebServer01"
Save-VM -Name "WebServer01"                 # Save state (hibernation)
```

### Querying VMs

```powershell
# List all VMs and their state
Get-VM

# Get detailed info about a specific VM
Get-VM -Name "WebServer01" | Format-List *

# Get all running VMs
Get-VM | Where-Object {$_.State -eq "Running"}

# Get VMs with less than 2GB RAM
Get-VM | Where-Object {$_.MemoryStartup -lt 2GB}

# Get all VMs and their CPU/RAM allocation
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime
```

### Working with Network Adapters

```powershell
# Add a network adapter to a VM and connect to a switch
Add-VMNetworkAdapter -VMName "WebServer01" -SwitchName "Internal-Lab"

# Connect an existing adapter to a different switch
Connect-VMNetworkAdapter -VMName "WebServer01" `
                         -Name "Network Adapter" `
                         -SwitchName "External-LAN"

# Set a VLAN on a network adapter
Set-VMNetworkAdapterVlan -VMName "WebServer01" -VlanId 10 -Access

# List all network adapters and their switch connections
Get-VMNetworkAdapter -VMName "WebServer01"
```

### Checkpoints

```powershell
# Create a checkpoint
Checkpoint-VM -Name "WebServer01" -SnapshotName "Before-IIS-Install"

# List all checkpoints for a VM
Get-VMSnapshot -VMName "WebServer01"

# Apply (revert to) a checkpoint
$checkpoint = Get-VMSnapshot -VMName "WebServer01" -Name "Before-IIS-Install"
Restore-VMSnapshot -VMSnapshot $checkpoint -Confirm:$false

# Delete a checkpoint
Remove-VMSnapshot -VMName "WebServer01" -Name "Before-IIS-Install"
```

## 10.5 Managing Virtual Switches

```powershell
# List all virtual switches
Get-VMSwitch

# Create switches
New-VMSwitch -Name "External-LAN" -NetAdapterName "Ethernet" -AllowManagementOS $true
New-VMSwitch -Name "Internal-Lab" -SwitchType Internal
New-VMSwitch -Name "Private-Test" -SwitchType Private

# Create a SET-teamed switch (NIC teaming)
New-VMSwitch -Name "TeamedSwitch" `
             -NetAdapterName "Ethernet","Ethernet 2" `
             -EnableEmbeddedTeaming $true

# Remove a switch
Remove-VMSwitch -Name "Private-Test"
```

## 10.6 Advanced Scripting: Real-World Examples

### Example 1: Automated VM Provisioning

This script creates a VM, configures it fully, and handles errors gracefully:

```powershell
#Requires -Module Hyper-V
<#
.SYNOPSIS
    Creates and configures a new Hyper-V virtual machine.
.PARAMETER VMName
    Name of the VM to create.
.PARAMETER SwitchName
    Name of the virtual switch to connect to.
.PARAMETER ISOPath
    Full path to the OS installation ISO.
#>
param (
    [Parameter(Mandatory)]
    [string]$VMName,

    [Parameter(Mandatory)]
    [string]$SwitchName,

    [string]$ISOPath = "",
    [string]$VMPath = "D:\VMs",
    [long]$MemoryGB = 4,
    [long]$DiskGB = 80,
    [int]$vCPUs = 2
)

# Validate inputs
if (-not (Get-VMSwitch -Name $SwitchName -ErrorAction SilentlyContinue)) {
    Write-Error "Virtual switch '$SwitchName' does not exist. Create it first."
    exit 1
}

$vhdPath = "$VMPath\$VMName\$VMName.vhdx"

try {
    Write-Host "Creating VM: $VMName" -ForegroundColor Cyan

    New-VM -Name $VMName `
           -MemoryStartupBytes ($MemoryGB * 1GB) `
           -NewVHDPath $vhdPath `
           -NewVHDSizeBytes ($DiskGB * 1GB) `
           -Generation 2 `
           -SwitchName $SwitchName `
           -ErrorAction Stop

    Set-VMMemory -VMName $VMName `
                 -DynamicMemoryEnabled $true `
                 -MinimumBytes 1GB `
                 -MaximumBytes ($MemoryGB * 2 * 1GB) `
                 -Buffer 20

    Set-VMProcessor -VMName $VMName -Count $vCPUs

    Set-VMFirmware -VMName $VMName -EnableSecureBoot On

    if ($ISOPath -and (Test-Path $ISOPath)) {
        Add-VMDvdDrive -VMName $VMName -Path $ISOPath
        # Set DVD as first boot device
        $dvd = Get-VMDvdDrive -VMName $VMName
        Set-VMFirmware -VMName $VMName -FirstBootDevice $dvd
    }

    Write-Host "VM '$VMName' created successfully." -ForegroundColor Green
    Get-VM -Name $VMName | Select-Object Name, State, Generation, MemoryStartup

} catch {
    Write-Error "Failed to create VM '$VMName': $_"
    # Clean up partially created VM if it exists
    if (Get-VM -Name $VMName -ErrorAction SilentlyContinue) {
        Remove-VM -Name $VMName -Force
        if (Test-Path $vhdPath) { Remove-Item $vhdPath -Force }
    }
    exit 1
}
```

### Example 2: Daily Health Check Report

```powershell
# Generate a daily health report for all VMs
$report = Get-VM | ForEach-Object {
    $vm = $_
    $snapshots = (Get-VMSnapshot -VMName $vm.Name -ErrorAction SilentlyContinue).Count
    [PSCustomObject]@{
        Name             = $vm.Name
        State            = $vm.State
        CPUUsage         = "$($vm.CPUUsage)%"
        MemoryAssigned   = "$([Math]::Round($vm.MemoryAssigned / 1GB, 1)) GB"
        Uptime           = $vm.Uptime
        Checkpoints      = $snapshots
        CheckpointAlert  = if ($snapshots -gt 3) { "WARNING: $snapshots checkpoints" } else { "OK" }
    }
}

$report | Format-Table -AutoSize

# Write to a log file
$report | Export-Csv -Path "C:\Logs\VMHealthReport_$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation
```

### Example 3: Bulk Checkpoint Before Patch Tuesday

```powershell
# Create checkpoints on all running VMs before applying updates
$timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm"
$checkpointName = "PrePatchTuesday_$timestamp"

Get-VM | Where-Object {$_.State -eq "Running"} | ForEach-Object {
    Write-Host "Creating checkpoint for $($_.Name)..."
    Checkpoint-VM -Name $_.Name -SnapshotName $checkpointName
}

Write-Host "Checkpoints created: $checkpointName" -ForegroundColor Green
```

## 10.7 PowerShell Remoting

Manage remote Hyper-V hosts without being physically present or using RDP:

```powershell
# Enable PowerShell Remoting on the remote host (run once, on the host)
Enable-PSRemoting -Force

# Run a command on a remote host
Invoke-Command -ComputerName "HyperVHost01" -ScriptBlock {
    Get-VM | Select-Object Name, State
}

# Start an interactive session on a remote host
Enter-PSSession -ComputerName "HyperVHost01"
# Now all commands run on the remote host
Get-VM
Exit-PSSession

# Run Hyper-V cmdlets against a remote host (many cmdlets have -ComputerName parameter)
Get-VM -ComputerName "HyperVHost01"
Start-VM -Name "WebServer01" -ComputerName "HyperVHost01"
```

## 10.8 Scheduling PowerShell Scripts

Task Scheduler runs scripts on a schedule -- useful for health checks, backup notifications, and maintenance tasks.

```powershell
# Create a scheduled task that runs a script daily at 7 AM
$action = New-ScheduledTaskAction `
    -Execute "pwsh.exe" `
    -Argument "-NonInteractive -File C:\Scripts\VMHealthCheck.ps1"

$trigger = New-ScheduledTaskTrigger -Daily -At "07:00"

$settings = New-ScheduledTaskSettingsSet -RunOnlyIfNetworkAvailable

Register-ScheduledTask `
    -TaskName "HyperV-DailyHealthCheck" `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -RunLevel Highest `
    -User "SYSTEM"
```

## 10.9 Security: Safe Script Practices

**Script Execution Policy:** By default, PowerShell may block running unsigned scripts. Set an appropriate policy:

```powershell
# Allow locally written scripts and signed remote scripts
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine
```

**Secure credential handling:** Never hard-code passwords in scripts:

```powershell
# Good: prompt for credentials at runtime
$cred = Get-Credential -Message "Enter admin credentials"
Invoke-Command -ComputerName "Server01" -Credential $cred -ScriptBlock { Get-VM }

# Better: use Windows Credential Manager or a secret manager
# Avoid: $password = "P@ssword123" in scripts
```

**Logging:** Production scripts should log their activity:

```powershell
function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $entry = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$Level] $Message"
    Write-Host $entry
    Add-Content -Path "C:\Logs\HyperV-Automation.log" -Value $entry
}

Write-Log "Starting VM provisioning for: $VMName"
Write-Log "VM created successfully." "SUCCESS"
Write-Log "Failed to connect to remote host." "ERROR"
```

---

> **Key Takeaways**
> - PowerShell 7 is the modern version -- install it alongside Windows PowerShell 5.1; use VS Code not ISE
> - The Hyper-V module works in both versions; most cmdlets also have a `-ComputerName` parameter for remote management
> - Always handle errors with `try/catch` blocks in production scripts
> - Never hard-code credentials in scripts -- use `Get-Credential` or a secret manager
> - `Invoke-Command` and `Enter-PSSession` let you manage any Hyper-V host from a single workstation

---

Automation handles the repetitive work, but large environments need integration -- with identity systems, monitoring platforms, and management tools that span the whole organisation. Chapter 11 covers connecting Hyper-V to what surrounds it: Windows Admin Center for modern management, System Center for enterprise-scale control, and Azure for hybrid workloads and off-site backup.

---

## Exercise 10: Build a VM Management Script

> **Estimated time: 45--60 minutes** (longer if you're new to PowerShell scripting; the concepts from the chapter are prerequisites)

Write a PowerShell script that:
1. Accepts a VM name as a parameter.
2. Checks if the VM exists. If not, create it with sensible defaults (2 GB RAM, 40 GB VHDX, connected to your External switch).
3. If the VM exists, display its current state, uptime, CPU usage, and memory assigned.
4. Creates a checkpoint named "Auto_[date]" if one doesn't already exist with today's date in its name.
5. Logs all actions to a file in `C:\Logs\`.

This script exercises parameter handling, error handling, conditional logic, VM management cmdlets, and file output -- the core skills you'll use in real Hyper-V automation.
