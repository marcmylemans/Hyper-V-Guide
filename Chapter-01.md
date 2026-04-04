# Chapter 1: Introduction to Hyper-V

## 1.1 What Is a Hypervisor?

Before diving into Hyper-V specifically, it's worth understanding what a hypervisor actually is -- because that understanding will make everything else in this guide click into place.

A **hypervisor** is software (or firmware) that creates and manages virtual machines. It sits between the physical hardware and the operating systems running on top of it, allocating CPU, memory, disk, and network resources to each virtual machine as needed. Each virtual machine is completely isolated from the others -- a crash or virus inside one VM cannot directly affect its neighbours.

There are two categories of hypervisor:

**Type 1 (bare-metal):** Runs directly on the physical hardware. There's no underlying operating system. Examples: Hyper-V, VMware ESXi, Proxmox VE. These are used in production environments because they're fast and efficient.

**Type 2 (hosted):** Runs as an application on top of an existing operating system. Examples: VMware Workstation, VirtualBox, Parallels. Great for development and testing on a personal laptop, but not used in production at scale.

Hyper-V is a **Type 1 hypervisor**. Even on Windows 10 or 11, when you enable Hyper-V, Windows itself actually runs as a guest on the hypervisor -- the hypervisor loads first and gives Windows its own privileged partition. You're always running on bare metal, even when it looks like you're running inside Windows.

## 1.2 Why Hyper-V?

You have options when it comes to hypervisors. Here's why Hyper-V is worth learning:

**It's already included.** If you have Windows 10/11 Pro, Enterprise, or Education, or any edition of Windows Server, Hyper-V is already part of your license. Nothing extra to buy or download.

**It scales from laptop to datacenter.** The same skills you build on a home lab translate directly to enterprise Windows Server environments. Hyper-V powers some of the largest virtualisation deployments in the world.

**It's what runs Azure.** Microsoft Azure is built on a customised version of Hyper-V. Understanding the platform here gives you direct insight into how cloud infrastructure works.

**It integrates natively with Windows.** Active Directory, PowerShell, Windows Admin Center, Windows Server Backup -- all of these work with Hyper-V without any extra configuration or compatibility layers.

## 1.3 Hyper-V Editions

Hyper-V is available in several forms depending on which Microsoft product you're running:

### Windows 10 / 11 Hyper-V

Available on **Pro, Enterprise, and Education** editions only. The Home edition does not include Hyper-V. This version is ideal for developers, IT students, and home lab users. It supports most Hyper-V features with some limitations (no Replica, no Live Migration).

### Windows Server Hyper-V (as a role)

The full production version. Install it by enabling the Hyper-V role through Server Manager. Available on Windows Server Standard and Datacenter (including Server Core installations -- no GUI required). The Datacenter edition unlocks additional features like Shielded VMs and Storage Spaces Direct.

> **Server Core:** Windows Server can be installed without a graphical interface at all, in a mode called Server Core. Hyper-V runs perfectly on Server Core -- management is done via PowerShell or remotely through Windows Admin Center. Many production Hyper-V hosts run Server Core to reduce the attack surface and update overhead.

### A Note on the Discontinued Hyper-V Server

Microsoft previously offered a free standalone product called "Microsoft Hyper-V Server" -- essentially Windows Server Core with only the Hyper-V role, free to download. The final version reached end-of-support in January 2022. You may encounter it in older environments, but it should not be used for new deployments.

## 1.4 System Requirements

Before enabling Hyper-V, verify your hardware meets these requirements.

### Required Hardware

| Requirement | Detail |
|---|---|
| Processor | 64-bit processor |
| Virtualisation | Intel VT-x or AMD-V, enabled in BIOS/UEFI |
| SLAT | Intel EPT (Extended Page Tables) or AMD NPT/RVI |
| RAM | Minimum 4 GB (8+ GB recommended for running multiple VMs) |
| Storage | Enough space for your VMs (plan at least 50-100 GB free) |

**What to look for in your BIOS/UEFI:**

- Intel systems: Look for "Intel Virtualization Technology" or "VT-x" -- enable it. Also look for "VT-d" if you plan to use DDA (passing physical devices to VMs).
- AMD systems: Look for "SVM Mode" or "AMD-V" -- enable it.
- SLAT is generally enabled by default on all processors made after 2010 and doesn't need a separate BIOS toggle.

