# Chapter 9: Security Best Practices in Hyper-V

Security in a virtualised environment has two layers: securing the Hyper-V host (a compromise of the host compromises every VM on it) and securing the VMs themselves. This chapter covers both, along with Hyper-V-specific features like Secure Boot, Shielded VMs, and network isolation.

---

*A company ran eleven VMs on a single Hyper-V host. A junior administrator, troubleshooting a connectivity issue, added their personal laptop to the Hyper-V Administrators group to "just quickly test something." Six months later, a security audit discovered the account had never been removed. The laptop had since been stolen. The right fix would have taken thirty seconds. The audit took three days.*

---

## 9.1 Why Host Security Is Critical

In a physical server environment, attacking one server affects one server. In a Hyper-V environment, the host machine is the backbone of every VM running on it. If an attacker gains control of the host operating system or the hypervisor itself, they potentially have access to the memory, disk, and network traffic of every VM on that host.

This means the Hyper-V host requires a higher level of security attention than a typical server. Apply patches faster, restrict access more aggressively, and monitor more carefully.

## 9.2 Host Hardening

### Keep the Host Updated

Run Windows Updates on the host promptly, especially security updates. Hyper-V patches frequently address vulnerabilities in the hypervisor itself or in how VMs interact with the host.

```powershell
# Check for and install all available updates
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate -Install -AcceptAll -AutoReboot
```

### Minimize What Runs on the Host

The Hyper-V host should be a dedicated Hyper-V host -- not also a file server, a domain controller, or a web server. Every additional role adds attack surface. Additional services running on the host can also consume resources that should be reserved for VMs.

**Consider Server Core:** Windows Server Core (no GUI) has a dramatically smaller attack surface than the full Desktop Experience installation. Fewer services, fewer ports open, less code running. Manage it via PowerShell or Windows Admin Center remotely.

### Restrict Administrative Access

Who can manage your Hyper-V host? Keep that list short and make it deliberate.

Windows provides a built-in **Hyper-V Administrators** local group specifically for this purpose. Members of this group can create, start, stop, and manage VMs without holding full local administrator rights on the host. This is an important distinction: a VM operator doesn't need the ability to install software, change firewall rules, or access host-level files. Add users to Hyper-V Administrators rather than to the Administrators group unless they genuinely need host-level control.

```powershell
# Add a user to the Hyper-V Administrators group
Add-LocalGroupMember -Group "Hyper-V Administrators" -Member "DOMAIN\HVAdmins"
```

In larger environments, layer Active Directory security groups on top of this. Create distinct groups for VM operators (can start and stop VMs), VM administrators (can also create and configure VMs), and host administrators (full host access). Assign permissions to groups, not individuals -- this ensures access is auditable, revocable, and survives personnel changes without anyone manually tracking who has what.

### Firewall Configuration

On the host, only the ports necessary for Hyper-V management and VM operation should be open. Default ports to be aware of:

| Port | Protocol | Purpose |
|---|---|---|
| 2179 | TCP | VMConnect (Hyper-V Manager remote console connections) |
| 443 | TCP | Hyper-V Replica (HTTPS) / Windows Admin Center |
| 80 | TCP | Hyper-V Replica (HTTP, unencrypted -- prefer HTTPS) |
| 445 | TCP | SMB (for SMB-based storage) |
| 5985/5986 | TCP | PowerShell Remoting (HTTP/HTTPS) |

Close everything else. Use Windows Firewall with Advanced Security to create explicit rules.

### Credential Guard

Windows Defender Credential Guard uses Hyper-V-based virtualisation to isolate and protect credential hashes from the LSASS process. Even if malware runs on the host, it cannot extract stored credentials from memory. This is one of the most impactful security features in Windows and is directly enabled by Hyper-V.

```powershell
# Enable Credential Guard via Group Policy or registry
# Via registry (requires reboot):
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\LSA" `
                 -Name "LsaCfgFlags" -Value 1 -PropertyType DWORD -Force
