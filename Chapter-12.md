# Chapter 12: Troubleshooting and Support

Something has gone wrong. A VM won't start, the network is down, performance has cratered, or the host is behaving strangely. This chapter covers a systematic approach to diagnosing Hyper-V problems and the specific tools and techniques that make troubleshooting faster and more reliable.

## 12.1 A Systematic Approach to Troubleshooting

The biggest mistake in troubleshooting is jumping straight to solutions without understanding the problem. This leads to making random changes that may fix the immediate symptom while introducing new issues -- or worse, making things worse.

A better approach:

1. **Define the problem precisely.** "The network is broken" is not precise. "VM WebServer01 cannot ping 192.168.1.1 but can ping other VMs on the same switch" is precise. The precise problem statement usually suggests the likely cause.

2. **Establish what changed.** Most problems have a trigger. Something changed: an update was installed, a configuration was modified, a cable was moved, a new VM was created. Find the change.

3. **Check the obvious things first.** Is the VM powered on? Is it connected to the right switch? Is the host itself under resource pressure? Are other VMs affected, or just this one?

4. **Check the logs.** Windows logs almost everything. The answer is usually in Event Viewer.

5. **Make one change at a time.** Each change is a test. If you make three changes simultaneously and the problem resolves, you don't know which change fixed it.

6. **Verify the fix.** Confirm the problem is actually resolved before calling it done.

## 12.2 Hyper-V Event Logs

Event Viewer is your first stop for diagnosing any Hyper-V issue. Hyper-V writes to several specific event logs -- knowing which one to check saves time.

**Navigate to:** Event Viewer > Applications and Services Logs > Microsoft > Windows

| Log Name | What It Contains | Most Useful For |
|---|---|---|
| `Hyper-V-VMMS` (Admin) | Virtual Machine Management Service | VM management failures, start/stop issues, Live Migration problems |
| `Hyper-V-Worker` (Admin) | Per-VM operational events | VM crashes, Integration Services issues, checkpoint failures |
| `Hyper-V-Hypervisor` (Admin) | Low-level hypervisor events | Hypervisor startup issues, very low-level failures |
| `Hyper-V-VmSwitch` (Admin) | Virtual switch events | Network adapter and switch failures |
| `Hyper-V-Network` | Network-related events | Network isolation and VLAN issues |

> **The most useful log for most problems is `Hyper-V-VMMS`.** It logs every significant action the Virtual Machine Management Service takes and why it failed if it didn't succeed.

```powershell
# View the last 50 errors from Hyper-V VMMS
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" -MaxEvents 50 |
    Where-Object { $_.LevelDisplayName -eq "Error" } |
    Select-Object TimeCreated, Id, Message |
    Format-List

# View all recent Hyper-V events (errors and warnings)
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" -MaxEvents 100 |
    Where-Object { $_.Level -le 3 } |
    Select-Object TimeCreated, LevelDisplayName, Message

# Get Hyper-V events related to a specific VM
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Hyper-V-VMMS-Admin"
    StartTime = (Get-Date).AddHours(-24)
} | Where-Object { $_.Message -like "*WebServer01*" }
```

## 12.3 Common Problems and Solutions

When something breaks, most Hyper-V problems fall into four categories. Start here to find your section:

```
Problem?
|
+-- VM won't start ---------> Section 12.3.1 (VM Won't Start)
|     What's the error?
|     - "Not enough memory"  --> reduce RAM on other VMs or enable Dynamic Memory
|     - "Hypervisor not running" --> bcdedit /set hypervisorlaunchtype auto, reboot
|     - "Cannot find VHDX"  --> update path in VM Settings
|     - Other error         --> check Event Viewer > Hyper-V-VMMS
|
+-- VM is slow -------------> Section 12.3.2 (Performance Problems)
|     Check in order:
|     CPU first (% Total Run Time > 85%) --> reduce vCPU count or move VMs
|     then RAM (Available MBytes < 2 GB) --> enable Dynamic Memory or add RAM
|     then Disk (Queue Length > 2)       --> check VHDX fragmentation, move to SSD
|
+-- Network problem --------> Section 12.3.3 (Network Issues)
|     - No IP address       --> check vSwitch type, DHCP scope, VLAN ID
|     - Can't reach host    --> check "Allow management OS" on External switch
|     - VM-to-VM broken     --> confirm both VMs on same vSwitch
|     - Intermittent drops  --> check for MAC address conflict (duplicate IDs)
|
+-- Storage issue ----------> Section 12.3.4 (Storage Problems)
      - Slow I/O            --> check disk queue, fragmentation, VHDX placement
      - VHDX corruption     --> Test-VHD, then Repair-VHD for minor issues
      - Out of space        --> Resize-VHD, then extend volume inside VM
```

### VM Won't Start

**Error: "Not enough memory in the system to start the virtual machine."**

