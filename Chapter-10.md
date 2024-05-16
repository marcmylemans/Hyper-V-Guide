# Chapter 10: Automation and Scripting with PowerShell

## 10.1 Introduction to PowerShell

PowerShell is a powerful scripting language and command-line shell developed by Microsoft. It provides a platform for automating administrative tasks and managing Hyper-V environments efficiently.

## 10.2 PowerShell Basics

### Getting Started

To use PowerShell for managing Hyper-V, you need to have the Hyper-V module installed. This module comes pre-installed with Windows Server editions and can be added to Windows 10/11 Pro, Enterprise, and Education editions.

### Basic Commands

- **Opening PowerShell:**
  - Open the Start menu, type `PowerShell`, and select **Windows PowerShell** or **Windows PowerShell ISE**.

- **Running Commands:**
  - Commands are executed by typing them at the PowerShell prompt and pressing Enter.

### Common Cmdlets

- **Get-Help:** Provides help information for cmdlets.
  ```powershell
  Get-Help <CmdletName>
  ```
- **Get-Command:** Lists all available cmdlets.
  ```powershell
  Get-Command
  ```
- **Get-Module:** Lists all installed modules.
  ```powershell
  Get-Module
  ```
## 10.3 Managing Hyper-V with PowerShell

### Importing the Hyper-V Module

Before running Hyper-V cmdlets, ensure the Hyper-V module is imported.

  ```powershell
  Import-Module Hyper-V
  ```

### Creating and Managing VMs
- Create a New VM:

```powershell
New-VM -Name "TestVM" -MemoryStartupBytes 2GB -NewVHDPath "C:\VMs\TestVM\TestVM.vhdx" -NewVHDSizeBytes 50GB -Generation 2
```

- Start a VM:

```powershell
Start-VM -Name "TestVM"
```
- Stop a VM:

```powershell
Stop-VM -Name "TestVM"
```
- Remove a VM:

```powershell
Remove-VM -Name "TestVM" -Force
```
### Configuring VM Settings
- Add a Network Adapter:

```powershell
Add-VMNetworkAdapter -VMName "TestVM" -SwitchName "ExternalSwitch"
```
- Change VM Memory:

```powershell
Set-VMMemory -VMName "TestVM" -DynamicMemoryEnabled $true -MinimumBytes 1GB -MaximumBytes 4GB -Buffer 20
```
- Add a Virtual Hard Disk:

```powershell
Add-VMHardDiskDrive -VMName "TestVM" -Path "C:\VMs\TestVM\AdditionalDisk.vhdx"
```
###Creating and Managing Virtual Switches

- Create an External Virtual Switch:

```powershell
New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true
```
- Create an Internal Virtual Switch:

```powershell
New-VMSwitch -Name "InternalSwitch" -SwitchType Internal
```

- Create a Private Virtual Switch:

```powershell
New-VMSwitch -Name "PrivateSwitch" -SwitchType Private
```

## 10.4 Advanced Scripting with PowerShell

### Automating Tasks with Scripts

PowerShell scripts can automate repetitive tasks, saving time and reducing the risk of errors.

- **Script Example:** Automated VM Creation
  ```powershell
  $vmName = "TestVM"
  $vhdPath = "C:\VMs\$vmName\$vmName.vhdx"
  $switchName = "ExternalSwitch"

  New-VM -Name $vmName -MemoryStartupBytes 2GB -NewVHDPath $vhdPath -NewVHDSizeBytes 50GB -Generation 2
  Add-VMNetworkAdapter -VMName $vmName -SwitchName $switchName
  Start-VM -Name $vmName
  ```
### Scheduling PowerShell Scripts
Use Task Scheduler to run PowerShell scripts at specified times.

1. **Open Task Scheduler:** Type Task Scheduler in the Start menu and open it.
2. **Create a New Task:** Click Create Task in the Actions pane.
3. **Configure the Task:**
  - Name the task and configure the trigger (e.g., daily, weekly).
  - In the Actions tab, select Start a program and enter powershell.exe with the script path as an argument.
    ```
    powershell.exe -File "C:\Scripts\AutomateVM.ps1"
    ```
 4. **Save the Task:** Complete the wizard and save the task.

### Using PowerShell Remoting

PowerShell Remoting allows you to manage Hyper-V hosts remotely.

- **Enable Remoting:**
  ```powershell
  Enable-PSRemoting -Force
  ```
- **Execute Remote Commands:**
  ```powershell
  Invoke-Command -ComputerName "RemoteHost" -ScriptBlock { Get-VM }
  ```

## 10.5 Best Practices for Automation

### Script Documentation

- **Commenting:** Include comments in your scripts to explain what each part does.
- **Logging:** Implement logging to track script execution and errors
  ```powershell
  # This script creates a new VM and configures its network settings
  ```
### Error Handling 
 
- Try-Catch Blocks: Use try-catch blocks to handle errors gracefully.
  ```powershell
  try {
    Start-VM -Name "TestVM"
  } catch {
      Write-Error "Failed to start VM: $_"
  }
  ```

### Security Considerations
  - **Secure Credentials:** Use secure methods to handle credentials, such as Get-Credential cmdlet.
  ```powershell
  $credential = Get-Credential
  ```
  - **Script Execution Policy:** Set appropriate script execution policies.
  ```powershell
  Set-ExecutionPolicy RemoteSigned
  ```
