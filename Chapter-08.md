# Chapter 8: High Availability and Failover Clustering

> **Enterprise environments.** This chapter assumes you have at least two physical servers running Windows Server, shared storage accessible to all servers, and both servers joined to an Active Directory domain. Home lab readers: you can read this chapter for understanding, but you'll need two physical machines and enterprise storage to implement it. Skip to Chapter 9 if you're working with a single host.

High availability (HA) in Hyper-V means that if a physical server fails, its virtual machines automatically restart on another server in the cluster -- typically within a minute or two -- without any manual intervention. This is achieved through **Failover Clustering**: a Windows Server feature that groups multiple hosts into a cooperative cluster where each host monitors the others.

At 3:17 AM, Node01 stops responding to heartbeats. The cluster detects the silence, confirms Node02 can still reach the witness, and reaches quorum without Node01. Within 90 seconds, every VM that was running on Node01 is restarted on Node02. The on-call engineer's phone never rings. By the time the morning shift arrives and checks Failover Cluster Manager, they see Node01 marked as down and a note in the event log: failover completed, all VMs healthy. That's what high availability actually delivers.

## 8.1 How Failover Clustering Works

A Failover Cluster is a group of physical servers (called **nodes**) that work together. Each node runs the Hyper-V role and can host VMs. All nodes have access to the same shared storage, which is where the VM files live.

The cluster continuously monitors the health of all nodes using a **heartbeat** network (a dedicated network between nodes used only for cluster communication). If a node stops responding to heartbeats, the cluster detects the failure and initiates failover: the VMs that were running on the failed node are restarted on the surviving nodes.

The cluster also needs to agree on what a "majority" is, to avoid **split-brain** -- the dangerous scenario where two halves of a cluster both think the other half has failed and both try to take control simultaneously. This is managed by **quorum**.

### Quorum

Quorum is the mechanism that prevents split-brain. The cluster has a vote count, and a majority of votes must agree that the cluster is healthy for it to operate. If a node fails and the cluster loses its majority, it stops operating rather than risk split-brain.

**Quorum modes:**

| Mode | How It Works | Best For |
|---|---|---|
| Node Majority | Each node votes; majority required | Clusters with an odd number of nodes |
| Node and Disk Majority | Nodes + a small "disk witness" on shared storage vote | Clusters with an even number of nodes |
| Node and File Share Majority | Nodes + a file share on another server vote | Clusters where the disk witness isn't available or desired |
| Cloud Witness | Nodes + an Azure Blob Storage object vote | Modern deployments; requires internet connectivity |

For a two-node cluster (the most common small deployment), you need a tiebreaker: either a Disk Witness or a File Share Witness. Without it, either node failing takes the cluster below quorum and the surviving node shuts down.

> **Getting quorum wrong is a common cause of cluster outages.** Configure it carefully based on your node count.

This is why HA clusters are standard in enterprise environments that can't afford downtime: not because failures are common, but because failures are unpredictable. A cluster that handles a node failure automatically -- in the middle of the night, without anyone being paged -- is infrastructure that earns trust.

The decision to build a cluster is also a decision about operational philosophy. A single powerful host is simpler to manage and cheaper to buy. But when it fails, everything stops and someone has to respond immediately. A cluster costs more and requires more planning, but shifts the operational posture from "respond to failures" to "review failures in the morning."

## 8.2 Prerequisites

Before building a Failover Cluster:

- **Windows Server edition:** Standard or Datacenter on all nodes.
- **Active Directory domain:** All nodes must be domain-joined.
- **Identical or compatible hardware:** Not strictly required, but strongly recommended. Mismatched hardware can cause migration issues.
- **Dedicated networks:** At minimum, separate networks for management, cluster heartbeat, and VM traffic. Shared storage traffic should also be on a dedicated network.
- **Shared storage:** All nodes must be able to access the same storage. Options: iSCSI, Fibre Channel, or SMB 3.0 file shares. Storage Spaces Direct (covered in section 8.7) pools the local disks of the nodes instead.
- **Cluster Validation:** Run the validation tool before creating the cluster and fix all failures it reports.

## 8.3 Building a Failover Cluster: Step by Step

### Step 1: Install the Failover Clustering Feature

