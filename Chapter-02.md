# Chapter 2: Setting Up Your First Virtual Machine

## 2.1 Planning Your Virtual Machine

Before creating a virtual machine (VM), consider the following:
- **Purpose of the VM:** What will it be used for? (e.g., testing, development, running specific applications)
- **Operating System:** Which OS will you install on the VM? (e.g., Windows, Linux)
- **Resource Allocation:** How much CPU, memory, and disk space will the VM need?

## 2.2 Creating a New Virtual Machine

### Step-by-Step Guide to Create a VM in Hyper-V:

1. **Open Hyper-V Manager:**
   - Open **Hyper-V Manager** from the Start menu or Server Manager.

2. **Connect to the Server:**
   - In Hyper-V Manager, select your host machine.

3. **Start the New Virtual Machine Wizard:**
   - In the right pane, click on **New**, then **Virtual Machine**.

4. **Specify Name and Location:**
   - Enter a name for your VM.
   - Choose a location to store the VM files or use the default.

5. **Specify Generation:**
   - Choose **Generation 1** for compatibility or **Generation 2** for newer features (UEFI, Secure Boot).

6. **Assign Memory:**
   - Allocate the amount of startup memory (e.g., 2048 MB).
   - Optionally, enable **Dynamic Memory** to optimize usage.

7. **Configure Networking:**
   - Connect the VM to a virtual switch. If no switch is available, create one in Hyper-V Manager.

8. **Connect Virtual Hard Disk:**
   - Create a new virtual hard disk (VHD) or use an existing one.
   - Specify the size of the disk (e.g., 50 GB).

9. **Install Operating System:**
   - Choose **Install an operating system from a bootable image file**.
   - Browse and select the ISO file for your OS.

10. **Complete the Wizard:**
    - Review your settings and click **Finish**.

![Create VM Wizard](https://www.nakivo.com/blog/wp-content/uploads/2019/01/Creating-New-VM-with-Hyper-V-Manager.webp)

## 2.3 Installing the Operating System

### Booting the VM:
1. **Start the VM:**
   - In Hyper-V Manager, right-click your new VM and select **Connect**.
   - Click **Start** in the Virtual Machine Connection window.

2. **Install the OS:**
   - Follow the OS installation steps as you would on a physical machine.

![OS Installation](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/media/osdeploy_upd.png)

## 2.4 Configuring the Virtual Machine

### Post-Installation Configuration:
1. **Install Integration Services:**
   - Integration Services improve VM performance and functionality.
   - For Windows VMs, they are included and automatically installed. For Linux VMs, you may need to install them manually.

2. **Configure VM Settings:**
   - Adjust CPU, memory, and network settings as needed.
   - Access **Settings** by right-clicking the VM in Hyper-V Manager and selecting **Settings**.

### Snapshot Management:
- **Create Snapshots:** Take snapshots to save the VM state at a specific point in time.
- **Apply/Remove Snapshots:** Use snapshots to revert to previous states or free up space.

![VM Settings](https://uploads-us-west-2.insided.com/workspot-en/attachment/08bc4bbd-7829-48eb-a91e-3698ef1c667c.png)

## 2.5 Managing Virtual Machines

### Starting and Stopping VMs:
- **Start:** Right-click the VM and select **Start**.
- **Shut Down:** Inside the VM, shut down the OS properly.
- **Turn Off:** Right-click the VM and select **Turn Off** (not recommended, as it forces the VM off).

### Monitoring Performance:
- Use **Task Manager** or **Resource Monitor** inside the VM to check resource usage.
- In Hyper-V Manager, view the **Performance Monitor** to monitor host and VM performance.

![Performance Monitoring](https://www.nakivo.com/blog/wp-content/uploads/2020/12/Hyper-V-performance-monitoring-by-using-Performance-Monitor.webp)
