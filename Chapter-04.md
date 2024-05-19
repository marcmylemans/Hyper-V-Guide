# Chapter 4: Storage in Hyper-V

## 4.1 Understanding Virtual Hard Disks (VHDs)

In Hyper-V, virtual hard disks (VHDs) are used to store the operating system, applications, and data for virtual machines. There are two main types of VHDs:

### Types of VHDs:

1. **VHD (Virtual Hard Disk):**
   - Older format, compatible with older versions of Hyper-V and other virtualization platforms.
   - Maximum size of 2 TB.

2. **VHDX (Virtual Hard Disk Extended):**
   - Newer format, introduced in Windows Server 2012.
   - Supports larger sizes (up to 64 TB) and provides better performance and resilience.

## 4.2 Creating and Managing Virtual Hard Disks

### Creating a New VHD:

1. **Open Hyper-V Manager:**
   - Open **Hyper-V Manager** from the Start menu or Server Manager.

2. **Access New Virtual Hard Disk Wizard:**
   - In the right pane, click on **New**, then **Hard Disk**.
     ![New Virtual Hard Disk Wizard](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-2-2.png)

3. **Choose Disk Format:**
   - Select either **VHD** or **VHDX** format.
     ![Choose Disk Format](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-2-3.png)

4. **Choose Disk Type:**
   - **Fixed Size:** Allocates the entire disk size immediately.
   - **Dynamically Expanding:** Expands as data is added, up to the maximum size.
   - **Differencing:** Tracks changes from a parent disk, useful for test environments.
     ![Choose Disk Type](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-2-4.png)

5. **Specify Name and Location:**
   - Enter a name for the VHD and choose a location to store the file.
     ![Name and Location](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-2-5.png)

6. **Specify Disk Size:**
   - Enter the maximum size for the VHD.
     ![Disk Size](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-2-6.png)

7. **Complete the Wizard:**
   - Review your settings and click **Finish**.

### Attaching a VHD to a VM:

1. **Open VM Settings:**
   - In Hyper-V Manager, right-click the VM and select **Settings**.

2. **Add Hard Drive:**
   - In the left pane, click **SCSI Controller** or **IDE Controller**, then **Hard Drive**.
   - Click **Add**.

3. **Select Virtual Hard Disk:**
   - Browse and select the VHD file to attach.
   - Click **Apply** and then **OK**.

## 4.3 Configuring Storage for VMs

### Storage Options:

- **Pass-through Disks:**
  - Allows VMs to access physical disks directly.
  - Offers better performance but less flexibility.

- **Shared Storage:**
  - Use shared storage solutions (e.g., iSCSI, SMB) for high availability and live migration.
  - Suitable for clustered environments.

### Expanding VHDs:

1. **Expand Virtual Hard Disk:**
   - In Hyper-V Manager, select **Edit Disk** from the Actions pane.
   - Browse to the VHD file and select **Expand**.
   - Specify the new size and complete the wizard.

2. **Extend Volume Inside VM:**
   - Inside the VM, use **Disk Management** to extend the volume.
   - Right-click the volume and select **Extend Volume**.
     ![Disk Size](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-04/Chapter-04-3-2.png)

## 4.4 Backing Up and Restoring VMs

### Backup Strategies:

- **Regular Backups:**
  - Schedule regular backups of VMs and their VHDs.
  - Use Windows Server Backup or third-party solutions.

- **Snapshots:**
  - Take snapshots to capture the state of a VM at a specific point in time.
  - Useful for quick recovery during updates or changes.

### Restoring VMs:

1. **Restore from Backup:**
   - Use your backup software to restore the VM files.
   - Import the restored VM in Hyper-V Manager.

2. **Apply Snapshots:**
   - In Hyper-V Manager, right-click the VM and select **Checkpoint**.
   - Choose the desired snapshot and click **Apply**.

## 4.5 Managing Storage Performance

### Optimizing VHD Performance:

- **Use VHDX Format:**
  - Prefer VHDX for better performance and resilience.
  
- **Separate VHDs:**
  - Store OS and data on separate VHDs for improved performance.

- **Defragment VHDs:**
  - Periodically defragment VHDs to maintain performance.

### Monitoring Disk Usage:

- **Performance Monitor:**
  - Use Performance Monitor to track disk I/O and latency.

- **Resource Monitor:**
  - Inside the VM, use Resource Monitor to monitor disk activity.
