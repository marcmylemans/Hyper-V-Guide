# Technical Fact-Check: Hyper-V Guide

*Reviewed against current Windows Server 2022 / Windows 11 documentation and Hyper-V behaviour as of 2025. Issues are rated: **[WRONG]** (factually incorrect), **[OUTDATED]** (was correct, no longer is), **[INCOMPLETE]** (correct but missing important context), **[FORMATTING]** (not a content error, but a structural problem).*

---

## Preface

No technical errors. Minor improvement: the audience statement ("system administrators, IT professionals, and anyone interested") is too broad to be useful.

---

## Chapter 1: Introduction to Hyper-V

**[INCOMPLETE] Supported editions list is missing Windows Server Core.**
The chapter lists Windows Server Standard and Datacenter, and Windows 10/11 Pro/Enterprise/Education -- but omits that Hyper-V also runs on Windows Server Core (no GUI) installations. Core-only Hyper-V is very common in production.

**[INCOMPLETE] No mention of Windows Server Hyper-V Server (free edition).**
Microsoft used to offer "Hyper-V Server" as a free standalone product. It reached end-of-support in 2022. Worth a footnote acknowledging this for readers who encounter it in the wild.

**[INCOMPLETE] Chapter does not explain the difference between the Hyper-V role (Windows Server) and the Hyper-V feature (Windows 10/11).**
These are enabled differently. On Windows Server: Server Manager -> Add Roles -> Hyper-V. On Windows 10/11: Control Panel -> Turn Windows features on or off -> Hyper-V. Both should be shown.

**[INCOMPLETE] Hardware virtualisation must be enabled in BIOS/UEFI -- but the chapter doesn't name what to look for.**
Readers should know to look for "Intel VT-x" / "AMD-V" in their BIOS, and "Intel EPT" / "AMD RVI" for SLAT. Without these labels they won't know what setting to toggle.

---

## Chapter 2: Setting Up Your First VM

**[INCOMPLETE] "50 GB of storage" presented as a general recommendation.**
This is highly OS-dependent. Windows Server 2022 requires a minimum of 32 GB. Ubuntu Server needs as little as 5-10 GB. Rocky Linux needs around 10 GB. The 50 GB figure is a reasonable starting point for a Windows Desktop OS but should be contextualised, not stated as a universal baseline.

**[INCOMPLETE] Linux Integration Services section is outdated.**
The chapter says Linux guests "may need to install them manually, depending on the distribution." Since Linux kernel 4.x (2015 and later), LIS (Linux Integration Services) are built directly into the mainline kernel. Almost all current Linux distributions include them automatically. Manual installation is now only relevant for very old or obscure distros. This should be updated to reflect modern reality.

**[INCOMPLETE] Generation 2 VMs and Linux compatibility caveat missing.**
The chapter recommends Generation 2 for all modern OS, but does not warn that some Linux distributions -- particularly older CentOS, older Debian, and FreeBSD -- do not support Generation 2. Readers who follow this advice and try to install an unsupported distro on a Gen 2 VM will hit a hard failure with no explanation.

**[INCOMPLETE] No mention of Enhanced Session Mode.**
Enhanced Session Mode (using the VMConnect RDP protocol internally) enables copy-paste, audio redirection, and drive sharing between host and VM. It's a critical quality-of-life feature for daily VM use that the chapter omits entirely.

---

## Chapter 3: Networking in Hyper-V