The host doesn't have enough free RAM.

```powershell
# Check host available memory
(Get-WmiObject -Class Win32_OperatingSystem).FreePhysicalMemory / 1MB  # MB free

# Check memory demand of all VMs
Get-VM | Select-Object Name, State, MemoryDemand, MemoryAssigned
```

Fix: Reduce startup RAM on other VMs, or enable Dynamic Memory (which allows VMs to return unused RAM to the host), or shut down VMs you don't currently need.

**Error: "The virtual machine could not be started because the hypervisor is not running."**

Hyper-V isn't enabled, or there's a boot configuration problem.

```powershell
# Check hypervisor auto-start setting (should be "autoinitialtype=auto")
bcdedit /enum | findstr /i hypervisor

# Fix it
bcdedit /set hypervisorlaunchtype auto
```

Then restart the host.

**Error: Cannot find the VHDX file**

The VM's VHDX file has been moved, deleted, or the path has changed.

```powershell
# Check what disk path a VM has configured
(Get-VMHardDiskDrive -VMName "WebServer01").Path
```

Fix: If the VHDX was moved, update the path in VM Settings > SCSI Controller > Hard Drive. If it was deleted, restore from backup.

### VM Performance Issues

VM performance problems in Hyper-V almost always trace to one of four resources. Rule them out in order: CPU, then RAM, then disk, then network — the answer is almost always in that sequence.

**Slow VM despite adequate resources:**

Start by ruling out things outside the VM's control.

```powershell
# Check host CPU usage
(Get-VMHost).LogicalProcessorCount
Get-Counter -Counter "\Hyper-V Hypervisor Logical Processor(_Total)\% Total Run Time"

# Check host available memory
Get-Counter -Counter "\Memory\Available MBytes"

# Check disk queue (above 2 = disk bottleneck)
Get-Counter -Counter "\PhysicalDisk(_Total)\Avg. Disk Queue Length"
```

If host resources look fine, the problem is likely inside the VM:
- Open Task Manager inside the VM to find which process is consuming resources.
- Check if antivirus is running a scan.
- Check if Windows Update is processing updates in the background.

**VM has very high CPU usage on the host, but looks idle inside the VM:**

This can indicate a tight processor loop inside the VM that Windows isn't attributing to any visible process, or Integration Services problems.

```powershell
# Check per-VM CPU usage
Get-VM | Select-Object Name, CPUUsage | Sort-Object CPUUsage -Descending
```

If Integration Services are outdated, the VM may be generating excessive CPU overhead. Verify Integration Services are up to date inside the VM.

### Network Connectivity Issues

Network problems in Hyper-V almost always trace to one of four causes: wrong switch type, missing DHCP, VLAN misconfiguration, or a MAC address conflict. Work through these systematically before looking elsewhere.

**VM has no IP address (on an Internal or Private switch):**

Expected -- Internal/Private switches have no DHCP. Assign a static IP inside the VM (see Chapter 3, section 3.6).

**VM has an APIPA address (169.254.x.x):**

DHCP is configured but no DHCP server responded. If on an External switch, verify your physical network's DHCP server is running. If on an Internal switch, you need to set up a DHCP server or use static IPs.

**VM connected to External switch has no internet, but physical hosts do:**

```powershell
# Verify the External switch is bound to the right physical adapter
Get-VMSwitch -Name "External-LAN" | Select-Object Name, NetAdapterInterfaceDescription, SwitchType
```

If the wrong adapter is listed, or the adapter is disconnected, that's the cause. Also check if the host itself can ping the default gateway -- if not, the problem is at the physical network level, not Hyper-V.

**Two VMs can't communicate with each other:**

```powershell
# Verify both VMs are connected to the same switch
Get-VM | ForEach-Object {
    Get-VMNetworkAdapter -VMName $_.Name | Select-Object VMName, SwitchName, MacAddress
}
```

If they're on the same switch, check VLAN settings (a VLAN mismatch silently blocks communication) and firewall rules inside each VM.

**MAC address conflict causing connectivity issues:**

When you clone VMs, the copy may retain the same MAC address. MAC conflicts cause intermittent and confusing network problems.

```powershell
# Check MAC addresses of all VMs (look for duplicates)
Get-VM | ForEach-Object {
    Get-VMNetworkAdapter -VMName $_.Name
} | Select-Object VMName, MacAddress | Sort-Object MacAddress
```

Fix: Set the network adapter to use a dynamically assigned MAC address in VM Settings > Network Adapter > Advanced Features.

### Storage Issues

Storage problems usually fall into two distinct categories: performance issues (slow I/O that degrades VM responsiveness) and integrity issues (corrupted or missing VHDX files that prevent VMs from starting or running at all).

**VM is running slowly and disk queue length on the host is high:**

