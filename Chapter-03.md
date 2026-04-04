# Chapter 3: Networking in Hyper-V

Networking is where a lot of Hyper-V setups go wrong, and it's also where things get genuinely interesting. Get this chapter right and you can build sophisticated multi-VM networks on a single laptop. Get it wrong and your VMs just sit there with no network connectivity wondering what happened.

## 3.1 How Virtual Networking Works

In a physical network, you have network interface cards (NICs) plugged into physical switches, which forward traffic between devices. Hyper-V recreates this model in software.

A **virtual switch** is a software-defined switch that exists entirely inside your Hyper-V host. VMs connect to virtual switches via **virtual network adapters** -- software NICs that appear to the guest OS as real network hardware. The virtual switch handles forwarding traffic between connected VMs and, depending on the switch type, to the physical network.

One important concept: when you create a virtual switch that connects to your physical NIC, your physical NIC effectively becomes part of the virtual switch. The host machine's network connectivity flows through the virtual switch rather than directly through the physical NIC. This is invisible in normal operation but explains why creating an External switch can briefly disrupt your host's network connection.

## 3.2 The Three Types of Virtual Switch

Hyper-V gives you three switch types. Choosing the right one is the single most important networking decision you'll make.

### External Virtual Switch

An External switch binds to one of your physical network adapters. VMs connected to it are visible on your physical network -- they can reach other devices on your LAN, reach the internet, and be reached from other machines.

Use this when VMs need full network access: web servers, domain controllers, anything that other machines need to talk to.

> The host machine can also communicate through the External switch if you leave the "Allow management operating system to share this network adapter" checkbox enabled during creation (recommended).

### Internal Virtual Switch

An Internal switch creates a private network that VMs can use to talk to each other **and to the Hyper-V host**. Traffic on an Internal switch cannot reach the physical network or the internet.

Use this when you want VMs to communicate with the host (for file sharing, management, etc.) but don't need external access. Also useful as a second NIC on VMs that have an External switch for internet access but also need a management network.

### Private Virtual Switch

A Private switch is completely isolated -- VMs on it can only communicate with other VMs connected to the same private switch. The host machine cannot communicate with them at all.

Use this for test environments that need to be isolated from everything else. Two VMs running a penetration test, a cluster simulation, or a replica environment are all good candidates.

**Quick comparison:**

| Switch Type | VM to VM | VM to Host | VM to Physical Network |
|---|---|---|---|
| External | Yes | Yes | Yes |
| Internal | Yes | Yes | No |
| Private | Yes | No | No |

Here's how those three types look in practice:

```
EXTERNAL SWITCH
  Physical Network (router, internet, other devices)
        |
  +-----+------+   <- physical NIC (shared with switch)
  | Hyper-V    |
  |   Host     |
  +-----+------+
        |  External vSwitch
   +----+----+
  VM1       VM2     <- VMs visible on physical LAN

INTERNAL SWITCH
  +-------------+
  | Hyper-V     |
  |   Host      |   <- host can communicate with VMs
  +------+------+
         |  Internal vSwitch
    +----+----+
   VM1       VM2     <- VMs talk to each other and the host; no physical LAN access

PRIVATE SWITCH
  +-------------+
  | Hyper-V     |
  |   Host      |   <- host CANNOT communicate with VMs
  +-------------+
         |  Private vSwitch
    +----+----+
   VM1       VM2     <- VMs talk only to each other; completely isolated
```

## 3.3 Creating a Virtual Switch

1. In Hyper-V Manager, click **Virtual Switch Manager** in the right pane.
2. Select the switch type you want to create.
3. Click **Create Virtual Switch**.
4. Give it a descriptive name (e.g., `External-LAN`, `Internal-Lab`, `Private-Test`).
5. For an External switch, select which physical network adapter to bind to from the dropdown.
6. Click **Apply**, then **OK**.

> **Creating External switches may cause a brief network interruption** on your host (usually 5-15 seconds) while Windows reconfigures the NIC binding. This is normal.

**Via PowerShell:**
```powershell
# Create an External switch using the adapter named "Ethernet"
New-VMSwitch -Name "External-LAN" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Create an Internal switch
New-VMSwitch -Name "Internal-Lab" -SwitchType Internal

# Create a Private switch
New-VMSwitch -Name "Private-Test" -SwitchType Private
```