**[INCOMPLETE] Internal/Private switches have no built-in DHCP.**
The chapter describes Dynamic IP Configuration and says VMs "will receive an IP address from a DHCP server." This is true for External switches (your physical network's DHCP server handles it), but Internal and Private switches have no DHCP server by default. A VM on an Internal or Private switch with DHCP selected will sit with an APIPA address (169.254.x.x) indefinitely unless you either run a DHCP server VM on the same switch or configure static IPs. This is one of the most common points of confusion for beginners.

**[INCOMPLETE] RSC disable command may need adapter name, not switch name.**
The command `Set-VMSwitch -Name vSwitchName -EnableSoftwareRsc $false` is syntactically correct, but RSC is actually a property of the vSwitch object in more recent Windows Server builds. This is fine to include, but readers should be aware this applies per-switch.

**[CORRECT] Three switch types, VLAN configuration, SET teaming** -- all technically accurate.

---

## Chapter 4: Storage in Hyper-V

**[WRONG / INCOMPLETE] Defragmentation advice is incorrect for SSD-backed storage.**
Section 4.5 states "Periodically defragment VHDs to maintain performance." This is only appropriate if the physical host drive is a spinning HDD. If the VHD files are stored on an SSD (which is increasingly common and recommended), defragmenting the VHD is not only unnecessary but can cause excess write cycles on the SSD, reducing its lifespan. The correct recommendation for SSD-backed VHDs is to ensure the host OS is running scheduled TRIM/Optimize operations on the SSD, not to defragment. This should be corrected.

**[OUTDATED] Pass-through disks.**
While pass-through disks are documented, Microsoft has moved away from recommending them in favour of VHDX on high-performance storage. Pass-through disks reduce flexibility (no snapshots, no easy migration). Worth adding a note that VHDX on NVMe or SSD is typically preferred over pass-through in modern deployments.

**[FORMATTING / NUMBERING] No content issue, but note that this chapter's subsections are logically sound.**

---

## Chapter 5: Advanced Hyper-V Features

**[WRONG] DDA (Discrete Device Assignment) command is incomplete.**
The command shown is:
```powershell
Dismount-VMHostAssignableDevice -Force
```
This is missing the mandatory `-LocationPath` parameter. Without it, the command will fail. The correct form is:
```powershell
Dismount-VMHostAssignableDevice -LocationPath $devicePath -Force
```
Where `$devicePath` is obtained first by running `Get-PnpDevice` to find the device, then `Get-VMHostAssignableDevice` or by locating the path via Device Manager. The book's version of this command cannot be run as written.

**[INCOMPLETE] Live Migration "requires shared storage" is no longer universally true.**
The chapter states shared storage is a requirement for Live Migration. This was true before Windows Server 2012. Since then, "Shared Nothing Live Migration" allows migrating a running VM between hosts without any shared storage -- the VM files themselves are transferred as part of the migration. This is a major feature that many readers will actually use (it's useful for home labs and small businesses that can't afford SANs) and it is simply absent from the book.

**[INCOMPLETE] Hyper-V Replica replication intervals.**
The chapter lists 30 seconds, 5 minutes, and 15 minutes as options. This is correct for Windows Server 2012 R2 and later. However, the 30-second interval requires Windows Server 2012 R2 or newer -- it was not available in the original Windows Server 2012 release. A version note here would help readers targeting older environments.

**[INCOMPLETE] Shielded VMs section omits Encryption Supported VMs.**
There are actually two protection levels: "Shielded" (full BitLocker + HGS) and "Encryption Supported" (encrypts the VM but with less strict attestation requirements). Many environments that want encryption without the full HGS infrastructure use the Encryption Supported mode. The chapter only mentions the full Shielded mode.

**[INCOMPLETE] Nested Virtualization has important caveats not mentioned.**
The VM must be powered off before enabling nested virtualisation. Dynamic Memory must be disabled on VMs running nested Hyper-V. Network adapters in the VM must be configured in MAC Address Spoofing mode. None of these are mentioned, and all of them will cause failures if missed.

---

## Chapter 6: Backup and Recovery in Hyper-V

**[FORMATTING] Section 6.6 is missing.**
The chapter goes directly from section 6.5 to section 6.7. There is no 6.6. This appears to be a numbering oversight.

**[OUTDATED] "Altaro VM Backup" is now Hornetsecurity VM Backup.**
Altaro was acquired by Hornetsecurity in January 2022. The product still exists but under the Hornetsecurity brand. Listing "Altaro VM Backup" as a current product recommendation will confuse readers who search for it.

**[INCOMPLETE] Windows Server Backup limitations not disclosed.**
Windows Server Backup works well but has notable limitations: it does not support application-consistent backups of all workloads without VSS writers in the guest, it doesn't support deduplication of backups, and it cannot back up to tape natively (requires third-party). Readers choosing between free tools and paid solutions deserve to know this.

**[INCOMPLETE] No mention of Veeam Backup & Replication Community Edition (free).**
Veeam offers a free Community Edition that supports backing up up to 10 VMs and is widely used in home labs and small businesses. This is the tool many readers will actually end up using, and omitting it while mentioning the commercial version creates a misleading picture.

---

## Chapter 7: Monitoring and Performance Tuning

**[WRONG / INCOMPLETE] Defragmentation advice repeated from Chapter 4.**
Section 7.3 again recommends defragmenting VHDs without the SSD caveat. Same issue as Chapter 4. Should be corrected consistently.

**[OUTDATED] SCOM (System Center Operations Manager) is now Azure Monitor / Azure Arc.**
Microsoft has been progressively moving SCOM workloads to Azure Monitor and Azure Arc-enabled servers. SCOM is still supported and in use at many enterprises, but presenting it as a current monitoring tool without mentioning the Azure-native alternatives gives a dated picture. At minimum, Windows Admin Center's built-in monitoring capabilities should be mentioned as a free alternative.

**[INCOMPLETE] Key performance counters list is useful but incomplete.**
The chapter gives 5 counters but misses some of the most actionable ones: `\Hyper-V Hypervisor Virtual Processor(_Total)\% Guest Run Time` (CPU steal by VMs), `\Hyper-V Dynamic Memory Balancer(*)\Average Pressure` (memory pressure indicator), and `\Hyper-V Virtual Storage Device(*)\Write Operations/Sec`. These are the counters administrators actually alert on.

---

## Chapter 8: High Availability and Failover Clustering

**[OUTDATED] External image hosted on Veeam's website (2014).**
Section 8.3 references `https://www.veeam.com/blog/wp-content/uploads/2014/09/1-adding-the-FC-feature.png`. This is a third-party image from a 2014 blog post. It may break at any time and does not reflect the current Failover Cluster Manager UI. Should be replaced with an original screenshot.

**[INCOMPLETE] No mention of Azure Stack HCI as a modern alternative.**
For readers evaluating HA clustering, Azure Stack HCI is Microsoft's current recommended platform for hyper-converged infrastructure. It uses the same underlying Hyper-V + Failover Clustering technology but with an Azure-managed lifecycle. Not mentioning it leaves the chapter disconnected from where Microsoft is directing enterprise customers.

**[INCOMPLETE] Quorum configuration explained only in one sentence.**
Quorum is one of the most misunderstood concepts in clustering. The chapter says "Ensure the cluster can continue to operate if some nodes fail" without explaining the different quorum modes (Node Majority, Node and Disk Majority, Node and File Share Majority, No Majority: Disk Only) or why getting this wrong causes split-brain scenarios that corrupt clustered storage. This needs more depth.

---

## Chapter 9: Security Best Practices

**[INCOMPLETE] RBAC section is missing Hyper-V-specific delegation.**
The chapter recommends RBAC but does not explain how it actually works in Hyper-V. Hyper-V has specific authorization built into the hypervisor via the "Hyper-V Administrators" local group, and more granular delegation through authorization manager (AzMan) -- though AzMan is largely deprecated. In practice, Hyper-V access is typically controlled through Active Directory group membership and delegation, and the chapter doesn't explain this at the Hyper-V level.

**[INCOMPLETE] No mention of Credential Guard.**
Windows Defender Credential Guard, which uses Hyper-V-based security to isolate credentials, is a major security feature that directly involves Hyper-V. It's notable enough to mention in a security chapter.

**[CORRECT] Secure Boot, Shielded VMs, network ACLs, BitLocker, IPsec** -- all technically accurate.

---

## Chapter 10: Automation and Scripting with PowerShell

**[FORMATTING] Multiple section headers missing spaces.**
`###Creating and Managing Virtual Switches` should be `### Creating and Managing Virtual Switches`. This causes the heading to render as plain text in most Markdown renderers. Affects at least three headers in the chapter.

**[OUTDATED] Windows PowerShell ISE is deprecated.**
Section 10.2 recommends "Windows PowerShell ISE" as a scripting environment. PowerShell ISE has been in maintenance mode since 2017 and Microsoft officially recommends VS Code with the PowerShell extension as its replacement. New readers should be directed to VS Code, not ISE.

**[INCOMPLETE] No distinction between Windows PowerShell (5.1) and PowerShell (7+).**
These are meaningfully different. The Hyper-V module works in Windows PowerShell 5.1 and in PowerShell 7+ (on Windows). However, some cmdlets behave differently, and PowerShell 7 offers much better cross-platform remoting. The chapter treats "PowerShell" as a single thing when there are now two maintained versions.

**[INCOMPLETE] The automated VM creation script has no error handling.**
The script in 10.4 creates a VM, attaches a switch, and starts it -- but if any step fails (switch doesn't exist, path invalid), the script will error out partially and leave things in an inconsistent state. The error-handling section exists in 10.5 but isn't applied to the example script, which undermines the lesson.

**[INCOMPLETE] No mention of the Hyper-V PowerShell module requiring Hyper-V to be installed.**
The chapter says the module "comes pre-installed with Windows Server editions" -- true, but the Hyper-V PowerShell module on a Windows Server that does NOT have the Hyper-V role installed is the RSAT (Remote Server Administration Tools) version, which allows remote management only. The local management cmdlets require the Hyper-V role to be installed. This distinction confuses readers who try to run `New-VM` on a management server without Hyper-V installed.

---

## Chapter 11: Integration with Microsoft Services

**[INCOMPLETE] SCVMM license cost not mentioned.**
System Center VMM is a paid product included in System Center Standard and Datacenter licensing, which costs thousands of dollars per year. Presenting it alongside free tools (Windows Admin Center) without any cost/complexity differentiation is misleading to readers who might assume all these tools are comparable in accessibility.

**[INCOMPLETE] Azure Site Recovery setup steps are significantly simplified.**
The chapter presents ASR setup as a 4-step process that omits the Site Recovery Mobility Agent (required for Hyper-V-to-Azure replication), the Recovery Services vault configuration specifics, and network mapping between on-premises networks and Azure VNets. The steps as written would not produce a working ASR setup.

**[INCOMPLETE] Windows Admin Center should be the leading free tool.**
WAC is free, browser-based, requires no agents, and handles most Hyper-V management tasks that home lab and small business users need. It should be introduced before SCVMM, not after it.

---

## Chapter 12: Troubleshooting and Support

**[INCOMPLETE] Hyper-V log paths are incomplete.**
The chapter only mentions `Hyper-V-Worker` logs. The full set of useful Hyper-V event logs is:
- `Microsoft-Windows-Hyper-V-Worker` (guest operations, VM start/stop)
- `Microsoft-Windows-Hyper-V-VMMS` (Virtual Machine Management Service)
- `Microsoft-Windows-Hyper-V-Hypervisor` (hypervisor events)
- `Microsoft-Windows-Hyper-V-Network` (virtual network events)
- `Microsoft-Windows-Hyper-V-VmSwitch` (virtual switch events)

For most real troubleshooting, VMMS is the most useful log, not Worker.

**[INCOMPLETE] No mention of `Get-VMEvent` PowerShell cmdlet.**
This cmdlet retrieves Hyper-V events programmatically and is extremely useful for scripted monitoring and alerting. It's absent from both the PowerShell chapter and the troubleshooting chapter.

**[INCOMPLETE] IP conflict troubleshooting advice should mention MAC address conflicts.**
In virtual environments, MAC address conflicts (especially when cloning VMs without generating new MACs) are a more common cause of connectivity issues than traditional IP conflicts. This is worth calling out.

---

## Chapter 13: Best Practices and Optimization

**[INCOMPLETE] Power plan recommendation should specify "High Performance."**
Section 13.5 mentions configuring "power plans" but doesn't specify which one. For Hyper-V hosts, Microsoft recommends the "High Performance" power plan (not Balanced) to prevent CPU throttling from degrading VM performance. This is a specific, actionable detail that the vague recommendation misses.

**[INCOMPLETE] No mention of disabling power savings features at the BIOS level.**
Even with the Windows High Performance power plan, some motherboards and server BIOSes have their own C-state and P-state management that can throttle CPUs independently of Windows. This is a common cause of inconsistent VM performance on server hardware.

**No technical errors in the general best practices** -- the advice on resource allocation, Dynamic Memory, regular backups, etc. is all sound.

---

## Glossary

**Missing entries that appear in the body text:**

| Missing Term | Chapter First Used |
|---|---|
| iSCSI | Ch8 |
| DDA (Discrete Device Assignment) | Ch5 |
| S2D / Storage Spaces Direct | Ch5 |
| SET (Switch Embedded Teaming) | Ch3 |
| HGS (Host Guardian Service) | Ch5, Ch9 |
| CAU (Cluster-Aware Updating) | Ch8 |
| NLA (Network Level Authentication) | Ch9 |
| RBAC (Role-Based Access Control) | Ch9 |
| CSV (Cluster Shared Volume) | Ch8 |
| SCVMM | Ch11 |
| ASR (Azure Site Recovery) | Ch11 |
| APIPA | Should be added |
| LIS (Linux Integration Services) | Ch2 |
| VT-x / AMD-V | Ch1 |
| SLAT | Ch1 |
| EPT / NPT | Ch1 |
| Type 1 / Type 2 Hypervisor | Ch1 (implied) |
| Enhanced Session Mode | Should be added |

---

## Summary of Issues by Severity

**Must Fix (factually wrong or will cause reader failures):**
1. DDA `Dismount-VMHostAssignableDevice` command missing `-LocationPath` parameter
2. Defragmentation recommendation for SSD-backed VHDs is harmful advice
3. Section 6.6 numbering is missing
4. Linux Integration Services advice is significantly outdated
5. Chapter 10 heading formatting errors cause rendering failures

**Should Fix (outdated or misleading):**
6. No Shared Nothing Live Migration mentioned -- a key feature
7. "Altaro VM Backup" should be "Hornetsecurity VM Backup"
8. PowerShell ISE deprecated in favour of VS Code
9. Windows PowerShell 5.1 vs PowerShell 7+ not distinguished
10. SCOM presented without Azure Monitor alternatives
11. External Veeam image (2014) in Chapter 8
12. DHCP caveat missing for Internal/Private switches

**Should Add (notable gaps):**
13. Enhanced Session Mode (Chapter 2)
14. Shared Nothing Live Migration (Chapter 5)
15. Veeam Community Edition free tier (Chapter 6)
16. Azure Stack HCI (Chapter 8)
17. Windows Admin Center as the primary free management tool (Chapter 11)
18. Full Hyper-V event log paths (Chapter 12)
19. High Performance power plan recommendation (Chapter 13)
20. Nested Virtualization caveats (Chapter 5)
21. Glossary: 18+ missing entries