```

Credential Guard is enabled by default on domain-joined Windows 11 Enterprise devices. On Windows Server, it requires explicit configuration.

## 9.3 Secure Boot for VMs

Secure Boot prevents unauthorized code from running during the VM's boot process. It verifies that the bootloader is signed by a trusted certificate before allowing it to execute.

- Available only on **Generation 2** VMs.
- Enabled by default when you create a Generation 2 VM.
- For Linux VMs, you may need to change the Secure Boot template from "Microsoft Windows" to "Microsoft UEFI Certificate Authority" in the VM's Firmware settings.

```powershell
# Check Secure Boot status on a VM
Get-VMFirmware -VMName "MyVM" | Select-Object SecureBoot

# Enable Secure Boot (for Windows VMs)
Set-VMFirmware -VMName "MyVM" -EnableSecureBoot On -SecureBootTemplate "MicrosoftWindows"

# Enable Secure Boot (for Linux VMs)
Set-VMFirmware -VMName "MyVM" -EnableSecureBoot On -SecureBootTemplate "MicrosoftUEFICertificateAuthority"
```

## 9.4 Network Security

### Virtual Switch Isolation

Private and Internal virtual switches (Chapter 3) are isolation mechanisms. VMs on a Private switch cannot communicate with the host or the physical network -- this is a security feature as much as a network design tool. Use Private switches for VMs that have no business reaching the internet or the production network.

### Port ACLs

You can apply access control rules to individual VM network adapters at the virtual switch level, controlling what traffic the VM can send or receive:

```powershell
# Block all inbound traffic from a specific IP to a VM
Add-VMNetworkAdapterAcl -VMName "WebServer" `
    -Direction Inbound `
    -Action Deny `
    -RemoteIPAddress "203.0.113.0/24"  # Block an entire subnet

# Allow only HTTP and HTTPS inbound, deny everything else
Add-VMNetworkAdapterAcl -VMName "WebServer" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 80

Add-VMNetworkAdapterAcl -VMName "WebServer" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 443

# View all ACLs on a VM's adapter
Get-VMNetworkAdapterAcl -VMName "WebServer"
```

### VLAN Segmentation

Use VLANs (Chapter 3) to separate different categories of VMs at the network level: production VMs in VLAN 10, management VMs in VLAN 20, DMZ VMs in VLAN 30. Combine this with a firewall between VLANs to control inter-VLAN traffic.

### Network Encryption

For traffic between VMs on the same host or between hosts on the same network, consider using **IPsec** to encrypt sensitive communication. Windows Server includes built-in IPsec support configurable via Windows Firewall with Advanced Security.

For Hyper-V Replica traffic, use the HTTPS/certificate-based authentication option (not Kerberos/HTTP) to encrypt replication traffic in transit.

## 9.5 Storage Security

### BitLocker for VHDX Files

Encrypting the host's drives with BitLocker means that even if someone physically removes a drive and tries to read the VHDX files from another machine, they can't access the data. Enable BitLocker on all volumes storing VM data.

```powershell
# Encrypt a drive (requires TPM or startup key)
Enable-BitLocker -MountPoint "D:" -EncryptionMethod XtsAes256 -TpmProtector
```

### NTFS Permissions on VHDX Files

The directory containing VHDX files should have restricted NTFS permissions. Only the Hyper-V Virtual Machine Management Service account (`NT VIRTUAL MACHINE\Virtual Machines`) and administrators should have access. Remove "Everyone" and any broad permission groups.

### Shielded VMs for Multi-Tenant or High-Security Environments

See Chapter 5, section 5.5 for the full Shielded VM setup. In brief: Shielded VMs encrypt the entire VM with BitLocker and tie decryption to HGS attestation, meaning the VM can only run on approved hosts. A Hyper-V administrator on an unapproved host cannot decrypt or access the VM's contents even if they have the VHDX file.

## 9.6 VM Security (Inside the Guest)

The Hyper-V host's security doesn't automatically protect what's running inside VMs. Each VM is its own system that needs its own security posture:

- **Keep guest OS patched:** VMs fall behind on updates surprisingly fast, especially if they're powered off for periods. Enable Windows Update or an equivalent patch management tool inside each VM.
- **Antivirus/EDR inside VMs:** Run endpoint protection inside each VM, not just on the host. Hyper-V integration does not provide antivirus scanning for VM contents.
- **Strong, unique passwords:** Every service account and administrator account inside VMs should have unique, complex passwords. A common mistake is reusing the same local administrator password across all VMs.
- **Disable unnecessary services:** Just like the host, VMs should only run what they need.