On each node:
```powershell
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools
```

### Step 2: Validate the Configuration

This step is mandatory. Running without a passed validation report is unsupported and risks data corruption.

```powershell
Test-Cluster -Node "Node01","Node02" -Include "Storage","Network","Inventory","System Configuration"
```

Review the HTML report that's generated. Fix all **Failures** before proceeding. **Warnings** should be investigated but may not block cluster creation.

### Step 3: Create the Cluster

```powershell
New-Cluster -Name "HVCluster01" `
            -Node "Node01","Node02" `
            -StaticAddress "192.168.1.100"  # Cluster IP address
```

### Step 4: Configure Quorum

For a two-node cluster with a file share witness:
```powershell
Set-ClusterQuorum -FileShareWitness "\\FileServer\ClusterWitness"
```

For a cloud witness (requires an Azure storage account):
```powershell
Set-ClusterQuorum -CloudWitness `
                  -AccountName "mystorageaccount" `
                  -AccessKey "your-storage-account-key"
```

### Step 5: Add Shared Storage to the Cluster

1. Open **Failover Cluster Manager** and connect to your cluster.
2. Navigate to **Storage > Disks**.
3. Click **Add Disk**. Shared disks visible to all nodes appear in the list.
4. Select them and add to the cluster.
5. Right-click the disks you want as CSVs > **Add to Cluster Shared Volumes**.

### Step 6: Make VMs Highly Available

For VMs already on the cluster's shared storage:

1. In Failover Cluster Manager, navigate to **Roles**.
2. Click **Configure Role** in the right pane.
3. Select **Virtual Machine**.
4. Select the VMs you want to make highly available.
5. Complete the wizard.

```powershell
# Add a VM to the cluster (make it highly available)
Add-ClusterVirtualMachineRole -VMName "MyVM"
```

## 8.4 Managing a Failover Cluster

### Moving VMs Between Nodes

**Live Migration** (Chapter 5) works within a cluster and is the preferred way to move running VMs for maintenance. When you need to take a node offline for updates:

1. In Failover Cluster Manager, right-click the node > **Pause > Drain Roles**.
2. This Live Migrates all VMs to other nodes before pausing the node.
3. Perform your maintenance.
4. Right-click the node > **Resume > Fail Roles Back** to return VMs to the original node.

```powershell
# Drain all roles from a node before maintenance
Suspend-ClusterNode -Name "Node01" -Drain
```

### Cluster-Aware Updating (CAU)

CAU automates Windows Update patching across cluster nodes while maintaining HA. It updates one node at a time, draining its VMs first, applying updates, restarting, then moving to the next node.

1. In Failover Cluster Manager, navigate to **Cluster-Aware Updating**.
2. Configure the update schedule and source (Windows Update, WSUS, or local).
3. Initiate an update run -- CAU handles the drain/update/restart cycle automatically.

### Monitoring Cluster Health

```powershell
# Get the status of all cluster resources
Get-ClusterResource

# Check which node each VM is currently running on
Get-ClusterGroup | Where-Object {$_.GroupType -eq "VirtualMachine"} |
    Select-Object Name, OwnerNode, State

# Get cluster events from the last 24 hours
Get-ClusterLog -Destination "C:\ClusterLog" -UseLocalTime -TimeSpan 1440
```

## 8.5 Testing Your HA Configuration

Build confidence in your cluster before it's needed in an emergency:

**Simulate a node failure:**
```powershell
# Stop the cluster service on one node (simulates a failure without physically pulling the power)
Stop-Service -Name ClusSvc -Force -ComputerName "Node01"
```
Watch VMs restart on the other node. Measure how long failover takes. Verify VMs are accessible after failover.

**Simulate a controlled failover:**
In Failover Cluster Manager, right-click a VM > **Move > Best Possible Node**. This performs a Live Migration to the best available node.

> **Test failovers regularly.** An HA system you've never tested under failure conditions is an HA system you cannot trust. Run a simulated failover quarterly.

## 8.6 Troubleshooting High Availability Issues

**Node failing to join the cluster:**
- Verify the node can reach all other nodes over all cluster networks.
- Check that the cluster service account has appropriate permissions.
- Review cluster event logs: Event Viewer > Applications and Services Logs > Microsoft > Windows > FailoverClustering.