The physical storage is the bottleneck. Options:
- Move the VHDX files to faster storage (NVMe/SSD if currently on HDD).
- Convert dynamic VHDXs to fixed-size (eliminates expansion overhead).
- Distribute VMs across multiple physical drives to reduce I/O contention.

**VHDX file cannot be found / corrupted:**

```powershell
# Test a VHDX file for corruption
Test-VHD -Path "D:\VMs\WebServer01\WebServer01.vhdx"

# Attempt to repair a VHDX
Repair-VHD -Path "D:\VMs\WebServer01\WebServer01.vhdx"
```

If the VHDX is unrecoverably corrupted, restore from backup (this is why you have backups).

## 12.4 Performance Monitor for Troubleshooting

When the problem is intermittent or you need data over time rather than a snapshot:

```powershell
# Capture performance data to a file for 1 hour (sample every 30 seconds)
$outputPath = "C:\PerfData\HyperV_$(Get-Date -Format 'yyyyMMdd_HHmm')"
logman create counter "HyperV-Capture" -o $outputPath -f csv `
    -c "\Hyper-V Hypervisor Logical Processor(_Total)\% Total Run Time" `
       "\Memory\Available MBytes" `
       "\PhysicalDisk(_Total)\Avg. Disk Queue Length" `
       "\Network Interface(*)\Bytes Total/sec" `
    -si 30 -cnf 01:00:00

logman start "HyperV-Capture"
# Wait an hour (or until the problem reproduces)
logman stop "HyperV-Capture"
```

Open the resulting CSV in Excel or import it into PowerShell for analysis.

## 12.5 Useful Diagnostic Commands

```powershell
# Full Hyper-V host configuration summary
Get-VMHost | Format-List *

# Get all VMs with their current state and resource usage
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime

# Check all network adapters for all VMs
Get-VM | ForEach-Object { Get-VMNetworkAdapter -VMName $_.Name } |
    Select-Object VMName, SwitchName, MacAddress, IPAddresses

# List all checkpoints (look for old ones consuming disk space)
Get-VMSnapshot -VMName * | Select-Object VMName, Name, CreationTime, ParentCheckpointName

# Check Integration Services status for all VMs
Get-VMIntegrationService -VMName * | Where-Object {$_.Enabled -eq $false}

# Get a full cluster event log (if using clustering)
Get-ClusterLog -Destination "C:\Logs" -UseLocalTime

# Export complete VM configuration (useful before making changes)
Export-VM -Name "WebServer01" -Path "D:\VMExports"
```

## 12.6 Support Resources

When you've exhausted local troubleshooting:

**Microsoft Documentation:**
- [Hyper-V on Windows Server](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/hyper-v-on-windows-server)
- [Hyper-V on Windows 10/11](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/)
- [Troubleshoot Hyper-V](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/troubleshoot/)

**Community:**
- [Microsoft Tech Community - Hyper-V](https://techcommunity.microsoft.com/t5/hyper-v/bd-p/Hyper-V)
- [r/HyperV on Reddit](https://www.reddit.com/r/HyperV/)
- [Spiceworks Community](https://community.spiceworks.com/)

**Support:**
- [Microsoft Support](https://support.microsoft.com) for paid support cases
- Check your Windows Server licensing -- active Software Assurance subscribers get additional support benefits.

---

> **Key Takeaways**
> - Define the problem precisely before looking for solutions -- "VM is slow" is not a problem statement
> - The most useful Hyper-V event log is `Hyper-V-VMMS`, not `Hyper-V-Worker`
> - Most VM startup failures are resource exhaustion (RAM), path problems (VHDX moved), or hypervisor not running
> - Internal/Private switch VMs with no IP: expected -- they have no DHCP server
> - MAC address conflicts from cloned VMs cause confusing, intermittent network problems
> - Use `Test-VHD` to check VHDX integrity; `Repair-VHD` for minor corruption

---

You now have the tools to diagnose and resolve most Hyper-V problems systematically. Chapter 13 brings everything together: the best practices and optimisation checklist that separates a well-run Hyper-V environment from one that works until it doesn't.

---

## Exercise 12: Fault Diagnosis Practice

> **Estimated time: 30--45 minutes**

Deliberately break things and practice finding and fixing them:

1. Create a VM and note its VHDX path.
2. Power off the VM. Rename its VHDX file on the host (simulating an accidental rename).
3. Try to start the VM. Read the error message. Find the specific error in Event Viewer > Hyper-V-VMMS.
4. Fix it by updating the path in VM Settings.
5. Start the VM. Now disconnect its network adapter (right-click > Settings > Network Adapter > Not Connected).
6. Inside the VM, run `ipconfig` and note the result. Fix the network adapter.
7. Create a checkpoint, write a test file to the desktop inside the VM, then apply the checkpoint.
8. Verify the test file is gone. This confirms your checkpoint revert is working.
