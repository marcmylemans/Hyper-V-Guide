# Chapter 6: Backup and Recovery in Hyper-V

Here's the honest truth about backups: everyone knows they need them, most people under-invest in them, and the ones who've experienced a serious data loss event become evangelists about them. This chapter covers the backup strategies that actually work, the tools available at various price points (including free ones), and how to restore a VM when things go wrong.

## 6.1 Why VM Backups Are Different from File Backups

Backing up a virtual machine isn't the same as backing up files. When a VM is running, its disk files are open and actively being written to. Simply copying a VHDX file while the VM is running will likely produce a corrupted backup -- the file system inside the VM may be in an inconsistent state mid-write.

Proper VM backups use one of two approaches:

**VSS (Volume Shadow Copy Service):** Windows creates a point-in-time snapshot of the VM's disk at the storage level, then backs up from that snapshot. The VM keeps running throughout. VSS requires the VM to have Integration Services installed so the guest OS can participate in the snapshot process (flushing write buffers, quiescing applications).

**Saved-state backup:** The hypervisor pauses the VM momentarily, saves its state, takes a snapshot, then resumes the VM. Fast, but causes a brief pause in VM operation. Used when VSS isn't available (for Linux VMs without VSS-aware tools).

Hyper-V's built-in backup tools use VSS by default, providing application-consistent backups with no VM downtime.

## 6.2 Backup Strategies

Before picking a tool, decide on your strategy. Three types of backup are relevant here:

**Full Backup**
Backs up the entire VM -- all disk data, current configuration, and state. Recovery is simple: restore the full backup and you're done. But full backups take the most time and storage. Running a full backup of a 200 GB VM every night consumes 1.4 TB per week.

**Incremental Backup**
Backs up only the data that changed since the last backup (whether that was a full or a previous incremental). The fastest and most storage-efficient option. The trade-off: to restore, you need the full backup *plus* every incremental since then, applied in order. More restore steps means more time.

**Differential Backup**
Backs up everything that changed since the last *full* backup. Faster than full, slower than incremental. To restore: full backup + one differential. Simpler than incremental restoration, but differentials grow over time as more data changes.

**Practical recommendation for most environments:** Weekly full backup + daily incrementals. This balances storage efficiency (small daily backups) against restore simplicity (never more than 7 incrementals to apply).

## 6.3 The 3-2-1 Rule

Before choosing tools, apply this principle to your backup design:

- **3** copies of your data (the live VM + 2 backups)
- **2** different storage media (e.g., local disk + NAS, or local disk + cloud)
- **1** copy offsite (different physical location)

A backup stored on the same physical machine as the VM you're protecting is not a real backup. A fire, flood, theft, or drive failure takes out both simultaneously.

## 6.4 Backup Tools

### Windows Server Backup (Free, Built-In)

Windows Server Backup is included with Windows Server at no extra cost. It handles full VM backups and supports scheduling. For straightforward backup needs, it's a capable starting point.

**How to install:**
```powershell
Install-WindowsFeature -Name Windows-Server-Backup
```

**Creating a backup schedule:**

1. Open **Server Manager > Tools > Windows Server Backup**.
2. Click **Backup Schedule** in the right pane.
3. Choose **Custom** to select specific VMs or **Full server** for everything.
4. Set the schedule (daily, twice-daily, etc.).
5. Choose a destination: a local drive, a network share, or a remote shared folder.
6. Complete the wizard.

**Known limitations of Windows Server Backup:**
- Does not support incremental backups of Hyper-V VMs -- it performs full backups each time (though VSS technology optimises some of this at the block level).
- No deduplication of backup data.
- No backup reporting dashboard -- you need to check Event Viewer for success/failure.
- Cannot back up to tape natively.
- Does not support application-consistent backups for all workloads -- depends on VSS writers in the guest being correctly configured.

For simple environments (a handful of VMs, limited backup storage), Windows Server Backup is fine. For anything more complex, evaluate the paid options.

### Veeam Backup & Replication -- Community Edition (Free)

Veeam's Community Edition is the most widely used free VM backup tool for Hyper-V. It supports:

- Up to **10 VMs** per installation
- Full, incremental, and synthetic full backups
- Application-aware backups (SQL Server, Exchange, Active Directory)
- Backup to local disk, NAS, or cloud
- Email notifications and a proper reporting dashboard
- Instant VM Recovery (boot a VM directly from the backup file while a full restore happens in the background)

