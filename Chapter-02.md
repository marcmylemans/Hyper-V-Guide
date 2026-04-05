# Chapter 2: Setting Up Your First Virtual Machine

Creating your first VM is one of those moments that makes the whole concept of virtualisation click. One minute you have an empty Hyper-V Manager, the next you're watching a real operating system boot inside a window on your desktop. This chapter walks through the whole process -- planning, creation, OS installation, and the post-install steps that most guides skip.

## 2.1 Planning Your Virtual Machine

Taking two minutes to think before clicking "New" will save you from having to reconfigure things later. Answer these three questions:

**What is this VM for?** A web server, a test environment, a domain controller, a Linux sandbox? The purpose determines what OS you need and how many resources to allocate.

**Which operating system?** Your OS choice affects which VM generation to use (more on that in section 2.2) and how much disk space to allocate. Use these minimums as a starting point:

| Operating System | Min RAM | Min Disk |
|---|---|---|
| Windows Server 2022 (Core) | 512 MB | 32 GB |
| Windows Server 2022 (Desktop) | 2 GB | 32 GB |
| Windows 11 | 4 GB | 64 GB |
| Ubuntu Server 22.04 | 512 MB | 10 GB |
| Ubuntu Desktop 22.04 | 2 GB | 25 GB |
| Rocky Linux / AlmaLinux | 1 GB | 10 GB |

**How much RAM does the host have to spare?** A good rule: always leave at least 2-4 GB of RAM free for the host OS itself. If your machine has 16 GB and the host needs 4 GB, you have 12 GB to distribute across VMs.

> **Tip:** Start conservative. It's easy to add RAM to a VM after creation. It's harder to shrink a disk that's already in use.

## 2.2 VM Generation: 1 or 2?

Hyper-V offers two VM generations, and this is one of the first choices the wizard asks you to make.

**Generation 1** emulates legacy BIOS hardware. It's compatible with older operating systems -- anything from Windows XP through Windows Server 2003 onwards, and older Linux distributions. It uses IDE controllers for the boot drive and has a simpler (but slower) firmware layer.

**Generation 2** uses UEFI firmware, Secure Boot, and SCSI for all storage. It boots significantly faster, supports larger VMs, and is more efficient. Use Generation 2 for any modern OS.

**Quick rule:** Use Generation 2 unless you specifically need to run an operating system that doesn't support UEFI. This includes: Windows XP, Windows Server 2003, FreeBSD (check current compatibility before using), and some very old Linux distributions.

> **Linux and Generation 2:** Most modern Linux distributions (Ubuntu 18.04+, Debian 10+, Rocky Linux 8+, RHEL 8+) work perfectly with Generation 2. If you're unsure about a specific distro, check the [Microsoft documentation on Supported Linux VMs](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/supported-linux-and-freebsd-virtual-machines-for-hyper-v-on-windows).

## 2.3 Creating a New Virtual Machine

Here's the step-by-step process using Hyper-V Manager:

1. **Open Hyper-V Manager** and select your host in the left pane.

2. In the right **Actions** pane, click **New > Virtual Machine**. The New Virtual Machine Wizard opens.

3. **Specify Name and Location:** Give your VM a clear, descriptive name (for example, `WS2022-DC01` for a Windows Server 2022 domain controller). You can also change the storage location here -- useful if your default drive is running low on space.

4. **Specify Generation:** Choose **Generation 2** for modern operating systems (see section 2.2).

5. **Assign Memory:**
   - Set a **Startup RAM** appropriate for your OS (see the table in section 2.1).
   - Enable **Dynamic Memory** for most VMs. This lets Hyper-V automatically scale the VM's RAM between a minimum and maximum based on actual demand, making better use of the host's physical memory.
   - Leave Dynamic Memory off if the VM will run a database or other workload that benefits from a consistent, reserved memory allocation.

6. **Configure Networking:** Connect the VM to a virtual switch. If you haven't created one yet, you can leave this as "Not Connected" and add a switch later.

