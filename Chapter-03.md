# Chapter 3: Networking in Hyper-V

## 3.1 Introduction to Virtual Networking

In Hyper-V, virtual networking allows VMs to communicate with each other and with the physical network. Hyper-V uses virtual switches to facilitate this communication.

## 3.2 Types of Virtual Switches

### External Virtual Switch:
- Connects VMs to the physical network.
- Allows communication between VMs and external network devices.

### Internal Virtual Switch:
- Connects VMs to each other and to the host machine.
- No connection to the external network.

### Private Virtual Switch:
- Connects VMs only to each other.
- No connection to the host machine or external network.

## 3.3 Creating a Virtual Switch

### Step-by-Step Guide to Create a Virtual Switch:

1. **Open Hyper-V Manager:**
   - Open **Hyper-V Manager** from the Start menu or Server Manager.

2. **Access Virtual Switch Manager:**
   - In the right pane, click on **Virtual Switch Manager**.

3. **Select Switch Type:**
   - Choose **External**, **Internal**, or **Private** and click **Create Virtual Switch**.

4. **Configure Switch Settings:**
   - **Name:** Enter a name for the virtual switch.
   - **External Network:** If creating an external switch, select the network adapter to bind to.
   - **Internal/Private Network:** No additional settings are required.

5. **Apply Settings:**
   - Click **Apply** and then **OK**.

![Virtual Switch Manager](https://img.veeam.com/hyperv_portal/blog/vSwitch-Manager.jpg)

## 3.4 Connecting VMs to a Virtual Switch

### Assigning a Network Adapter to a VM:

1. **Open VM Settings:**
   - In Hyper-V Manager, right-click the VM and select **Settings**.

2a. **Add Network Adapter:**
   - In the left pane, click **Add Hardware** and select **Network Adapter**.
   - Click **Add**.

2b. **Select Network Adapter:**
   - In the left pane, click **Network Adapter**.

3. **Select Virtual Switch:**
   - In the right pane, select the desired virtual switch from the dropdown list.
   - Click **Apply** and then **OK**.

![VM Network Adapter Settings](https://handsontek.net/images/hot2/Hyper-V/Add%20Virtual%20Switch.png)

## 3.5 Configuring Network Settings Inside the VM

### Setting Up IP Addresses:

1. **Static IP Configuration:**
   - Open **Network and Sharing Center** inside the VM.
   - Click on the network connection and select **Properties**.
   - Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
   - Enter the IP address, subnet mask, gateway, and DNS servers.

2. **Dynamic IP Configuration:**
   - Ensure **Obtain an IP address automatically** is selected.
   - The VM will receive an IP address from a DHCP server on the network.

![Network Settings](https://www.xsofthost.com/help/wp-content/uploads/2018/04/hyper-v-network-bridging-windows-VM-Network-adapter.jpg.webp)

## 3.6 Advanced Networking Features

### VLAN Configuration:
- Virtual LANs (VLANs) allow you to segment network traffic.
- Configure VLAN settings in the **Network Adapter** settings of the VM.

### Network Isolation:
- Use Private and Internal switches to isolate VMs from the external network.
- Useful for testing and development environments.

### Bandwidth Management:
- Limit network bandwidth for each VM in the **Network Adapter** settings.
- Ensures fair distribution of network resources.

![Advanced Network Settings](https://askme4tech.com/sites/default/files/styles/resize/public/advance-features.png?itok=uVOD8dPo)

## 3.7 Troubleshooting Network Issues

### Common Problems:
- **No Network Connectivity:** Check virtual switch configuration and VM network adapter settings.
- **IP Address Conflict:** Ensure unique IP addresses for each VM.
- **Slow Network Performance:** Check for bandwidth limits and network adapter configuration.

### Diagnostic Tools:
- **Ping:** Test connectivity between VMs and external devices.
- **ipconfig:** Check IP address configuration inside the VM.
- **Hyper-V Manager Logs:** Review logs for network-related errors.