### Remote Management Security

When managing VMs remotely, use secure protocols:

- **PowerShell Remoting over HTTPS:** `Enter-PSSession -ComputerName "Server01" -UseSSL`
- **RDP with Network Level Authentication (NLA):** NLA authenticates the user before a full RDP session is established, reducing the attack surface against the RDP service itself.
- **Windows Admin Center over HTTPS:** WAC uses HTTPS with a certificate. In production, use a proper CA-signed certificate, not the self-signed one generated at installation.

## 9.7 Auditing and Monitoring Security Events

### Enable Windows Auditing

Configure audit policies inside VMs and on the host to log security-relevant events:

```powershell
# Enable auditing via auditpol (run inside each VM or via GPO)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Account Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Object Access" /success:enable /failure:enable
```

### Monitor Security Logs

Check Windows Security Event logs regularly:

- **Event ID 4624:** Successful logon
- **Event ID 4625:** Failed logon (repeated failures indicate brute-force attempts)
- **Event ID 4648:** Logon with explicit credentials (may indicate lateral movement)
- **Event ID 4776:** NTLM authentication attempt

```powershell
# Find failed logon attempts in the last 24 hours
Get-WinEvent -LogName Security -MaxEvents 1000 |
    Where-Object {$_.Id -eq 4625 -and $_.TimeCreated -gt (Get-Date).AddDays(-1)} |
    Select-Object TimeCreated, Message
```

## 9.8 Incident Response Planning

Security incidents happen. Having a plan before one occurs makes the difference between a recoverable situation and a catastrophic one.

**Define your plan before you need it:**

1. **Detection:** What triggers an incident declaration? (Unusual process, unexpected outbound connections, failed access to the hypervisor, ransomware indicators.)
2. **Isolation:** Which VMs/networks get isolated immediately? How do you isolate a VM without losing evidence? (Use the Hyper-V "Save" function rather than power-off to preserve memory state for forensic analysis.)
3. **Eradication:** Where are your clean backups? How long does a restore take? Have you tested this?
4. **Communication:** Who needs to be notified? Do you have contact info stored somewhere that isn't on the compromised systems?
5. **Post-incident review:** What failed? What will you change?

**Preserving evidence from a compromised VM:**

Instead of shutting down a compromised VM (which destroys its memory state), save it:
```powershell
# Preserve VM memory state for forensic analysis
Save-VM -Name "CompromisedVM"
# Then copy the saved state files and VHDX to a forensic location before restoring from backup
```

---

> **Key Takeaways**
> - The Hyper-V host is the most critical security target -- a compromised host means compromised VMs
> - Use Server Core for lower attack surface; restrict host access to the minimum necessary users
> - Credential Guard uses Hyper-V to protect credentials -- enable it on all domain-joined systems
> - Secure Boot prevents unauthorized boot code; enabled by default on Gen 2 VMs
> - Private virtual switches provide network isolation at the hypervisor level
> - BitLocker on host drives protects VHDX files from physical theft
> - Test your incident response plan -- the worst time to figure it out is during an incident

---

Security policies are only as good as the people applying them consistently -- and consistent application at scale requires automation. Chapter 10 covers PowerShell scripting for Hyper-V: from simple one-liners that save minutes to scripts that manage entire VM fleets without touching Hyper-V Manager.

---

## Exercise 9: Security Audit Your Hyper-V Host

> **Estimated time: 30--45 minutes**

1. Run `Get-LocalGroupMember -Group "Administrators"` on your host. Who has admin access? Is everyone there who needs to be? Is anyone there who shouldn't be?
2. Run `Get-LocalGroupMember -Group "Hyper-V Administrators"`. Is this group being used, or are VM managers in the full Administrators group?
3. Check Windows Firewall: `Get-NetFirewallRule -Enabled True | Where-Object {$_.Direction -eq "Inbound"} | Select-Object DisplayName, LocalPort`. Review the open inbound ports. Are any unexpected?
4. Check Secure Boot status on your VMs: `Get-VMFirmware -VMName * | Select-Object VMName, SecureBoot`.
5. Create a new local user on your host with minimal permissions. Add them to the Hyper-V Administrators group. Log in as that user and verify they can manage VMs but cannot install software or change system settings.
