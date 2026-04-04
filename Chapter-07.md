# Chapter 7: Monitoring and Performance Tuning

> **Part 2 begins here.** Chapters 7 onward cover topics that go deeper into production Hyper-V environments. Most of this chapter is relevant whether you're running a single host or a cluster -- good monitoring habits apply everywhere.

Running virtual machines without monitoring them is like driving without looking at the dashboard. Things might be fine. Or you might be about to run out of fuel (memory), the engine might be overheating (CPU), or there might be a brake problem you haven't noticed yet (disk I/O). This chapter covers what to watch, what the numbers mean, and what to do when they look wrong.

## 7.1 What to Monitor

There are four resource dimensions that matter for Hyper-V hosts:

**CPU:** Is the host processing everything VMs demand, or is it CPU-bound? Are VMs waiting for CPU time?

**Memory:** Is there enough physical RAM for all VMs to run comfortably? Is the host swapping (a disaster scenario for performance)?

**Disk I/O:** Are disk reads and writes fast enough? Are VMs waiting for storage?

**Network:** Are any VMs saturating their network adapters? Is there unusual traffic?

## 7.2 Hyper-V Manager: Basic Monitoring

Hyper-V Manager shows real-time status for each VM -- whether it's running, paused, or saved, and basic resource usage. It's the first place to check when something seems wrong.

For more granular data, Hyper-V Manager shows per-VM statistics if you enable them. Right-click a running VM > **Settings > Management > Resource Metering**. Enable resource metering, then use PowerShell to query the data:

```powershell
# Enable resource metering for a VM
Enable-VMResourceMetering -VMName "MyVM"

# Get current resource usage
Measure-VM -Name "MyVM"
```

This gives you average CPU usage, peak memory demand, total disk I/O, and total network I/O over the metering period.

## 7.3 Performance Monitor: Detailed Analysis

Performance Monitor (`perfmon.exe`) is the deep-dive tool for Hyper-V performance. It records any counter at any interval and creates graphs, reports, and alerts.

You won't need to watch all of these counters every day. Most of the time, Windows Admin Center gives you enough visibility. But knowing what the counter names mean -- and what thresholds to care about -- means you'll recognise trouble when a monitoring tool flags it, and you'll know what to do next.

**Key counters for Hyper-V hosts:**

| Counter | What It Tells You | Concern If... |
|---|---|---|
| `\Hyper-V Hypervisor Logical Processor(_Total)\% Total Run Time` | Total CPU load across all VMs | Consistently over 85% |
| `\Hyper-V Hypervisor Virtual Processor(_Total)\% Guest Run Time` | CPU used by VM workloads | Consistently over 80% |
| `\Hyper-V Hypervisor Root Virtual Processor(_Total)\% Guest Run Time` | CPU used by the host itself | Consistently over 20% |
| `\Memory\Available MBytes` | Free RAM on the host | Falls below 2 GB |
| `\Hyper-V Dynamic Memory Balancer(*)\Average Pressure` | Memory pressure index | Sustained above 100 |
| `\PhysicalDisk(_Total)\Avg. Disk Queue Length` | Storage bottleneck | Consistently over 2 |
| `\Hyper-V Virtual Storage Device(*)\Write Operations/Sec` | Write I/O per VM disk | Spikes + high queue length |
| `\Network Interface(_Total)\Bytes Total/sec` | Network throughput | Near physical NIC capacity |

**To add these counters to Performance Monitor:**

1. Open **Performance Monitor** (search in Start menu).
2. Click the **+** button to add counters.
3. Find the counter category in the list (e.g., "Hyper-V Hypervisor Logical Processor"), select the counter, and click **Add**.
4. Click **OK** to start graphing.

For sustained monitoring (not just real-time), create a **Data Collector Set**:

1. In Performance Monitor, expand **Data Collector Sets > User Defined**.
2. Right-click > **New > Data Collector Set**.
3. Choose "Create from template" and select a template, or "Create manually" to add specific counters.
4. Set the collection interval (every 60 seconds is a good starting point).
5. Schedule when to collect, and where to save the data.

## 7.4 Windows Admin Center: Modern Dashboard

Windows Admin Center (WAC) provides a browser-based monitoring dashboard that's easier to use than Performance Monitor for day-to-day monitoring. It's free to download and install.

In WAC, navigate to a Hyper-V host and you'll see CPU, memory, disk, and network graphs with a clean interface. You can also manage VMs, virtual switches, and storage directly from the same console.

> **WAC should be your first choice for regular monitoring.** Performance Monitor is the right tool when you need granular historical data, custom alerts, or data collection for analysis.

