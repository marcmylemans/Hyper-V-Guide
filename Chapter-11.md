# Chapter 11: Integration with Microsoft Services

Hyper-V doesn't live in isolation. Microsoft has built a broad ecosystem of management, backup, monitoring, and cloud services that integrate with Hyper-V -- some free, some licensed, some cloud-based. This chapter covers the most relevant options, starting with the free tools most environments should be using, then moving into enterprise and cloud options.

---

*An IT team managing 30 Hyper-V hosts across three sites was doing all patching manually -- RDP into each host, run Windows Update, wait, reboot, move to the next one. Patching all 30 hosts manually took two full days. After deploying Windows Admin Center with Cluster-Aware Updating and a WSUS server, the same patch cycle ran overnight and completed in four hours with no manual intervention.*

---

## 11.1 Windows Admin Center (Free)

Windows Admin Center (WAC) is the modern, browser-based management tool for Windows Server and Hyper-V. It replaces the need to use multiple disconnected tools (Hyper-V Manager, Server Manager, Remote Desktop, Device Manager) by providing a unified interface for managing servers, clusters, and Hyper-V hosts.

**What WAC does:**

- Manage VMs: create, start, stop, configure, connect to consoles
- Monitor performance: real-time CPU, memory, disk, and network charts
- Manage storage: virtual disks, volumes, and Storage Spaces
- Network configuration: IP addresses, DNS, firewall rules
- Event log viewer
- PowerShell console in the browser
- Remote Desktop integration
- Failover cluster management
- Azure integration (Arc, backup, Site Recovery)

**Installing WAC:**

WAC installs on either a dedicated management server (recommended for production) or directly on a Hyper-V host.