7. **Connect Virtual Hard Disk:** Select **Create a virtual hard disk**. Choose a path, enter a name, and set a size appropriate for your OS. The disk will be created in VHDX format (the correct modern format -- more on this in Chapter 4).

8. **Installation Options:** Select **Install an operating system from a bootable image file** and browse to your ISO file. Download ISOs from official sources: [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) for Windows Server, or the distribution's official site for Linux.

9. **Summary:** Review your settings. Click **Finish**.

Your VM now appears in Hyper-V Manager. It's created but not yet started.

## 2.4 Installing the Operating System

1. Right-click your new VM in Hyper-V Manager and select **Connect**. The Virtual Machine Connection window opens.

2. Click **Start** (the green play button). The VM boots from your ISO.

3. Follow the OS installation steps as you would on a physical machine.

> **Tip for Windows Server:** You'll be asked to choose between "Server Core" (no GUI) and "Desktop Experience" (full GUI). Server Core uses significantly less RAM and disk, and is Microsoft's recommended deployment for most server roles -- but it requires PowerShell or remote management tools. Desktop Experience is easier for beginners. Choose whichever matches your comfort level.

> **Tip for Linux:** Most Linux installers work well in Hyper-V without any special steps. If your installer boots but shows a black screen, try switching the display adapter to "Standard" in the VM settings, or append `nomodeset` to the kernel boot parameters.

## 2.5 Post-Install: Integration Services

After the OS is installed, the first thing to check is Integration Services. These are drivers and guest utilities that Hyper-V provides to improve VM performance and enable features like:

- Graceful shutdown (so the host can ask the VM to shut down properly, rather than cutting power)
- Time synchronisation with the host
- Copy-paste between host and VM (via Enhanced Session Mode)
- Heartbeat (the host monitors that the VM is still responsive)

**For Windows guests:** Integration Services are included in Windows Vista and later. They install automatically when Hyper-V detects a compatible OS. You don't need to do anything. Verify they're running by checking **Services** inside the VM for "Hyper-V Guest Service Interface."

**For Linux guests:** Since Linux kernel version 4.x (which covers Ubuntu 16.04 and later, Debian 9 and later, RHEL/CentOS 7.4 and later, and virtually all modern distributions), Linux Integration Services are built into the kernel itself. No manual installation required. The old practice of downloading and installing LIS separately is now only needed for very old distros with kernels earlier than 4.x.

## 2.6 Enhanced Session Mode

Enhanced Session Mode is a feature that dramatically improves the day-to-day experience of working in VMs. Instead of the basic VMBus video connection, it uses an internal RDP connection that enables:

- **Copy-paste** between the host and VM
- **Audio** playback from inside the VM
- **Local drive and USB redirection** -- share folders and USB devices from your host into the VM
- **Clipboard history** between host and VM

**To enable Enhanced Session Mode:**

On **Windows 10 and Windows 11 hosts**, Enhanced Session Mode is enabled by default -- you don't need to change anything.

On **Windows Server hosts**, you need to enable it manually: in Hyper-V Manager, click on your host name in the left pane, then click **Hyper-V Settings** in the Actions pane on the right. In the Settings dialog, expand the **Server** section in the left tree and select **Enhanced Session Mode Policy** -- check **Allow enhanced session mode**. Then expand the **User** section and select **Enhanced Session Mode** -- check **Use enhanced session mode**. Click OK.

> **Note:** Enhanced Session Mode requires the guest OS to support it. Windows 8/10/11 and Windows Server 2012 and later support it out of the box. For Linux, you need to install the `xrdp` package and the `hyperv-daemons` package, then enable the services.

## 2.7 Configuring VM Settings

After install, it's worth reviewing the VM's settings. Right-click the VM in Hyper-V Manager and select **Settings**.

Key settings to review:

- **Processor:** By default, a VM gets 1 virtual processor. For VMs doing CPU-heavy work, increase this. Be careful not to over-allocate -- giving 8 VMs 8 vCPUs each on a 4-core host creates contention.
- **Memory:** Adjust startup RAM and Dynamic Memory min/max as needed.
- **Network Adapter:** Add or change virtual switch connections here.
- **Integration Services:** Confirm which integration services are enabled (Heartbeat, Shutdown, Time Synchronization, etc.).
- **Checkpoints:** By default, Hyper-V creates "Production Checkpoints" (VSS-based, application-consistent). This is the recommended setting. "Standard Checkpoints" capture memory state too, which is useful for testing but not recommended for production VMs.

## 2.8 Checkpoints (Snapshots)

A checkpoint saves the state of a VM at a specific point in time. If a change you make breaks something, you can revert to the checkpoint.

**Create a checkpoint:** Right-click the VM in Hyper-V Manager > **Checkpoint**. Or in PowerShell:
```powershell
Checkpoint-VM -Name "MyVM" -SnapshotName "Before-Updates"
```

**Apply a checkpoint (roll back):** Right-click the checkpoint in the Checkpoints pane at the bottom of Hyper-V Manager > **Apply**.

**Delete a checkpoint:** Right-click > **Delete Checkpoint**. This does not delete the VM -- it just removes that saved point.

> **Best Practice:** Always create a checkpoint before:
> - Installing new software
> - Applying Windows Updates (especially major feature updates)
> - Making significant configuration changes
> - Running anything you're unsure about
>
> Checkpoints are not a substitute for backups. They live alongside the VM on the same physical disk. If the disk fails, you lose both the VM and the checkpoint. Chapter 6 covers proper backups.

## 2.9 Starting, Stopping, and Managing VMs

**Starting a VM:** Right-click > **Start**, or select the VM and press the Start button in the toolbar.

**Graceful shutdown:** Right-click > **Shut Down**. This asks the guest OS to shut down cleanly. Requires Integration Services.

**Force off:** Right-click > **Turn Off**. Equivalent to yanking the power cable. Use only when the VM is unresponsive and you need to stop it immediately.

**Pausing:** Right-click > **Pause**. Freezes the VM's execution without shutting it down. Useful for temporarily freeing up CPU on the host. Resume with right-click > **Resume**.

**Saving state:** Right-click > **Save**. Writes the VM's memory to disk and stops it. When you start it again, it resumes from exactly where it left off (similar to hibernating a laptop). Note that saved-state VMs cannot be backed up in a consistent way.

---

> **Key Takeaways**
> - Generation 2 is the right choice for all modern operating systems; Generation 1 is for legacy OS compatibility
> - Dynamic Memory lets Hyper-V scale VM RAM automatically -- enable it for most VMs
> - Integration Services are built into the Linux kernel 4.x+ -- no manual install needed on modern distros
> - Enhanced Session Mode enables copy-paste, audio, and drive sharing between host and VM
> - Checkpoints are your safety net before risky changes -- but they're not a backup

---

You now have a VM running, but it's isolated from everything -- no network means no updates, no communication with other VMs, no access to the outside world. Networking is what makes VMs genuinely useful, and Chapter 3 covers how to connect them: virtual switches, the difference between External, Internal, and Private, and how to give your VMs the right level of access.

---

## Exercise 2: Build and Configure Your First VM

> **Estimated time: 45--60 minutes** (excluding ISO download time -- the Windows Server 2022 evaluation ISO is approximately 5 GB)

1. Download a Windows Server 2022 Evaluation ISO from [microsoft.com/en-us/evalcenter](https://www.microsoft.com/en-us/evalcenter/) (free, 180-day trial).
2. Create a Generation 2 VM with 2 GB startup RAM, Dynamic Memory enabled (min 512 MB, max 4 GB), and a 40 GB VHDX.
3. Install Windows Server 2022 Desktop Experience.
4. After install, enable Enhanced Session Mode on the host.
5. Connect to the VM using Enhanced Session Mode and verify copy-paste works between host and VM.
6. Create a checkpoint named "Fresh Install".
7. Make a change inside the VM (install Notepad++ or change the desktop wallpaper).
8. Apply the "Fresh Install" checkpoint and verify the change is gone.
