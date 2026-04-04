# Chapter 4: Storage in Hyper-V

Every virtual machine needs a virtual hard drive. This chapter covers how Hyper-V handles storage: the file formats available, the different disk types and when to use each, how to expand and manage disks after creation, and how to get the best performance out of your storage setup.

## 4.1 VHD vs VHDX: Choose VHDX

Hyper-V supports two virtual hard disk formats:

**VHD (Virtual Hard Disk):** The original format, introduced with Hyper-V 1.0. Maximum size of **2 TB**. No resilience against corruption from unexpected power loss. Supported by older hypervisors and virtualisation platforms.

**VHDX (Virtual Hard Disk Extended):** Introduced with Windows Server 2012. Maximum size of **64 TB**. Resistant to corruption during unexpected host power failures. Better alignment with modern storage hardware. Native support for large sector disks (4K sectors).

**Always use VHDX** for new virtual machines. The only reason to use VHD is compatibility with older Hyper-V hosts (Windows Server 2008 R2 or earlier), or if you're moving a disk to a non-Microsoft hypervisor.

## 4.2 Disk Types: Fixed, Dynamically Expanding, and Differencing

When you create a VHDX, you choose how it allocates space on the host's physical storage. Each has a different performance and storage trade-off.

### Fixed Size

A fixed-size disk allocates the full amount of disk space from the host immediately when the VHDX file is created. A 100 GB fixed disk creates a 100 GB file on your host right now, regardless of how much data is inside the VM.

- **Pros:** Best I/O performance because there's no overhead from expanding the file. No risk of the VM running out of space unexpectedly because the host fills up.
- **Cons:** Uses the full allocation immediately, even if the VM only needs 10 GB of actual data.
- **Use when:** Running I/O-intensive workloads (databases, file servers) where performance matters more than storage efficiency.

### Dynamically Expanding

A dynamically expanding disk starts small and grows as data is written to it, up to the maximum size you specified. A 100 GB dynamic disk might only be 15 GB on the host if the VM only has 15 GB of actual data.

- **Pros:** Efficient use of host storage. Easier to manage when you're running many VMs on limited space.
- **Cons:** Slight performance overhead as the file expands. If the host runs out of physical disk space while the VHDX is trying to expand, the VM will crash.
- **Use when:** Running development VMs, test machines, or any VM where storage efficiency matters more than peak I/O performance.

### Differencing

A differencing disk stores only the changes made from a **parent disk**. The differencing disk doesn't contain a full copy of the OS -- it contains only what's different from the parent.

Imagine creating a "base" VHDX with Windows Server fully installed and configured. You then create three differencing disks, each pointing at that base. Each differencing disk looks like a complete VM to the guest, but only stores its own unique changes. You've saved two full OS installations worth of disk space.

- **Pros:** Massive space savings when running multiple VMs with similar base configurations.
- **Cons:** Performance overhead from the indirection layer. If the parent disk is corrupted or moved, all differencing disks become invalid. Not suitable for production VMs without careful management.
- **Use when:** Running a lab environment with many similar VMs (e.g., multiple copies of "Windows Server base install" for testing).

## 4.3 Creating a Virtual Hard Disk

**Via Hyper-V Manager:**

1. In the right pane of Hyper-V Manager, click **New > Hard Disk**.
2. Choose **VHDX** format.
3. Choose your disk type (Fixed, Dynamically Expanding, or Differencing).
4. Specify a name and location.
5. Set the maximum disk size.
6. Click **Finish**.

**Via PowerShell:**
```powershell
# Create a 100 GB dynamically expanding VHDX
New-VHD -Path "C:\VMs\MyVM\Data.vhdx" -SizeBytes 100GB -Dynamic

# Create a 100 GB fixed-size VHDX
New-VHD -Path "C:\VMs\MyVM\OS.vhdx" -SizeBytes 100GB -Fixed

# Create a differencing disk based on a parent
New-VHD -Path "C:\VMs\Clone1\Clone1.vhdx" `
        -ParentPath "C:\VMs\Base\BaseOS.vhdx" `
        -Differencing
```

## 4.4 Attaching a Disk to a VM

A VHDX file on its own does nothing -- it needs to be connected to a VM.

**Via Hyper-V Manager:**

1. Right-click the VM > **Settings**.
2. In the left pane, select **SCSI Controller** (for Generation 2 VMs) or **IDE Controller** (for Generation 1).
3. Select **Hard Drive** on the right side, then click **Add**.
4. Select the VHDX file.
5. Click **Apply** and **OK**.

**Via PowerShell:**
```powershell
Add-VMHardDiskDrive -VMName "MyVM" -Path "C:\VMs\MyVM\Data.vhdx"
```

## 4.5 Expanding a Disk After Creation

VMs grow. At some point you'll need more disk space in a VM than you originally planned. Hyper-V makes this straightforward -- but there are two steps:

**Step 1: Expand the VHDX file (on the host)**

The VM must be shut down or the disk must be a data disk (not the boot disk) to expand it.

Via Hyper-V Manager: Click **Edit Disk** in the Actions pane. Browse to the VHDX file. Select **Expand**. Enter the new maximum size. Complete the wizard.

Via PowerShell:
```powershell
Resize-VHD -Path "C:\VMs\MyVM\OS.vhdx" -SizeBytes 200GB
```

**Step 2: Extend the partition inside the VM**

Expanding the VHDX makes more space available to the VM, but the VM doesn't automatically start using it. You need to extend the partition inside the guest OS.

In Windows (inside the VM):
1. Right-click the Start button > **Disk Management**.
2. You'll see your existing partition followed by "Unallocated" space.
3. Right-click the existing partition > **Extend Volume**.
4. Follow the wizard to use the unallocated space.

