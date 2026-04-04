# Chapter 5: Advanced Hyper-V Features

The features in this chapter are where Hyper-V moves from "useful virtualisation tool" to "serious infrastructure platform." Hyper-V Replica gives you disaster recovery. Live Migration lets you move running VMs with zero downtime. Nested Virtualization lets you run a hypervisor inside your hypervisor. Each section includes a clear note about what's needed to use the feature, so you can quickly identify what's relevant for your environment.

## 5.1 Feature Requirements at a Glance

Before diving into the details, here's what each feature requires:

| Feature | Required Edition | Multiple Hosts Needed? | Notes |
|---|---|---|---|
| Hyper-V Replica | Standard or Datacenter | Yes (2 hosts) | Hosts must be in same AD domain or trusted domains |
| Live Migration (shared storage) | Standard or Datacenter | Yes (2+ hosts) | Requires shared storage (iSCSI, SMB, SAN) |
| Shared Nothing Live Migration | Standard or Datacenter | Yes (2+ hosts) | No shared storage required |
| Nested Virtualization | Standard or Datacenter | No | Works on a single host; VM must be Gen 2 |
| Shielded VMs | Datacenter only | No (HGS required) | Full HGS infrastructure needed |
| Discrete Device Assignment | Standard or Datacenter | No | Hardware must support DDA; check compatibility first |
| Storage Spaces Direct | Datacenter only | Yes (2+ hosts) | Requires a Failover Cluster |

## 5.2 Hyper-V Replica

Hyper-V Replica continuously replicates VMs from a primary Hyper-V host to a replica host. If your primary host fails, you can fail over to the replica and bring the VM back up quickly -- without waiting for a full restore from backup.

### How It Works

Replica uses asynchronous replication: the primary VM runs normally, and changes are sent to the replica host at regular intervals (30 seconds, 5 minutes, or 15 minutes). The replica host maintains a copy of the VM at the last synchronisation point. In a failover, the replica host starts its copy of the VM from that point.

> **Note on the 30-second interval:** This option requires Windows Server 2012 R2 or later on both hosts. If either host is running Windows Server 2012 (original), only the 5-minute and 15-minute options are available.

### Requirements

- Both hosts must be in the same Active Directory domain, or in domains with a trust relationship.
- Both hosts must have the Hyper-V role installed.
- Reliable network connectivity between the hosts (the replica traffic uses HTTP port 80 or HTTPS port 443).
- Enough storage on the replica host for the replicated VM files.

### Setting Up Hyper-V Replica

**Step 1: Configure the replica host to accept replication**

On the *replica* host (the one that will receive the copy):

1. Open **Hyper-V Manager** and click on the host name in the left pane.
2. Click **Hyper-V Settings** in the right pane.
3. Select **Replication Configuration**.
4. Check **Enable this computer as a Replica server**.
5. Choose your authentication method:
   - **Kerberos (HTTP):** Easier to set up within a domain but replication traffic is unencrypted.
   - **Certificate-based (HTTPS):** Requires certificates but encrypts replication traffic. Recommended.
6. Choose to allow replication from any authenticated server, or specify which servers are allowed.
7. Set the default storage location for replica VMs.
8. Click **Apply**.

**Step 2: Open the firewall on the replica host**

```powershell
# For Kerberos/HTTP
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTP Listener (TCP-In)"

# For Certificate/HTTPS
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTPS Listener (TCP-In)"
```

**Step 3: Enable replication on the primary VM**

On the *primary* host:

1. Right-click the VM you want to replicate > **Enable Replication**.
2. The Enable Replication Wizard opens. Specify the replica server's name.
3. Choose authentication (must match what you configured on the replica host).
4. Select which VHDX files to replicate (typically all of them).
5. Choose the replication frequency.
6. Choose the initial replication method: over the network (for small VMs) or via external media (for large VMs where network transfer would take too long).
7. Complete the wizard.

**Testing failover (without disrupting the primary):**

```powershell
# Perform a test failover
Start-VMFailover -VMName "MyVM" -ComputerName "ReplicaHost" -AsTest

# Stop the test and clean up
Stop-VMFailover -VMName "MyVM" -ComputerName "ReplicaHost"
```

## 5.3 Live Migration

Live Migration moves a running VM from one Hyper-V host to another -- while it's running, with users connected to it, with zero downtime. The VM's memory pages are transferred to the destination host while the VM keeps running, and the final cutover takes milliseconds.

### Two Types of Live Migration

**Classic Live Migration (with shared storage):** The VM files (VHDXs) stay on shared storage (iSCSI, SMB file share, SAN) that both hosts can access. Only the VM's memory state is transferred during migration. Fast and the most common enterprise implementation.