## 3.4 An Important Note About DHCP on Internal and Private Switches

This trips up almost every beginner: **Internal and Private switches have no built-in DHCP server.** If you connect a VM to an Internal or Private switch and configure it to "obtain an IP address automatically," it will sit with an APIPA address (169.254.x.x) and no network connectivity.

To use DHCP on an Internal or Private switch, you need one of these:

**Option A (simplest): Assign static IP addresses** inside each VM. Go into the VM's network settings and manually set an IP, subnet mask, and (if needed) gateway. VMs on the same switch can communicate as long as they're in the same subnet.

**Option B: Run a DHCP server VM on the same switch.** A Windows Server VM configured as a DHCP server on the Private/Internal switch will hand out addresses to other VMs on that switch.

**Option C (Internal switches only): Configure Internet Connection Sharing on the host.** This lets the host share its external network connection with the Internal switch via NAT, and includes a basic DHCP service. It's messy to set up and not recommended for production, but works for simple lab scenarios.

For External switches, your physical network's DHCP server (your router) handles IP assignment automatically.

## 3.5 Connecting a VM to a Virtual Switch

1. Right-click the VM in Hyper-V Manager and select **Settings**.
2. Click **Network Adapter** in the left pane.
3. In the right pane, select your virtual switch from the dropdown.
4. Click **Apply** and **OK**.

You can add multiple network adapters to a VM (select **Add Hardware > Network Adapter** in the VM settings). This is common for server VMs that need a management interface on an Internal switch and a service interface on an External switch.

**Via PowerShell:**
```powershell
# Connect an existing adapter to a switch
Connect-VMNetworkAdapter -VMName "MyVM" -SwitchName "External-LAN"

# Add a new network adapter and connect it
Add-VMNetworkAdapter -VMName "MyVM" -SwitchName "Internal-Lab"
```

## 3.6 Configuring IP Addresses Inside the VM

Once a VM is connected to an External switch, your existing DHCP server will assign it an IP automatically. For Internal and Private switches (or if you prefer static IPs), configure them inside the VM.

### Windows (inside the VM)

1. Open **Network and Sharing Center** (or Settings > Network & Internet).
2. Click on the network adapter.
3. Click **Properties**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
5. Enter your IP address, subnet mask, and gateway.

### Linux (inside the VM)

For modern distributions using `netplan` (Ubuntu 18.04+):
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.100.10/24
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
Then apply with `sudo netplan apply`.

For distributions using `nmcli` (RHEL, Rocky Linux, AlmaLinux):
```bash
nmcli con mod "ens3" ipv4.addresses 192.168.100.10/24
nmcli con mod "ens3" ipv4.gateway 192.168.100.1
nmcli con mod "ens3" ipv4.method manual
nmcli con up "ens3"
```

## 3.7 Advanced Networking Features

### VLAN Configuration

VLANs (Virtual Local Area Networks) let you segment network traffic for security and organisational purposes. In Hyper-V, you can assign a VLAN ID to a VM's network adapter.

In VM Settings > Network Adapter > Advanced Features, enable VLAN identification and enter the VLAN ID. VMs on the same VLAN communicate as if on the same physical segment; VMs on different VLANs are isolated unless a router/firewall handles inter-VLAN routing.

Via PowerShell:
```powershell
Set-VMNetworkAdapterVlan -VMName "MyVM" -VlanId 10 -Access
```

### Bandwidth Management

You can limit how much network bandwidth a VM can consume, preventing one busy VM from starving others.

In VM Settings > Network Adapter > Advanced Features, set **Minimum bandwidth** (guaranteed allocation) and **Maximum bandwidth** (hard cap) in Mbps.

Via PowerShell:
```powershell
Set-VMNetworkAdapter -VMName "MyVM" -MaximumBandwidth 100MB
```

### Network Adapter Port ACLs

You can control network traffic at the virtual switch port level using Access Control Lists (ACLs). This is useful for blocking specific traffic from reaching a VM without a full firewall.

```powershell
# Block all inbound traffic from a specific IP
Add-VMNetworkAdapterAcl -VMName "MyVM" `
    -Direction Inbound `
    -Action Deny `
    -RemoteIPAddress "192.168.1.100"