In Windows via PowerShell (inside the VM):
```powershell
# Get the partition to extend
$partition = Get-Partition -DriveLetter C

# Extend it to fill all available space
Resize-Partition -DiskNumber $partition.DiskNumber `
                 -PartitionNumber $partition.PartitionNumber `
                 -Size (Get-PartitionSupportedSize -DriveLetter C).SizeMax
```

In Linux (inside the VM):
```bash
# For an ext4 partition -- resize partition first, then filesystem
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1

# For XFS (common on RHEL/Rocky/AlmaLinux)
sudo xfs_growfs /
```

## 4.6 Storage Configuration for Better Performance

### Where you store VHDX files matters enormously

The single biggest storage performance factor is the physical disk underneath. Guidelines:

- **NVMe SSD:** Excellent for OS disks and I/O-intensive workloads. The best choice for active VMs.
- **SATA SSD:** Good for most VMs. A significant step up from spinning disks.
- **Spinning HDD (7200 RPM):** Acceptable for archiving, slow VMs, or large storage-only disks where I/O is not the bottleneck.
- **Spinning HDD (5400 RPM):** Not recommended for VM workloads.

### Separate OS and data disks

For VMs running databases or file servers, store the OS and data on separate VHDX files -- ideally on separate physical drives. This distributes I/O across two drives and makes it easier to resize or move data independently of the OS.

### Fixed vs Dynamic: performance note

Fixed-size VHDs consistently outperform dynamic ones in I/O-heavy workloads because there's no overhead from extending the file. For most home lab and development use, the difference is imperceptible. For databases and file servers in production, use fixed-size.

### Defragmentation and TRIM: important distinction

**If your VHDX files are stored on a spinning HDD:** Defragmenting the VHDX files themselves (from outside the VM, using the host OS) can improve sequential read performance on heavily fragmented dynamic VHDs. Use Windows' built-in Disk Defragmenter on the host.

**If your VHDX files are stored on an SSD:** Do NOT defragment. Defragmentation on an SSD causes unnecessary write cycles that wear out the drive without any performance benefit. Instead, ensure the host OS has scheduled TRIM/Optimise operations enabled (they are by default on Windows). TRIM tells the SSD which blocks are no longer in use, allowing it to maintain performance over time.

> **To summarise:** Defragment for HDDs. TRIM for SSDs. Never defragment an SSD.

## 4.7 Pass-Through Disks

A pass-through disk gives a VM direct access to a physical disk, bypassing the VHDX abstraction layer entirely. The physical disk appears directly inside the VM.

**When pass-through was useful:** In older Hyper-V versions (pre-2012), before VHDX was introduced, pass-through was the recommended way to achieve high I/O performance. It also allowed VMs to access disks larger than the then 2 TB VHD limit.

**Why VHDX is now preferred over pass-through:**
- Pass-through disks cannot be included in checkpoints/snapshots.
- Pass-through disks cannot be used with Live Migration to a new host unless the physical disk is accessible on both hosts (requires a SAN).
- A VHDX file on a modern NVMe SSD delivers performance that is nearly identical to pass-through in practice.

Pass-through disks are still supported but are no longer recommended for new deployments. Use VHDX on high-performance storage instead.

## 4.8 Shared Storage

> **Enterprise environments only.** This section applies to multi-host setups with centralised storage.

For Hyper-V clusters (Chapter 8), all hosts need to access the same storage. Common options:

**iSCSI:** Block storage over Ethernet. A NAS or storage server presents a LUN (Logical Unit Number) over your network. Hyper-V hosts mount it as a local disk. Cost-effective for small clusters.

**SMB 3.0 File Shares:** Hyper-V can store VM files on a network share using SMB 3.0. Windows Server 2012 and later support placing VHDXs directly on SMB file shares with good performance.

**Fibre Channel:** High-speed block storage using dedicated FC networking. Enterprise-grade performance but requires specialised hardware.

**Storage Spaces Direct (S2D):** Hyper-V hosts pool their local storage together into shared storage using Windows clustering features. Requires Windows Server Datacenter edition. Covered in Chapter 8.

---

> **Key Takeaways**
> - Always use VHDX format (not VHD) for new disks -- 64 TB max, better resilience
> - Fixed-size: best I/O performance, uses full allocation immediately
> - Dynamically expanding: efficient storage use, slight performance overhead
> - Differencing: great for labs with many similar VMs, not for production
> - After expanding a VHDX, you must also extend the partition inside the VM
> - Defragment VHDX files stored on HDDs; do NOT defragment VHDX files on SSDs -- use TRIM instead
> - Pass-through disks are largely obsolete -- VHDX on fast storage is preferred

---

Your VM infrastructure now has compute, networking, and storage covered -- the three fundamentals. Chapter 5 moves into what those foundations make possible: Hyper-V Replica for disaster recovery, Live Migration for moving VMs between hosts without downtime, and nested virtualization for running Hyper-V inside a VM.

---

## Exercise 4: Storage Experiments

> **Estimated time: 30--45 minutes**

1. Create a 30 GB dynamically expanding VHDX file manually (not via the VM wizard) using PowerShell.
2. Attach it to one of your existing VMs as a second disk.
3. Inside the VM, open Disk Management. You should see the new disk as "Unknown / Not Initialized." Initialize it as GPT, create a simple volume, and format it NTFS.
4. Copy some files to the new drive. Note the VHDX file size on the host before and after copying -- you should see it grow.
5. Shut down the VM. Expand the VHDX to 50 GB using PowerShell (`Resize-VHD`).
6. Start the VM. Use Disk Management to extend the volume to use the new space.
7. Confirm the drive now shows 50 GB of total capacity inside the VM.
