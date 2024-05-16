# Chapter 5: Advanced Hyper-V Features

## 5.1 Introduction to Advanced Features

Hyper-V offers several advanced features to enhance virtualization capabilities, improve performance, and ensure high availability. This chapter will cover some of the key advanced features.

## 5.2 Hyper-V Replica

### What is Hyper-V Replica?
Hyper-V Replica is a disaster recovery solution that allows you to replicate VMs from one Hyper-V host to another. It ensures business continuity by enabling failover to a secondary site in case of a primary site failure.

### Requirements:
- **Windows Server Edition:** Hyper-V Replica is available in Windows Server editions (Standard and Datacenter).
- **Domain Membership:** Both primary and replica hosts must be part of the same Active Directory domain or have a trust relationship between domains.
- **Network Configuration:** Reliable network connectivity between the primary and replica sites.
- **Storage:** Sufficient storage on both the primary and replica hosts to accommodate replicated VMs.

### Setting Up Hyper-V Replica:

1. **Enable Replication:**
   - On the primary host, open **Hyper-V Manager**, right-click the VM, and select **Enable Replication**.

2. **Specify Replica Server:**
   - Enter the name of the secondary Hyper-V host.

3. **Configure Connection Settings:**
   - Choose the authentication method (Kerberos or Certificate-based).
   - Specify the compression settings if needed.

4. **Select VHDs to Replicate:**
   - Choose which virtual hard disks to replicate.

5. **Configure Replication Frequency:**
   - Select the replication interval (30 seconds, 5 minutes, or 15 minutes).

6. **Choose Initial Replication Method:**
   - Select how the initial copy of the VM will be transferred (over the network, using external media, etc.).

7. **Complete the Wizard:**
   - Review the settings and click **Finish**.

![Hyper-V Replica Setup](https://www.nakivo.com/blog/wp-content/uploads/2017/12/How-to-enable-replication-in-Hyper-V.webp)

## 5.3 Live Migration

### What is Live Migration?
Live Migration allows you to move running VMs between Hyper-V hosts without downtime. This feature is useful for load balancing, maintenance, and minimizing downtime.

### Requirements:
- **Windows Server Edition:** Live Migration is available in Windows Server editions (Standard and Datacenter).
- **Domain Membership:** Hosts must be part of the same Active Directory domain.
- **Network Configuration:** Dedicated network for live migration traffic to ensure performance.
- **Shared Storage:** Common storage accessible by all participating Hyper-V hosts, such as iSCSI or SMB-based storage.

### Setting Up Live Migration:

1. **Configure Hosts:**
   - On each Hyper-V host, open **Hyper-V Manager**, click on the host name, and select **Hyper-V Settings**.
   - Go to **Live Migrations** and check **Enable incoming and outgoing live migrations**.

2. **Configure Authentication Protocol:**
   - Choose **CredSSP** or **Kerberos** for authentication.

3. **Configure Networks:**
   - Specify the networks to be used for live migration.

4. **Move VM:**
   - Right-click the running VM, select **Move**, and follow the wizard to transfer the VM to another host.

![Live Migration](https://www.ubackup.com/screenshot/en/acb/virtual-machine/hyper-v-live-migrations/open-hyper-v-move-wizard.png)

## 5.4 Nested Virtualization

### What is Nested Virtualization?
Nested Virtualization allows you to run Hyper-V inside a VM. This feature is useful for testing and development environments where you need to simulate a complete Hyper-V setup.

### Enabling Nested Virtualization:

1. **Enable on the Host:**
   - Open PowerShell as an administrator and run the following command:
     ```powershell
     Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
     ```

2. **Configure the VM:**
   - Inside the nested VM, install Hyper-V as you would on a physical machine.

![Nested Virtualization](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/media/hvnesting.png)

## 5.5 Shielded VMs

### What are Shielded VMs?
Shielded VMs provide enhanced security by protecting VMs from unauthorized access and tampering. They use BitLocker encryption and a Host Guardian Service (HGS) to ensure that only authorized hosts can run the VMs.

### Requirements:
- **Windows Server Datacenter Edition:** Shielded VMs require the Datacenter edition.
- **Host Guardian Service (HGS):** A properly configured HGS environment.
- **Generation 2 VMs:** Only Generation 2 VMs support shielding.

### Setting Up Shielded VMs:

1. **Deploy Host Guardian Service (HGS):**
   - Install and configure HGS in your environment.

2. **Prepare the VM:**
   - Create a Generation 2 VM with TPM support.

3. **Shield the VM:**
   - Use **Shielded VM Tools** to convert an existing VM or create a new shielded VM.

![Shielded VMs](https://i0.wp.com/cloudbase.it/wp-content/uploads/2016/04/vm.jpg?ssl=1)

## 5.6 Discrete Device Assignment (DDA)

### What is Discrete Device Assignment?
Discrete Device Assignment allows you to assign physical hardware devices directly to a VM. This feature provides near-native performance for hardware resources.

### Requirements:
- **Compatible Hardware:** Ensure the device supports DDA.
- **Windows Server Edition:** DDA is available in Windows Server editions (Standard and Datacenter).
- **Physical Device Availability:** The device must not be in use by the host.

### Enabling DDA:

1. **Prepare the Device:**
   - Ensure the device is compatible with DDA and is not in use by the host.

2. **Dismount the Device:**
   - Open PowerShell and run the following command to dismount the device:
     ```powershell
     Dismount-VMHostAssignableDevice -Force
     ```

3. **Assign the Device to a VM:**
   - Use PowerShell to assign the device to a VM:
     ```powershell
     Add-VMAssignableDevice -VMName <VMName> -LocationPath <DeviceLocationPath>
     ```

![Discrete Device Assignment](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/deploy/media/dda-devicemanager.png)

## 5.7 Storage Spaces Direct (S2D)

### What is Storage Spaces Direct?
Storage Spaces Direct (S2D) is a feature that enables you to create highly available storage using local disks on Hyper-V hosts. It combines the storage resources of multiple hosts into a single storage pool.

### Requirements:
- **Windows Server Datacenter Edition:** S2D requires the Datacenter edition.
- **Clustered Environment:** Multiple Hyper-V hosts in a Failover Cluster.
- **Compatible Storage:** NVMe, SSD, or HDD devices compatible with S2D.

### Setting Up S2D:

1. **Prepare the Hosts:**
   - Ensure each host has compatible storage devices.

2. **Enable S2D:**
   - Open PowerShell and run the following command:
     ```powershell
     Enable-ClusterS2D
     ```

3. **Create Storage Pool:**
   - Use **Failover Cluster Manager** or PowerShell to create a storage pool from the available disks.

4. **Create Volumes:**
   - Create virtual disks and volumes from the storage pool.
