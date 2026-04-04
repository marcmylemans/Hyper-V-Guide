# Glossary

**Active Directory (AD):** A directory service developed by Microsoft for Windows domain networks. Provides centralized authentication, authorization, and policy management. Hyper-V features like Live Migration and Replica require AD for Kerberos authentication.

**APIPA (Automatic Private IP Addressing):** When a device configured for DHCP cannot reach a DHCP server, Windows automatically assigns an address in the 169.254.0.0/16 range. A VM with an APIPA address has no network connectivity to the broader network. Common in Hyper-V when a VM is connected to an Internal or Private switch with no DHCP server.

**ASR (Azure Site Recovery):** A Microsoft service that replicates on-premises VMs to Azure for cloud-based disaster recovery. Enables failover to Azure when the on-premises site is unavailable.

**Attestation:** In the context of Shielded VMs, the process by which a Host Guardian Service verifies that a Hyper-V host is trusted and authorized to run shielded VMs.

**Azure Arc:** A Microsoft service that extends Azure management, security, and governance to on-premises and multi-cloud infrastructure without migrating workloads to Azure.

**Azure Stack HCI:** Microsoft's hyper-converged infrastructure solution, built on Hyper-V and Failover Clustering, managed through Azure. The modern recommended platform for new HA Hyper-V deployments.

**BitLocker:** A full-disk encryption feature included in Windows. In Hyper-V, used to encrypt the host's physical drives (protecting VHDX files from physical theft) and as the encryption mechanism inside Shielded VMs.

**CAU (Cluster-Aware Updating):** A Windows Server Failover Clustering feature that automates Windows Updates across cluster nodes while maintaining high availability by draining and resuming nodes sequentially.

**Checkpoint / Snapshot:** A saved state of a virtual machine at a specific point in time. Allows rolling back the VM to that state. Not a substitute for backups -- checkpoints live on the same physical storage as the VM.

**Cluster Shared Volume (CSV):** Storage attached to a Windows Failover Cluster that all nodes can read and write simultaneously. VMs placed on CSVs can be managed by any node in the cluster and can be migrated between nodes without unmounting the storage.

**Credential Guard:** A Windows security feature that uses Hyper-V-based virtualization to isolate and protect credential hashes from the LSASS process. Prevents credential theft even if malware runs on the system.

**DDA (Discrete Device Assignment):** A Hyper-V feature that passes a physical PCIe hardware device (GPU, NVMe drive, network card) directly from the host to a VM, providing near-native performance. Requires the `-LocationPath` parameter with `Dismount-VMHostAssignableDevice`.

**Dynamic Memory:** A Hyper-V feature that automatically adjusts the RAM assigned to a VM based on its current demand, within configured minimum and maximum bounds. Improves host memory utilization when multiple VMs are running.

**Enhanced Session Mode:** A Hyper-V feature that uses an internal RDP connection for VM console sessions, enabling copy-paste, audio, and drive redirection between host and VM.

**EPT (Extended Page Tables):** Intel's implementation of SLAT (Second Level Address Translation). Required by Hyper-V. See also: NPT.

**Failover Clustering:** A Windows Server feature that groups multiple physical servers (nodes) into a cluster that monitors each other's health. If a node fails, its workloads (VMs) are automatically restarted on surviving nodes.

**Fixed-Size VHD/VHDX:** A virtual disk type that allocates the full specified amount of storage immediately when the file is created. Offers the best I/O performance. See also: Dynamically Expanding, Differencing.

**Generation 1 VM:** A Hyper-V virtual machine that uses legacy BIOS firmware emulation. Compatible with older operating systems. Uses IDE controllers for the boot drive.

**Generation 2 VM:** A Hyper-V virtual machine that uses UEFI firmware. Faster, more efficient, supports Secure Boot. Required for Shielded VMs and Nested Virtualization. Use Generation 2 for all modern operating systems.

**HGS (Host Guardian Service):** A Windows Server role that manages attestation and key protection for Shielded VMs. Verifies that Hyper-V hosts are trusted before allowing them to decrypt and run Shielded VMs.

**Heartbeat:** The regular signal exchanged between nodes in a Failover Cluster (or between a VM and the host via Integration Services) to confirm that each party is alive and responsive.

**Hypervisor:** Software or firmware that creates and manages virtual machines. Sits between hardware and guest operating systems, allocating resources to each VM.

**iSCSI:** A protocol for carrying SCSI storage commands over a TCP/IP network. Commonly used in Hyper-V clusters to provide shared block storage to all cluster nodes without dedicated Fibre Channel hardware.

**Integration Services:** Drivers and guest utilities provided by Hyper-V to improve VM performance and enable features like graceful shutdown, time synchronisation, live backup, and Enhanced Session Mode. Built into the Linux kernel since version 4.x; installed automatically in modern Windows VMs.

**LIS (Linux Integration Services):** The Linux-specific implementation of Hyper-V Integration Services. Included in the Linux kernel since version 4.x -- no manual installation required for modern distributions.

**Live Migration:** A Hyper-V feature that moves a running VM from one Hyper-V host to another with zero downtime. Classic Live Migration requires shared storage; Shared Nothing Live Migration (introduced Windows Server 2012) transfers both memory and disk without shared storage.

**NLA (Network Level Authentication):** An RDP security feature that requires the user to authenticate before a full remote desktop session is established. Reduces the attack surface of the RDP service by authenticating the connection before creating the full session.