Download WAC from [microsoft.com/en-us/windows-server/windows-admin-center](https://www.microsoft.com/en-us/windows-server/windows-admin-center).

## 7.5 Performance Tuning: CPU

**Over-subscription:** Hyper-V allows you to allocate more virtual CPUs (vCPUs) to VMs than the host has physical cores. A limited amount of over-subscription is fine -- if not all VMs are CPU-active simultaneously, they share the physical cores efficiently. Heavy over-subscription (more than 4:1 vCPU:pCPU ratio on busy hosts) causes visible performance degradation.

**Virtual processor weight:** You can give critical VMs priority access to CPU time by adjusting their virtual processor weight. In VM Settings > Processor, the "Virtual machine reserve" slider gives a VM a guaranteed percentage of the host CPU.

**Disable CPU power saving on the host:** Windows' "Balanced" power plan can throttle CPU speeds to save power. On a Hyper-V host, this causes intermittent performance degradation. Set the host's power plan to **High Performance**:

```powershell
# Set power plan to High Performance
powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
```

> **Check your BIOS too:** Some motherboard and server BIOSes have their own CPU power management (C-states, P-states) that can throttle performance independently of Windows. In the BIOS, look for settings like "CPU C-State Control" or "Processor Power Management" and disable aggressive power saving for Hyper-V hosts.

## 7.6 Performance Tuning: Memory

**Dynamic Memory configuration:** Set startup RAM to what the VM typically needs at idle. Set minimum RAM just low enough that the host can reclaim memory from idle VMs. Set maximum RAM high enough that the VM never hits its ceiling during peak load. Leave a memory buffer of 20-25% to account for rapid demand spikes.

**Watch for memory pressure:** The `\Hyper-V Dynamic Memory Balancer(*)\Average Pressure` counter tells you if VMs are competing for RAM. A value above 100 means the Dynamic Memory balancer cannot satisfy all VM requests. A value consistently above 100 means you need more RAM or fewer VMs.

**Never let the host swap:** If the host runs out of physical RAM and starts paging to disk, VM performance collapses. As a hard rule, keep at least 2-4 GB of physical RAM free for the host OS at all times. If you're hitting this limit, add RAM or reduce the number of VMs.

## 7.7 Performance Tuning: Storage

**Use the right storage type:** NVMe SSDs for active VMs, SATA SSDs as a minimum, spinning HDDs only for archive/cold storage. The storage hierarchy has a larger impact on VM performance than almost any other factor.

**Use fixed-size VHDXs for I/O-intensive workloads:** Dynamic VHDXs have overhead as they expand. For databases, file servers, or anything doing high write I/O, use fixed-size.

**Store VHDs and configuration separately if possible:** If you have two physical drives, store VM config files on one and VHDX files on the other. Reduces I/O contention.

**Avoid storing VHDX files on the same drive as the host OS:** The host OS constantly reads and writes system files. Sharing a drive with VM storage creates I/O contention that degrades both.

**Defragmentation vs TRIM:** See Chapter 4, section 4.6. The short version: defragment VHDX files on HDDs; do not defragment VHDX files on SSDs (use TRIM instead).

## 7.8 Performance Tuning: Network

**Use synthetic network adapters:** Generation 2 VMs always use synthetic adapters. Generation 1 VMs also support "Legacy Network Adapter" -- a slower, emulated adapter. If you're using a Legacy Network Adapter, switch to a regular (synthetic) Network Adapter.

**Dedicated NICs for different traffic types:** On servers with multiple physical NICs, use separate NICs for management traffic, VM traffic, live migration traffic, and storage (iSCSI/SMB). Mixing all traffic on one NIC creates contention and makes troubleshooting harder.

**Jumbo frames:** If your physical network supports MTU 9000 (jumbo frames), enabling jumbo frames on your virtual switches and physical NICs improves storage (iSCSI, SMB) throughput by reducing packet overhead.

## 7.9 Troubleshooting Performance Issues

**VM is running slow:**
1. Check the VM's CPU and memory usage from inside the VM (Task Manager).
2. Check the *host's* CPU and memory usage (Performance Monitor or WAC).
3. Check the disk queue length on the host. If it's above 2 consistently, storage is the bottleneck.
4. Check if other VMs are heavily utilising resources at the same time.

**Host is unresponsive:**
1. If possible, connect to the host via its management IP (not through a VM).
2. Check available memory on the host first -- memory exhaustion is the most common cause of host unresponsiveness.
3. Review Event Viewer > Windows Logs > System for critical errors.

**Event Viewer for Hyper-V:**
Navigate to: `Applications and Services Logs > Microsoft > Windows` and check these specific logs:

- `Hyper-V-Worker`: Events related to individual VM operations (starts, stops, failures).
- `Hyper-V-VMMS`: Virtual Machine Management Service. **Most useful for diagnosing VM management failures.**
- `Hyper-V-Hypervisor`: Low-level hypervisor events.
- `Hyper-V-VmSwitch`: Virtual network switch events.

```powershell
# Get recent Hyper-V errors via PowerShell
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" `
    -MaxEvents 50 | Where-Object {$_.LevelDisplayName -eq "Error"}
```

---

> **Key Takeaways**
> - Set the host power plan to High Performance -- Balanced power saving throttles CPU and causes VM performance issues
> - Check C-states in your server BIOS too -- Windows settings alone may not be enough
> - Key metrics to watch: CPU run time, memory pressure, disk queue length, network saturation
> - Windows Admin Center is the modern, free monitoring dashboard; Performance Monitor is the deep-analysis tool
> - The most important Hyper-V log for troubleshooting is Hyper-V-VMMS (not Hyper-V-Worker)
> - Never let the host run out of physical RAM -- VM performance collapses if the host swaps

---

Performance tuning keeps individual hosts healthy, but it doesn't protect you from the hardware dying. For that, you need redundancy -- multiple physical hosts that can absorb each other's workloads if one fails. Chapter 8 covers Failover Clustering: the technology that keeps VMs running even when a physical host goes down.

---

## Exercise 7: Build a Monitoring Baseline

> **Estimated time: 20--30 minutes**

1. On your Hyper-V host, open Performance Monitor.
2. Add these four counters: `% Total Run Time` (Hyper-V Hypervisor Logical Processor), `Available MBytes` (Memory), `Avg. Disk Queue Length` (PhysicalDisk _Total), and `Bytes Total/sec` (Network Interface _Total).
3. Run your VMs for 15 minutes under normal load. Note the baseline values.
4. Start all your VMs simultaneously and run something CPU-intensive inside each one (e.g., a compression task, a video transcode). Watch the counters.
5. Identify which resource (CPU, RAM, disk, or network) becomes constrained first on your hardware.
6. This is your bottleneck -- the most constrained resource determines your VM density limit.