**VMs not failing over after node failure:**
- Check that the VMs are configured as Highly Available in Failover Cluster Manager (they must appear under Roles, not just in the node's VM list).
- Verify shared storage is accessible from all nodes.
- Check cluster quorum status: `Get-ClusterQuorum`.

**Split-brain prevention -- why quorum matters:**
If half your nodes lose network connectivity to the other half, both halves could try to run the same VMs. Quorum prevents this: only the side with a majority of votes continues operating. The minority side shuts down its cluster service. Configure your quorum carefully so that the side with your most important VMs is the side that survives a split.

## 8.7 Azure Stack HCI: The Modern Alternative

> **Worth knowing for new deployments.**

Azure Stack HCI is Microsoft's current recommended platform for hyper-converged infrastructure. Under the hood, it uses the same Failover Clustering and Hyper-V technology covered in this chapter, but it adds:

- **Azure Arc integration:** Manage and monitor your on-premises cluster from the Azure portal.
- **Automated updates:** Microsoft manages the update baseline for the HCI stack.
- **Storage Spaces Direct (S2D) out of the box:** No dedicated shared storage hardware required -- the nodes pool their local disks.
- **Simplified licensing:** Subscription-based per-node rather than the complex Windows Server Datacenter + System Center licensing.

If you're evaluating a new HA Hyper-V deployment and aren't already invested in traditional Windows Server Failover Clustering, Azure Stack HCI is worth evaluating. It's designed for exactly this use case.

## 8.8 Storage Spaces Direct (S2D)

> **Requires: Windows Server Datacenter edition + Failover Cluster**

Storage Spaces Direct pools the local disks of multiple Hyper-V hosts into a single, redundant storage pool that all nodes can access -- no dedicated SAN or NAS required. This is the hyper-converged infrastructure (HCI) model.

```powershell
# Enable Storage Spaces Direct on an existing cluster
Enable-ClusterStorageSpacesDirect

# Create a storage pool and volumes via Windows Admin Center (recommended)
# or via PowerShell:
New-Volume -StoragePoolFriendlyName "S2D*" `
           -FriendlyName "VMStorage" `
           -FileSystem CSVFS_ReFS `
           -Size 2TB
```

S2D uses cache tiers (fast NVMe or SSD for caching) and capacity tiers (larger HDDs or SSDs for bulk storage) to deliver SAN-class performance from commodity hardware.

---

> **Key Takeaways**
> - Failover Clustering keeps VMs running through hardware failures by automatically restarting them on surviving nodes
> - Quorum prevents split-brain -- configure it carefully, especially for two-node clusters
> - Always run Cluster Validation and fix all failures before creating a cluster
> - Cluster-Aware Updating patches nodes without downtime by draining VMs first
> - Test failovers regularly -- quarterly at minimum
> - Azure Stack HCI is the modern evolution of Windows Server Failover Clustering -- worth evaluating for new deployments

---

A cluster keeps your VMs running if hardware fails -- but a misconfigured host or a compromised account can take down even a well-clustered environment. Chapter 9 covers the security practices that protect your hosts and VMs from attack and unauthorized access: the controls that matter, what to lock down first, and how to verify your posture.

---

## Exercise 8: Cluster Simulation (Two Physical Hosts or Nested VMs)

> **Estimated time: 90--120 minutes** -- This is the most complex exercise in the guide. Allocate at least two hours and don't start it twenty minutes before a meeting.

If you have two physical hosts (or set up two nested Hyper-V VMs as in Exercise 5):

1. Install Failover Clustering on both: `Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools`.
2. Run `Test-Cluster -Node "Host1","Host2"` and review the report.
3. Create a cluster: `New-Cluster -Name "LabCluster" -Node "Host1","Host2" -StaticAddress "10.0.0.50"`.
4. Configure a File Share Witness using a network share on a third machine (or your management PC).
5. Create a VM on shared storage accessible to both nodes.
6. Make the VM highly available: `Add-ClusterVirtualMachineRole -VMName "TestVM"`.
7. While the VM is running, stop the cluster service on the node that owns it: `Stop-Service ClusSvc -Force`.
8. Observe the VM failing over to the other node. Time how long it takes.