1. Download WAC from [microsoft.com/en-us/windows-server/windows-admin-center](https://www.microsoft.com/en-us/windows-server/windows-admin-center).
2. Run the installer on your management server or Hyper-V host.
3. Choose a port (default 443) and a certificate (self-signed is fine for lab use; use a CA-signed cert for production).
4. After installation, open a browser and navigate to `https://hostname:443`.

```powershell
# Silent installation (for scripted deployments)
msiexec /i WindowsAdminCenter.msi /qn /L*v log.txt SME_PORT=443 SSL_CERTIFICATE_OPTION=generate
```

**Adding a Hyper-V host to WAC:**

1. In the WAC home screen, click **Add**.
2. Select **Servers**.
3. Enter the hostname or IP of your Hyper-V host.
4. Click **Add**. WAC connects using your current credentials (or you can specify different credentials).

> **Start with WAC.** For most Hyper-V environments, WAC provides everything you need for day-to-day management without the complexity of System Center or the cost of enterprise tooling. Use it as your primary management interface.

## 11.2 Active Directory Integration

Hyper-V works best as part of an Active Directory domain. Benefits:

- **Single Sign-On:** Administrators authenticate once to AD and gain access to all Hyper-V hosts without separate logins.
- **Group Policy:** Apply configuration policies (firewall rules, audit settings, security baselines) to Hyper-V hosts via Group Policy Objects (GPOs).
- **Kerberos authentication:** Required for Live Migration and Hyper-V Replica. AD provides the authentication infrastructure these features rely on.
- **Centralized credential management:** Password changes propagate automatically; no per-host local accounts to maintain.

**Joining a Hyper-V host to a domain:**

```powershell
# Join to the domain (requires domain admin credentials)
Add-Computer -DomainName "contoso.local" -Credential (Get-Credential) -Restart
```

**Useful GPOs for Hyper-V hosts:**
- Microsoft Security Baseline for Windows Server (download from Microsoft Security Compliance Toolkit)
- Audit policy settings (logon, object access, privilege use)
- Windows Firewall with Advanced Security rules
- Windows Update configuration

## 11.3 System Center Virtual Machine Manager (SCVMM)

> **Licensed product -- significant cost.** SCVMM requires a System Center license (Standard or Datacenter), which is priced per-core and typically costs thousands of dollars per server. Evaluate whether the features justify the cost before deploying.

SCVMM is Microsoft's enterprise VM management platform. It provides a single management console for Hyper-V hosts, clusters, and VMs across an entire datacenter.

**What SCVMM adds beyond WAC:**

- **Multi-host management at scale:** Manage hundreds of Hyper-V hosts from one console with delegation and role-based access across the whole estate.
- **VM templates and service templates:** Standardise VM configurations and deploy them in bulk.
- **Library server:** Central repository for ISOs, scripts, and VM templates.
- **Network virtualization:** Software-defined networking with NVGRE.
- **Self-service portal:** Let business units request and manage their own VMs within IT-defined boundaries.
- **Integration with Azure Arc:** Extend management to Azure-registered VMs.

**When SCVMM makes sense:** You're managing 50+ Hyper-V hosts, need self-service VM provisioning for multiple departments, or are already invested in the System Center stack (Configuration Manager, Operations Manager, etc.).

**When WAC is sufficient:** You're managing fewer than 20 hosts, don't need the self-service portal, and don't have an existing System Center investment.

## 11.4 Azure Arc for On-Premises Hyper-V

Azure Arc extends Azure management capabilities to on-premises VMs and servers -- without moving them to the cloud. With Arc:

- Your Hyper-V VMs appear in the Azure portal alongside Azure VMs.
- Azure Policy can govern on-premises VMs.
- Microsoft Defender for Cloud monitors on-premises VMs.
- Azure Monitor collects logs and metrics from on-premises VMs.
- Patch management via Azure Update Manager.

Azure Arc is consumption-based: some features are free (basic inventory and governance), others are paid (Defender for Cloud, extended security updates).

```powershell
# Install the Azure Arc agent on a Windows VM (run inside the VM)
# Download the agent from https://aka.ms/AzureConnectedMachineAgent
# Then register it:
azcmagent connect --subscription-id "your-sub-id" `
                  --resource-group "MyRG" `
                  --location "westeurope" `
                  --tenant-id "your-tenant-id"
```

## 11.5 Azure Site Recovery

Azure Site Recovery (ASR) replicates on-premises Hyper-V VMs to Azure. If your on-premises site fails (power outage, fire, hardware disaster), you can fail over to Azure and run your VMs there until the on-premises environment is restored.

**How ASR works with Hyper-V:**

1. Install the **Azure Site Recovery Provider** on your Hyper-V hosts.
2. Register the hosts with a Recovery Services vault in Azure.
3. Configure a replication policy (RPO, frequency, recovery points to retain).
4. Azure continuously receives replicated data from your VMs.
5. In a disaster, you fail over to Azure -- Azure instantiates VMs from the replicated data.
6. After recovery, fail back to on-premises when ready.

**Setting up ASR:**

1. In the Azure portal, create a **Recovery Services vault** (search "Recovery Services vaults").
2. In the vault, go to **Site Recovery > Prepare Infrastructure**.
3. Set the source to "On-premises," protection to "Hyper-V," and configure your Hyper-V site.
4. Download and install the **ASR Provider** on each Hyper-V host.
5. Register each host with the vault using the vault registration key.
6. Configure replication for individual VMs.
7. Run a **Test Failover** to a non-production Azure VNet to verify recovery works.

> **Important:** The ASR setup steps in the Azure portal change frequently as the interface is updated. Always follow the current in-portal wizard rather than older step-by-step guides (including this one). The concepts remain the same.

## 11.6 Azure Backup for Hyper-V

Azure Backup protects Hyper-V VMs by storing backups in Azure cloud storage. It provides:

- Off-site backup without managing your own off-site infrastructure
- Long-term retention (years, not just weeks)
- Encryption in transit and at rest
- Geo-redundant storage options

**Basic integration process:**

1. Create a Recovery Services vault in Azure.
2. Download and install the **Microsoft Azure Backup Server (MABS)** or use the **Azure Backup agent** directly.
3. Register the server with the vault.
4. Configure a backup policy and select VMs to protect.

Azure Backup is consumption-based: you pay for the storage used and the number of protected instances. For small numbers of VMs, it's cost-effective. For large deployments, evaluate total cost against on-premises alternatives.

## 11.7 Microsoft Azure Stack HCI

Azure Stack HCI is Microsoft's hyper-converged infrastructure solution built on the same Hyper-V and Failover Clustering foundation covered in this guide. It's worth understanding even if you're using traditional Windows Server Hyper-V:

- **Subscription-based:** Licensed per physical core, billed via Azure.
- **Azure-managed lifecycle:** Microsoft validates and certifies hardware/software combinations for Stack HCI.
- **Integrated with Azure services:** Arc, Monitor, Backup, and Site Recovery are all pre-integrated.
- **Continuously updated:** Stack HCI uses a rolling update model rather than major version releases.

For new HA cluster deployments, Azure Stack HCI is Microsoft's recommended platform going forward. Traditional Windows Server Failover Clustering remains fully supported but represents the older model.

---

> **Key Takeaways**
> - Start with Windows Admin Center -- it's free, powerful, and covers most day-to-day Hyper-V management
> - Active Directory integration enables SSO, Group Policy, and Kerberos-based features (Live Migration, Replica)
> - SCVMM is a significant cost and complexity investment -- only appropriate at enterprise scale
> - Azure Arc connects on-premises VMs to Azure management without migrating them
> - Azure Site Recovery provides cloud-based DR -- test failovers regularly
> - Azure Stack HCI is the direction Microsoft is going for new HA deployments

---

Integration with enterprise tools adds visibility and control, but something will still eventually break -- usually at the worst possible time. Chapter 12 covers troubleshooting: a systematic approach to diagnosing the most common Hyper-V issues before they turn into outages.

---

## Exercise 11: Install and Configure Windows Admin Center

> **Estimated time: 20--30 minutes**

1. Download Windows Admin Center from Microsoft's website.
2. Install it on your Hyper-V host (or a management PC on the same network).
3. Open WAC in your browser and add your Hyper-V host as a managed server.
4. Navigate to the host's Virtual Machines section. Verify you can see your VMs and their status.
5. Start and stop a VM from WAC.
6. View real-time CPU, memory, and disk charts in WAC for your Hyper-V host.
7. Compare the information available in WAC vs Hyper-V Manager. Note what WAC offers that Hyper-V Manager doesn't.