**Shared Nothing Live Migration:** Introduced in Windows Server 2012. Both the VM's memory *and* its disk files are migrated to the destination host. No shared storage required. Slower than classic Live Migration (because it's transferring potentially hundreds of GB of disk data) but works in environments where shared storage isn't available -- including home labs with two physical machines.

### Requirements

- Both hosts must be in the same Active Directory domain (or trusted domains).
- If using Kerberos authentication: both hosts must be able to contact a domain controller.
- If using CredSSP authentication: you need to run the migration from one of the hosts (not from a remote management station).
- For classic Live Migration: shared storage accessible from both hosts.
- For Shared Nothing Live Migration: just network connectivity between the hosts.

### Setting Up Live Migration

On each Hyper-V host:

1. In Hyper-V Manager, click on the host name > **Hyper-V Settings**.
2. Under **Live Migrations**, check **Enable incoming and outgoing live migrations**.
3. Choose **Use Kerberos** (if you don't mind managing SPN delegation) or **Use CredSSP** (simpler to set up).
4. Under **Advanced Features**, specify which network interfaces to use for live migration traffic (use a dedicated high-speed interface if available, not the same NIC as your VM traffic).

**Performing a Live Migration:**

1. Right-click the running VM in Hyper-V Manager.
2. Select **Move**.
3. Choose **Move the virtual machine** (for classic LM with shared storage) or **Move the virtual machine and its storage** (for Shared Nothing LM).
4. Specify the destination host.
5. Click **Move**. Watch the progress bar -- the VM keeps running throughout.

Via PowerShell:
```powershell
# Classic Live Migration (shared storage)
Move-VM -Name "MyVM" -DestinationHost "Server02"

# Shared Nothing Live Migration
Move-VM -Name "MyVM" `
        -DestinationHost "Server02" `
        -DestinationStoragePath "D:\VMs"
```

## 5.4 Nested Virtualization

Nested Virtualization lets you run Hyper-V inside a Hyper-V virtual machine. The VM can then create and manage its own VMs. This is particularly useful for:

- Testing Hyper-V configurations without a second physical host
- Building Hyper-V training environments
- Simulating cluster environments on a single machine
- Running Azure Stack HCI evaluation scenarios

### Requirements and Important Caveats

Before enabling Nested Virtualization, be aware of these requirements -- getting any of them wrong will cause failures:

1. The VM must be a **Generation 2** VM.
2. The VM must be **powered off** when you enable nested virtualisation.
3. **Dynamic Memory must be disabled** on the VM running nested Hyper-V (static RAM allocation only).
4. The VM's network adapters need **MAC Address Spoofing enabled** for nested VMs to have network connectivity.
5. Both host and nested VM must be running Windows Server 2016 or later, or Windows 10 version 1607 or later.

### Enabling Nested Virtualization

On the **host machine**, run these commands (the VM must be powered off):

```powershell
# Enable nested virtualisation
Set-VMProcessor -VMName "NestedHVHost" -ExposeVirtualizationExtensions $true

# Disable Dynamic Memory (required for nested Hyper-V)
Set-VMMemory -VMName "NestedHVHost" -DynamicMemoryEnabled $false

# Enable MAC Address Spoofing on each network adapter
# (required for nested VMs to have network access)
Get-VMNetworkAdapter -VMName "NestedHVHost" | Set-VMNetworkAdapter -MacAddressSpoofing On
```

Then start the VM, install the Hyper-V role inside it, and you can create VMs within that VM.

## 5.5 Shielded VMs

> **Requires: Windows Server Datacenter edition + Host Guardian Service infrastructure**

Shielded VMs protect virtual machines from unauthorized access and tampering -- including from Hyper-V administrators on the host. They're designed for scenarios where the VM owner doesn't fully trust the people who manage the physical infrastructure (common in hosting provider environments).

### How Shielded VMs Work

A Shielded VM's disk is encrypted with BitLocker. The encryption keys are managed by a **Host Guardian Service (HGS)** server. The VM will only start on Hyper-V hosts that HGS has verified as "guarded" -- hosts that are joined to the guarded fabric and have passed attestation. An administrator on an unguarded host cannot access the VM's data even if they copy the VHDX file.

### Two Protection Levels

**Shielded:** Full BitLocker encryption + strict HGS attestation. The VM cannot run on any host that hasn't been approved by HGS. Maximum security.

**Encryption Supported:** BitLocker encryption, but with relaxed attestation requirements. The VM can still run on hosts that aren't fully guarded. Useful for environments that want encryption-at-rest without the full HGS infrastructure overhead.

For most environments that want disk encryption, "Encryption Supported" provides strong protection with significantly less operational complexity.

### Requirements

- Windows Server 2016 Datacenter edition or later on both the Hyper-V host and the guest OS.
- Host Guardian Service (HGS) deployed on at least one separate server (not the Hyper-V host itself).
- Generation 2 VMs only.
- TPM (Trusted Platform Module) support on the host hardware.

Deploying HGS is a multi-step process involving Active Directory integration or a standalone HGS forest. For full deployment instructions, see [Microsoft's HGS documentation](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-and-shielded-vms).

## 5.6 Discrete Device Assignment (DDA)

DDA passes a physical hardware device from the host directly to a VM, giving the VM near-native access to the hardware. Common uses:

- Passing a GPU to a VM for compute workloads or graphics acceleration
- Passing an NVMe SSD to a VM for maximum storage performance
- Passing a network card to a VM for specialised networking features

### Requirements

- The hardware device must support DDA (check the manufacturer's documentation -- most modern PCIe devices do).
- The device must not be in use by the host.
- Windows Server Standard or Datacenter edition.

### Assigning a Device to a VM

**Step 1: Find and prepare the device on the host**

```powershell
# Find the device's location path
# Look for your device in the list
Get-PnpDevice | Where-Object {$_.InstanceId -like "PCI*"} | Select-Object FriendlyName, InstanceId

# Disable the device in Windows (so the host releases it)
Disable-PnpDevice -InstanceId "PCI\VEN_XXXX&DEV_XXXX..."  -Confirm:$false

# Get the location path needed for DDA
$locationPath = (Get-PnpDeviceProperty -InstanceId "PCI\VEN_XXXX&DEV_XXXX..." `
                                       -KeyName DEVPKEY_Device_LocationPaths).Data[0]

# Dismount the device from the host
Dismount-VMHostAssignableDevice -LocationPath $locationPath -Force
```

> **Important:** The `Dismount-VMHostAssignableDevice` command requires the `-LocationPath` parameter. The command will fail without it.

**Step 2: Assign the device to the VM (VM must be powered off)**

```powershell
Add-VMAssignableDevice -VMName "MyVM" -LocationPath $locationPath
```

**Step 3: Start the VM.** The device appears inside the VM as if physically installed. Install the manufacturer's drivers inside the VM.

**To return the device to the host:**
```powershell
Remove-VMAssignableDevice -VMName "MyVM" -LocationPath $locationPath
Mount-VMHostAssignableDevice -LocationPath $locationPath
```

---

> **Key Takeaways**
> - Hyper-V Replica provides async DR replication to a second host -- good enough for many SMB disaster recovery needs
> - Live Migration requires shared storage; Shared Nothing Live Migration doesn't -- but transfers both memory and disk
> - Nested Virtualization requires: Gen 2 VM, powered off to enable, static RAM (no Dynamic Memory), MAC Address Spoofing on
> - Shielded VMs encrypt the VM with BitLocker via HGS -- Datacenter edition only; "Encryption Supported" is the lighter alternative
> - DDA: the `Dismount-VMHostAssignableDevice` command requires a `-LocationPath` parameter or it will fail

---

Advanced features add real capability, but they also add complexity -- a Replica relationship or Live Migration gone wrong can leave VMs in unexpected states. Before you run any of this in production, you need a recovery plan for when something goes sideways. Chapter 6 covers backup and restore: not glamorous, but the chapter you'll be glad you read when something goes wrong.

---

## Exercise 5: Enable Nested Virtualization

> **Estimated time: 60--90 minutes** (requires a running Windows Server VM and some patience -- Hyper-V inside a VM can be slow to initialize)

If you have a Windows Server VM available:

1. Power off the VM.
2. On the host, run `Set-VMProcessor -VMName "YourVM" -ExposeVirtualizationExtensions $true`.
3. Disable Dynamic Memory: `Set-VMMemory -VMName "YourVM" -DynamicMemoryEnabled $false`. Set a static startup RAM of 4 GB.
4. Enable MAC Address Spoofing: `Get-VMNetworkAdapter -VMName "YourVM" | Set-VMNetworkAdapter -MacAddressSpoofing On`.
5. Start the VM. Inside the VM, open PowerShell as Administrator and run: `Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart`.
6. After the restart, open Hyper-V Manager inside the VM. Confirm you can see the nested Hyper-V host.
7. Create a small VM inside the nested Hyper-V (even just a tiny Linux VM) to confirm nested virtualisation is working.