**NPT (Nested Page Tables):** AMD's implementation of SLAT (Second Level Address Translation). Required by Hyper-V on AMD processors. See also: EPT.

**Nested Virtualization:** A Hyper-V feature that allows a VM to run Hyper-V inside itself, creating virtual machines within virtual machines. Useful for lab environments and training scenarios.

**NIC Teaming / SET (Switch Embedded Teaming):** A technique for combining multiple physical network adapters into a single logical interface for redundancy and/or load balancing. In Hyper-V, SET is the recommended implementation and is configured at the virtual switch level.

**NTFS Permissions:** Access controls applied to files and folders on NTFS-formatted drives. Used to restrict access to VHDX files on the host to authorized accounts only.

**Quorum:** The mechanism in Failover Clustering that determines whether the cluster has enough healthy nodes to continue operating safely. Prevents split-brain scenarios where two halves of a cluster operate independently.

**RBAC (Role-Based Access Control):** A security model where permissions are assigned to roles, and users are assigned to roles, rather than assigning permissions directly to users. In Hyper-V, the "Hyper-V Administrators" local group provides VM management rights without full server administrator access.

**Replica:** The Hyper-V Replica feature asynchronously replicates VMs from a primary host to a replica host for disaster recovery. Supports replication intervals of 30 seconds, 5 minutes, or 15 minutes (30 seconds requires Windows Server 2012 R2+).

**S2D / Storage Spaces Direct:** A Windows Server Datacenter feature that pools local storage from multiple Hyper-V hosts in a cluster into a shared, redundant storage pool. Eliminates the need for dedicated SAN hardware in hyper-converged deployments.

**SCVMM (System Center Virtual Machine Manager):** A licensed Microsoft enterprise management product for virtualized datacenters. Manages multiple Hyper-V hosts, provides VM templates, a self-service portal, and centralized lifecycle management. Requires System Center licensing.

**Secure Boot:** A UEFI firmware security feature that verifies the digital signature of bootloader software before executing it. Prevents unauthorized or malicious code from running during the boot process. Available on Generation 2 VMs only.

**SET (Switch Embedded Teaming):** See NIC Teaming / SET.

**Shared Nothing Live Migration:** Live Migration without shared storage. Both the VM's memory and its disk files are transferred to the destination host. Slower than classic Live Migration but available in environments without SAN infrastructure.

**Shielded VM:** A Hyper-V VM that uses BitLocker encryption and requires Host Guardian Service attestation to run. The VM's data is inaccessible to unauthorized hosts or administrators. Requires Windows Server Datacenter edition.

**SLAT (Second Level Address Translation):** A hardware CPU feature required by Hyper-V. Manages memory address translation between VMs and physical memory. Intel calls it EPT; AMD calls it NPT. Present in all modern processors since approximately 2010.

**Split-Brain:** A failure mode in clustered systems where two halves of a cluster both believe the other half has failed and both try to take control simultaneously. In Hyper-V clusters, this could cause two nodes to try to run the same VM, corrupting data. Quorum prevents split-brain.

**Type 1 Hypervisor:** A hypervisor that runs directly on the physical hardware (bare-metal). No underlying operating system. Examples: Hyper-V, VMware ESXi, Proxmox VE. Hyper-V is a Type 1 hypervisor even on Windows 10/11 -- Windows itself runs as a guest.

**Type 2 Hypervisor:** A hypervisor that runs as an application on top of an existing operating system. Examples: VirtualBox, VMware Workstation, Parallels. Suitable for development and testing on personal hardware.

**VHD (Virtual Hard Disk):** The older Hyper-V virtual disk format. Maximum 2 TB. Susceptible to corruption from unexpected power loss. Use VHDX for all new deployments.

**VHDX (Virtual Hard Disk Extended):** The modern Hyper-V virtual disk format. Maximum 64 TB. Resilient to corruption from unexpected power loss. Better alignment with modern storage hardware. Introduced with Windows Server 2012.

**VLAN (Virtual Local Area Network):** A logical network segment defined at the switch level. VMs can be assigned to a VLAN by setting a VLAN ID on their network adapter, providing segmentation without requiring separate physical switches.

**VM (Virtual Machine):** A software emulation of a physical computer. Runs an operating system and applications in complete isolation from other VMs on the same host.

**VMBus:** The high-speed internal communication channel between the Hyper-V hypervisor and guest VMs. Used by synthetic devices (network adapters, SCSI controllers) for efficient I/O without hardware emulation overhead.

**VT-x:** Intel Virtualization Technology for IA-32/64 processors. Required for running Hyper-V on Intel hardware. Must be enabled in BIOS/UEFI. See also: AMD-V.

**AMD-V / SVM:** AMD's hardware virtualization technology. Functionally equivalent to Intel VT-x. Called "SVM Mode" in some BIOS menus. Must be enabled for Hyper-V on AMD hardware.

**VSS (Volume Shadow Copy Service):** A Windows framework for creating consistent point-in-time snapshots of data while applications continue running. Used by Hyper-V backup solutions to create application-consistent VM backups without downtime.

**WAC (Windows Admin Center):** A free, browser-based management tool from Microsoft for Windows Server and Hyper-V. Replaces the need for multiple separate management tools (Hyper-V Manager, Server Manager, etc.).

**Windows Server Core:** A Windows Server installation option with no graphical interface. Smaller attack surface, lower memory footprint, fewer required updates. Managed via PowerShell or remotely through Windows Admin Center.