# Remove that rule
Remove-VMNetworkAdapterAcl -VMName "MyVM" `
    -Direction Inbound `
    -Action Deny `
    -RemoteIPAddress "192.168.1.100"
```

### NIC Teaming with Switch Embedded Teaming (SET)

> **Enterprise / multi-NIC hosts only.** This feature requires a host with multiple physical network adapters.

Switch Embedded Teaming (SET) lets you combine multiple physical NICs into a single virtual switch, providing both redundancy (if one NIC fails, the other takes over) and increased bandwidth through load balancing. SET replaced the older NIC Teaming approach and is the recommended method.

```powershell
New-VMSwitch -Name "TeamedSwitch" `
    -NetAdapterName "Ethernet1","Ethernet2" `
    -EnableEmbeddedTeaming $true
```

### Disabling RSC (Receive Segment Coalescing)

RSC reduces CPU overhead by merging multiple incoming TCP segments. In most cases, leave it enabled. If you're troubleshooting specific network performance or compatibility issues, disable it per-switch:

```powershell
Set-VMSwitch -Name "External-LAN" -EnableSoftwareRsc $false
```

## 3.8 Troubleshooting Network Issues

The most common Hyper-V networking problems and how to fix them:

**VM has no IP address on an Internal/Private switch:** Expected behaviour -- no DHCP server exists on these switch types. Assign a static IP manually (see section 3.6).

**VM connected to External switch has no internet:** Check that the virtual switch is bound to the correct physical adapter. In Virtual Switch Manager, confirm the External switch shows the right NIC. Also confirm the host itself has internet access on that adapter.

**VM has an IP but can't reach other devices:** Check the VM's subnet mask. A common mistake is assigning an IP in the right range but with a wrong subnet mask, causing the VM to think devices it should be able to reach directly are on a different network.

**Two VMs on the same switch can't reach each other:** Verify both are connected to the same virtual switch (check VM Settings > Network Adapter for each). If using VLANs, verify both are on the same VLAN. Check firewall rules inside each VM -- Windows Firewall may be blocking ICMP (ping) by default.

**Useful diagnostic commands inside VMs:**
```powershell
# Check IP configuration
ipconfig /all

# Test connectivity to another VM
ping 192.168.1.10

# Trace the network path
tracert 192.168.1.10

# Check if a port is open on another VM
Test-NetConnection -ComputerName 192.168.1.10 -Port 80
```

---

> **Key Takeaways**
> - External switch: VMs join your physical network and the internet
> - Internal switch: VMs talk to each other and the host, but not outside
> - Private switch: VMs only talk to each other -- completely isolated
> - Internal and Private switches have no DHCP -- assign static IPs or run your own DHCP server
> - You can add multiple network adapters to a single VM for different network segments
> - VLAN IDs, bandwidth limits, and port ACLs are all configurable per-VM-adapter

---

Storage is the third leg of the foundation -- get it wrong and performance suffers or VMs run out of space at the worst possible moment. Chapter 4 covers VHD vs VHDX, fixed vs dynamically expanding disks, how to expand a disk after the fact, and how to get the best performance from your storage configuration.

---

## Exercise 3: Build a Three-Switch Lab Network

> **Estimated time: 45--60 minutes**

This exercise creates the network topology you'll use throughout the rest of the guide.

1. Create three virtual switches: one External (bound to your physical NIC), one Internal named `Lab-Internal`, one Private named `Lab-Private`.
2. Create two VMs. Give each VM two network adapters: one connected to `External-LAN`, one connected to `Lab-Internal`.
3. Start both VMs. The External adapters should automatically get DHCP addresses from your router. Assign static IPs on the `Lab-Internal` adapters (e.g., 10.0.0.1/24 on VM1, 10.0.0.2/24 on VM2).
4. Verify: VM1 and VM2 can ping each other using the 10.0.0.x addresses.
5. Verify: Both VMs can reach the internet through their External adapters.
6. Create a third VM. Give it only a `Lab-Private` adapter with a static IP of 192.168.99.10/24.
7. Verify: VM3 cannot ping either of the first two VMs (different switch, no route between them).