> **How to check quickly (Windows):** Open PowerShell and run `systeminfo`. Look for the "Hyper-V Requirements" section at the bottom. It will tell you whether each requirement is met.

### What You Don't Need

You do not need a dedicated server. A modern laptop or desktop with at least 8 GB of RAM will run two or three VMs comfortably. An 8-core CPU with 16-32 GB of RAM is an excellent Hyper-V host for home lab use.

## 1.5 How to Enable Hyper-V

### On Windows 10 / 11

**Method 1: Windows Features (GUI)**

1. Press **Win + R**, type `optionalfeatures`, and press Enter.
2. Scroll to **Hyper-V**, expand it, and check both **Hyper-V Management Tools** and **Hyper-V Platform**.
3. Click **OK** and allow Windows to make the changes.
4. Restart when prompted.

**Method 2: PowerShell (faster)**

Open PowerShell as Administrator and run:
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```
Restart when it completes.

### On Windows Server

1. Open **Server Manager**.
2. Click **Manage > Add Roles and Features**.
3. Step through the wizard. When you reach "Server Roles", select **Hyper-V**.
4. Follow through the wizard -- you'll be asked to configure a virtual switch and live migration settings (you can skip these and configure them later).
5. The server will restart to complete installation.

**Via PowerShell (Server Core or scripted installs):**
```powershell
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
```

## 1.6 Hyper-V Manager

Once Hyper-V is installed, your primary management tool is **Hyper-V Manager**. Find it by searching "Hyper-V Manager" in the Start menu.

The Hyper-V Manager window has three panes:

- **Left pane:** Lists your Hyper-V hosts. Initially just your local machine. You can add remote Hyper-V hosts here too.
- **Centre pane:** Shows all virtual machines on the selected host, their status, and resource usage.
- **Right pane:** Actions you can take -- create new VMs, open Virtual Switch Manager, edit disk settings, etc.

For more powerful management (especially across multiple hosts), **Windows Admin Center** is the modern alternative. It's a free, browser-based tool from Microsoft that manages Hyper-V, storage, networking, and more from a single interface. We'll cover it in Chapter 11.

## 1.7 Key Concepts to Know Before Moving On

Before creating your first VM in the next chapter, make sure these concepts are solid:

**Host:** The physical machine running Hyper-V.

**Guest / Virtual Machine (VM):** A computer running inside the host, fully isolated from other VMs.

**Virtual Switch:** Software-defined network switch that connects VMs to each other and/or the physical network.

**Virtual Hard Disk (VHDX):** A file on the host that acts as a virtual hard drive for a VM.

**Checkpoint / Snapshot:** A saved state of a VM at a point in time. Lets you roll back if something goes wrong.

**Integration Services:** Drivers and guest utilities that Hyper-V installs into a VM to improve performance and enable features like graceful shutdown and time synchronisation.

---

> **Key Takeaways**
> - Hyper-V is a Type 1 (bare-metal) hypervisor -- it loads before Windows, not inside it
> - Included free with Windows 10/11 Pro/Enterprise/Education and all Windows Server editions
> - Requires a 64-bit CPU with Intel VT-x/AMD-V and SLAT (EPT/NPT) -- check your BIOS
> - Minimum 4 GB RAM; 8+ GB strongly recommended for running multiple VMs
> - Hyper-V Manager is the default tool; Windows Admin Center is the modern web-based alternative

---

Hyper-V is installed and verified -- but a hypervisor with no virtual machines is an empty stage. The next step is to populate it: Chapter 2 takes you through creating your first VM from scratch, covering the configuration choices that matter, the installation wizard, and getting the OS running before you hand it off.

---

## Exercise 1: Verify Your Host is Ready

> **Estimated time: 15 minutes**

Before proceeding to Chapter 2, confirm Hyper-V is installed and working:

1. Open PowerShell as Administrator.
2. Run: `Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V`
   - The `State` should show `Enabled`.
3. Open Hyper-V Manager. Confirm you can see your computer name in the left pane.
4. Run: `Get-VMHost` in PowerShell. This returns your host's current Hyper-V configuration. If it returns results, Hyper-V is fully operational.

If `Get-WindowsOptionalFeature` shows `Disabled`, go back to section 1.5 and enable it.
