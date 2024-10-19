# Chapter 2: Setting Up Your First Virtual Machine

## 2.1 Planning Your Virtual Machine

Before you begin creating a virtual machine (VM), it’s important to plan out the configuration and resources your VM will need. A well-thought-out plan can prevent performance issues or the need to reconfigure later on. Consider the following:

- **Purpose of the VM:** What will the VM be used for? Common purposes include testing, development, or running specific applications in a controlled environment.
- **Operating System:** Determine which operating system (OS) you will install on the VM. This could be a Windows OS or a distribution of Linux, depending on your needs.
- **Resource Allocation:** Plan how much CPU power, memory (RAM), and disk space the VM will need. This depends on the purpose of the VM and the OS it will run.

> **Tip:** If you’re unsure about resource allocation, start with a moderate configuration and adjust as needed later. You can always modify the VM settings in Hyper-V after the machine is created.

## 2.2 Creating a New Virtual Machine

Now that you have a plan, let’s walk through the steps to create your first virtual machine in Hyper-V.

### Step-by-Step Guide to Create a VM in Hyper-V:

1. **Open Hyper-V Manager:**
   - Launch **Hyper-V Manager** from the Start menu or through **Server Manager**.

2. **Connect to the Server:**
   - In Hyper-V Manager, ensure you are connected to the correct host machine (the server or local machine running Hyper-V).

3. **Start the New Virtual Machine Wizard:**
   - In the right-hand pane, click on **New**, then select **Virtual Machine** from the dropdown.

   ![Create Virtual Machine](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-2/Chapter-2-2-3.png)

4. **Specify Name and Location:**
   - Give your virtual machine a descriptive name, especially if you plan to manage multiple VMs.
   - You can choose a custom location to store the VM files, or leave it as default if storage location is not a concern.

   ![Name and location](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-2/Chapter-2-2-4.png)

5. **Specify Generation:**
   - Choose the **Generation 1** option if you need broader compatibility with older operating systems or applications.
   - Choose **Generation 2** for newer VMs that support features like UEFI, Secure Boot, and more efficient resource utilization.

   ![Generation 2](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-2/Chapter-2-2-5.png)

   > **Note:** Generation 2 is recommended for most modern operating systems, but some older OS versions may not support it.

6. **Assign Memory:**
   - Allocate memory for your VM. For example, a typical VM might start with **2048 MB** (2 GB) of RAM for basic tasks.
   - Optionally, you can enable **Dynamic Memory**, which allows Hyper-V to adjust the amount of memory assigned to the VM based on its workload, optimizing resource usage.

   ![Assign Memory](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-2/Chapter-2-2-6.png)

7. **Configure Networking:**
   - Connect the VM to an existing virtual switch. If no switch is available, you will need to create one in Hyper-V Manager to enable network connectivity for the VM.

8. **Connect Virtual Hard Disk:**
   - Create a new virtual hard disk (VHD) for the VM or use an existing one if applicable.
   - Specify the size of the disk based on your needs. For instance, a basic installation might require at least **50 GB** of storage.

   ![Connect Virtual Hard Disk](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-2/Chapter-2-2-8.png)

9. **Install Operating System:**
   - Select the option **Install an operating system from a bootable image file**.
   - Browse for and select the **ISO file** for your chosen OS.

   > **Tip:** Make sure to download a compatible ISO file beforehand from the official website or another trusted source.

10. **Complete the Wizard:**
    - Review all the settings you’ve chosen in the summary page. If everything looks good, click **Finish** to complete the creation of your new VM.

## 2.3 Installing the Operating System

Once the VM has been created, the next step is to install the operating system. The process is similar to installing an OS on a physical machine.

### Booting the VM:
1. **Start the VM:**
   - In Hyper-V Manager, right-click your new VM and select **Connect**.
   - In the Virtual Machine Connection window, click **Start**.

2. **Install the OS:**
   - Follow the typical OS installation steps, just as you would on a physical machine. For Windows, this may involve entering a product key, selecting partitions, and creating user accounts. For Linux, it might involve similar steps depending on the distribution.

> **Tip:** Keep in mind that OS installation may take several minutes depending on your system’s performance and the resources allocated to the VM.

## 2.4 Configuring the Virtual Machine

After the OS installation, there are several post-installation tasks to complete to ensure your virtual machine is fully optimized and functional.

### Post-Installation Configuration:
1. **Install Integration Services:**
   - Hyper-V’s **Integration Services** improve the performance of the VM by providing better drivers and guest services (like time synchronization, shutdown integration, etc.).
   - For Windows VMs, Integration Services are typically installed automatically. For Linux, you may need to install them manually, depending on the distribution.

2. **Configure VM Settings:**
   - After installation, you may want to adjust the VM’s CPU, memory, and network settings.
   - Access **Settings** by right-clicking the VM in Hyper-V Manager and selecting **Settings**. From here, you can change configurations such as increasing memory or adjusting processor allocation.

### Snapshot Management:
- **Create Snapshots:** A snapshot captures the state of your VM at a given point in time, allowing you to revert back to this state if necessary. This is particularly useful before major system updates or configuration changes.
- **Apply/Remove Snapshots:** Snapshots can be applied to roll back the VM to a previous state. You can also delete unnecessary snapshots to free up space.

> **Best Practice:** Always create a snapshot before making significant changes to the VM’s configuration or installing new software. This gives you a safety net if something goes wrong.

## 2.5 Managing Virtual Machines

Managing your VM after it’s up and running is key to maintaining performance and ensuring smooth operation. Here are some basic management tasks:

### Starting and Stopping VMs:
- **Start:** To start a VM, right-click the VM in Hyper-V Manager and select **Start**.
- **Shut Down:** Inside the VM, shut down the operating system properly to avoid data corruption, just as you would on a physical machine.
- **Turn Off:** This option forces the VM off without shutting down the OS properly. It is not recommended except in emergency situations.

### Monitoring Performance:
- Inside the VM, use **Task Manager** (Windows) or **Resource Monitor** to monitor CPU, memory, and disk usage.
- In Hyper-V Manager, use the built-in **Performance Monitor** to keep an eye on both the host machine and the VM’s resource usage over time.

> **Pro Tip:** Regularly monitor your VMs to ensure they are using resources efficiently and make adjustments as necessary. Hyper-V allows you to easily increase memory or add more CPUs if needed.

---

With your first virtual machine now up and running, you’re ready to explore more advanced configurations and management techniques. In the next chapter, we’ll dive deeper into networking and virtual switch management, allowing you to connect your VMs to different environments or external networks.