For home labs and small businesses, Veeam Community Edition frequently outperforms Windows Server Backup in flexibility and ease of use. Download from [veeam.com](https://www.veeam.com/virtual-machine-backup-solution-free.html).

### Hornetsecurity VM Backup (formerly Altaro VM Backup)

A solid commercial option with a free tier supporting up to 2 VMs. Acquired by Hornetsecurity in 2022 (note: older guides may still refer to this as "Altaro VM Backup" -- it's the same product). Features include augmented inline deduplication, cloud backup to Azure/AWS, and instant boot.

> **If you see references to "Altaro VM Backup" in older documentation:** This product was rebranded to Hornetsecurity VM Backup in 2022. The product still exists and is actively developed.

### System Center Data Protection Manager (Paid -- Enterprise)

Microsoft's enterprise backup solution. Full integration with Active Directory, SQL Server backup agents, tape support, and centralised management. Significant cost and complexity. Primarily relevant for large enterprise environments already invested in the System Center stack.

## 6.5 Configuring a Backup with Windows Server Backup

Here's a complete example: backing up a specific VM to a network share.

```powershell
# Install Windows Server Backup if not already installed
Install-WindowsFeature -Name Windows-Server-Backup

# Backup a specific VM to a network share (one-time backup)
$policy = New-WBPolicy
$vmToBackup = Get-WBVirtualMachine | Where-Object { $_.VmName -eq "MyVM" }
Add-WBVirtualMachine -Policy $policy -VirtualMachine $vmToBackup
$backupLocation = New-WBBackupTarget -NetworkPath "\\BackupServer\VmBackups" `
                                     -Credential (Get-Credential)
Add-WBBackupTarget -Policy $policy -Target $backupLocation
Start-WBBackup -Policy $policy
```

For a scheduled backup, use the GUI wizard in Windows Server Backup -- it handles the scheduling service registration more cleanly than the PowerShell cmdlets.

## 6.6 Restoring a VM

### Restore Using Windows Server Backup

1. Open **Server Manager > Tools > Windows Server Backup**.
2. Click **Recover** in the right pane.
3. Select where the backup is stored (this server, another server, or a network location).
4. Select the backup date to restore from.
5. Select **Hyper-V** as the recovery type.
6. Select the VM(s) to recover.
7. Choose whether to restore to the original location or an alternate location.
8. Complete the wizard.

### Restore Using a Checkpoint (Quick Recovery)

If you need to roll back a VM to a recent state and a checkpoint exists:

1. In Hyper-V Manager, select the VM. The checkpoint list appears at the bottom.
2. Right-click the desired checkpoint > **Apply**.
3. Choose **Create Checkpoint and Apply** (to preserve the current state as a new checkpoint before reverting) or **Apply** (to discard the current state entirely).

Checkpoints are fast but not a substitute for backups. They live on the same physical storage as the VM and don't protect against disk failure.

## 6.7 Best Practices for Backup and Recovery

**Test your restores.** A backup that has never been successfully restored is not a backup you can trust. Schedule a test restore quarterly. Actually boot the restored VM and verify its data. Many backup failures are only discovered during a real recovery emergency.

**Monitor backup jobs.** Configure email alerts for backup failures. Windows Server Backup writes to Event Viewer -- check Application and Services Logs > Windows Backup for success/failure entries. Veeam has a built-in notification system.

**Encrypt your backups.** If your backup media is stolen or a network share is compromised, unencrypted backups expose all of your VM data. Enable encryption in your backup tool, or at least store backups on a BitLocker-encrypted drive.

**Document your recovery procedure.** In a real emergency, you won't have time to figure out how to restore. Write down the exact steps for your environment, store that document somewhere accessible (not only on the Hyper-V host), and include: where backups are stored, credentials needed to access them, and the step-by-step restore process.

**Follow the 3-2-1 rule.** As described in section 6.3: three copies, two media types, one offsite. At minimum, replicate your backups to a NAS, external drive rotated offsite, or cloud storage.

---

> **Key Takeaways**
> - Don't copy VHDX files directly for backup -- VSS-based tools create consistent, application-aware backups
> - Windows Server Backup is free and built-in but limited (full backups only, no dedup, no reporting)
> - Veeam Community Edition is free for up to 10 VMs and is far more capable for most environments
> - "Altaro VM Backup" is now "Hornetsecurity VM Backup" -- same product, renamed in 2022
> - Always test restores -- a backup you've never restored from is a theory, not a safety net
> - Follow the 3-2-1 rule: 3 copies, 2 media, 1 offsite

---

> ### What You've Built: End of Part 1
>
> You've reached the end of Part 1. You can now install Hyper-V, create and configure virtual machines, design virtual networks, manage storage, use checkpoints safely, and protect your environment with real backups.
>
> If you've followed the exercises, you have a working Hyper-V host with at least two or three VMs, virtual switches connecting them in different isolation configurations, and a backup you've actually tested restoring. That's a functional virtualisation environment -- not a thought experiment.
>
> Part 2 builds on this foundation: monitoring, high availability, security hardening, PowerShell automation, and integration with enterprise tools. Everything ahead assumes you're comfortable with the infrastructure you've built in these six chapters.

---

A backup is only as good as your confidence in it -- and that confidence comes from knowing your environment is healthy before something goes wrong. Chapter 7 covers the monitoring and performance tools that give you that visibility: the counters worth watching, the thresholds that matter, and how to tell the difference between a busy host and a struggling one.

---

## Exercise 6: Set Up and Test a Backup

> **Estimated time: 30--45 minutes** (depends on VM size; a 40 GB VM backup can take 10--20 minutes on typical hardware)

1. Install Windows Server Backup on your Hyper-V host (`Install-WindowsFeature -Name Windows-Server-Backup`).
2. Create a one-time backup of one of your VMs to a local folder.
3. Deliberately make a visible change inside the VM (create a file on the desktop, or change the wallpaper).
4. Restore the VM from the backup to an alternate location (so you don't overwrite the running VM).
5. Import the restored VM in Hyper-V Manager (Actions > Import Virtual Machine).
6. Start the restored VM and verify the change you made is absent -- confirming the backup captured the pre-change state.
7. Clean up: delete the imported test VM and the backup files when done.
