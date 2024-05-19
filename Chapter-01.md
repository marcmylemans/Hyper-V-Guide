# Chapter 1: Introduction to Hyper-V

## 1.1 What is Hyper-V?

Hyper-V is a virtualization technology developed by Microsoft. It allows you to create and manage virtual machines (VMs) on a single physical machine, known as the host. Each VM operates like a separate computer with its own operating system and applications. Hyper-V is available on Windows Server and some versions of Windows 10 and 11.

## 1.2 Why Use Hyper-V?

### Benefits of Virtualization:
1. **Resource Optimization:** Utilize hardware resources more efficiently.
2. **Isolation:** Each VM is isolated, preventing issues in one VM from affecting others.
3. **Testing and Development:** Easily create test environments without additional hardware.
4. **Disaster Recovery:** Simplified backup and recovery processes.

### Use Cases:
- **Development and Testing:** Create isolated environments to test software.
- **Server Consolidation:** Run multiple server roles on a single physical machine.
- **Business Continuity:** Maintain critical services in case of hardware failure.

## 1.3 Hyper-V Editions

### Windows Server Hyper-V:
- **Enterprise-level virtualization** with advanced features.
- Available in **Windows Server** editions like Standard and Datacenter.

### Windows 10/11 Hyper-V:
- Included in **Pro, Enterprise, and Education** editions.
- Suitable for **development and testing** environments.

## 1.4 System Requirements

### Hardware Requirements:
- **64-bit processor** with Second Level Address Translation (SLAT).
- **4 GB or more RAM.**
- **Virtualization enabled** in BIOS/UEFI.

### Software Requirements:
- **Windows Server 2016/2019/2022** or **Windows 10/11 Pro, Enterprise, or Education** editions.

## 1.5 How to Enable Hyper-V

### On Windows 10/11:
1. Open **Control Panel**.
2. Navigate to **Programs > Turn Windows features on or off**.
3. Check the box for **Hyper-V** and click **OK**.
4. Restart your computer.

![Enable Hyper-V](https://mylemans.online/assets/img/Hyper-V-Guide/Chapter-01/enable_role_upd.png)


### On Windows Server:
1. Open **Server Manager**.
2. Click **Add roles and features**.
3. Follow the wizard to install **Hyper-V**.
4. Restart your server.

![Enable Hyper-V on Server](https://www.sqlskills.com/blogs/tim/wp-content/uploads/2016/02/AddRoles1.png)

## 1.6 Hyper-V Manager

Hyper-V Manager is the main tool to manage Hyper-V. It allows you to:
- Create and manage VMs.
- Configure VM settings.
- Monitor VM performance.

### How to Open Hyper-V Manager:
- **Windows 10/11:** Type **Hyper-V Manager** in the Start menu search and open it.
- **Windows Server:** Open **Server Manager**, go to **Tools**, and select **Hyper-V Manager**.

![Hyper-V Manager Interface](https://docs.oracle.com/en/database/oracle/key-vault/21.6/okvig/img/hyper-v-manager.png)
