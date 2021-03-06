---
title: SQL Server FCI - Azure Virtual Machines | Microsoft Docs
description: "This article explains how to create SQL Server Failover Cluster Instance on Azure Virtual Machines."
services: virtual-machines
documentationCenter: na
authors: MikeRayMSFT
manager: jhubbard
editor: monicar
tags: azure-service-management

ms.assetid: 9fc761b1-21ad-4d79-bebc-a2f094ec214d
ms.service: virtual-machines-sql
ms.devlang: na
ms.custom: na
ms.topic: article
ms.tgt_pltfrm: vm-windows-sql-server
ms.workload: iaas-sql-server
ms.date: 03/17/2017
ms.author: mikeray

---

# Configure SQL Server Failover Cluster Instance on Azure Virtual Machines

This article explains how to create a SQL Server Failover Cluster Instance (FCI) on Azure virtual machines in Resource Manager model. This solution uses [Windows Server 2016 Datacenter edition Storage Spaces Direct \(S2D\)](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/storage-spaces-direct-overview) as a software-based virtual SAN that synchronizes the storage (data disks) between the nodes (Azure VMs) in a Windows Cluster. S2D is new in Windows Server 2016.

The following diagram shows the complete solution on Azure virtual machines:

![Availability Group](./media/virtual-machines-windows-portal-sql-create-failover-cluster/00-sql-fci-s2d-complete-solution.png)

The preceding diagram shows:

- Two Azure virtual machines in a Windows Failover Cluster. When a virtual machine is in a failover cluster it is also called a *cluster node*, or *nodes*.
- Each virtual machine has two or more data disks.
- S2D synchronizes the data on the data disk and presents the synchronized storage as a storage pool.
- The storage pool presents a cluster shared volume (CSV) to the failover cluster.
- The SQL Server FCI cluster role uses the CSV for the data drives.
- An Azure load balancer to hold the IP address for the SQL Server FCI.
- An Azure availability set holds all the resources.

   >[!NOTE]
   >All Azure resources are in the diagram are in the same resource group.

For details about S2D, see [Windows Server 2016 Datacenter edition Storage Spaces Direct \(S2D\)](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/storage-spaces-direct-overview).

S2D supports two types of architectures - converged and hyper-converged. The architecture in this document is hyper-converged. A hyper-converged infrastructure places the storage on the same servers that host the clustered application. In this architecture, the storage is on each SQL Server FCI node.

### Example Azure template

You can create the entire solution in Azure from a template. An example of a template is available in the GitHub [Azure Quickstart Templates](https://github.com/MSBrett/azure-quickstart-templates/tree/master/sql-server-2016-fci-existing-vnet-and-ad). This example is not designed or tested for any specific workload. You can run the template to create a SQL Server FCI with S2D storage connected to your domain. You can evaluate the template, and modify it for your purposes.

## Before you begin

There are a few things you need to know and a couple of things that you need in place before you proceed.

### What to know
You should have an operational understanding of the following technologies:

- [Windows cluster technologies](http://technet.microsoft.com/library/hh831579.aspx)
-  [SQL Server Failover Cluster Instances](http://msdn.microsoft.com/library/ms189134.aspx).

Also, you should have a general understanding of the following technologies:

- [Hyper-converged solution using Storage Spaces Direct in Windows Server 2016](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct)
- [Azure resource groups](../../../azure-resource-manager/resource-group-portal.md)

### What to have

Before following the instructions in this article, you should already have:

- A Microsoft Azure subscription.
- A Windows domain on Azure virtual machines.
- An account with permission to create objects in the Azure virtual machine.
- An Azure virtual network and subnet with sufficient IP address space for the following components:
   - Both virtual machines.
   - The failover cluster IP address.
   - An IP address for each FCI.
- DNS configured on the Azure Network, pointing to the domain controllers.

With these prerequisites in place, you can proceed with building your failover cluster. The first step is to create the virtual machines.

## Step 1: Create virtual machines

1. Log in to the [Azure portal](http://portal.azure.com) with your subscription.

1. [Create an Azure availability set](../tutorial-availability-sets.md).

   The availability set groups virtual machines across fault domains and update domains. The availability set makes sure that your application is not affected by single points of failure, like the network switch or the power unit of a rack of servers.

   If you have not created the resource group for your virtual machines, do it when you create an Azure availability set. If you're using the Azure portal to create the availability set, do the following steps:

   - In the Azure portal, click **+** to open the Azure Marketplace. Search for **Availability set**.
   - Click **Availability set**.
   - Click **Create**.
   - On the **Create availability set** blade, set the following values:
      - **Name**: A name for the availability set.
      - **Subscription**: Your Azure subscription.
      - **Resource group**: If you want to use an existing group, click **Use existing** and select the group from the drop-down list. Otherwise choose **Create New** and type a name for the group.
      - **Location**: Set the location where you plan to create your virtual machines.
      - **Fault domains**: Use the default (3).
      - **Update domains**: Use the default (5).
   - Click **Create** to create the availability set.

1. Create the virtual machines in the availability set.

   Provision two SQL Server virtual machines in the Azure availability set. For instructions, see [Provision a SQL Server virtual machine in the Azure portal](virtual-machines-windows-portal-sql-server-provision.md).

   Place both virtual machines:

   - In the same Azure resource group that your availability set is in.
   - On the same network as your domain controller.
   - On a subnet with sufficient IP address space for both virtual machines, and all FCIs that you may eventually use on this cluster.
   - In the Azure availability set.   

      >[!IMPORTANT]
      >You cannot set or change availability set after a virtual machine has been created.

   Choose an image from the Azure Marketplace. You can use a Marketplace image with that includes Windows Server and SQL Server, or just the Windows Server. For details, see [Overview of SQL Server on Azure Virtual Machines](../../virtual-machines-windows-sql-server-iaas-overview.md)

   The official SQL Server images in the Azure Gallery include an installed SQL Server instance, plus the SQL Server installation software, and the required key.

   Choose the right image according to how you want to pay for the SQL Server license:

   - **Pay per usage licensing**: The per-minute cost of these images includes the SQL Server licensing:
      - **SQL Server 2016 Enterprise on Windows Server Datacenter 2016**
      - **SQL Server 2016 Standard on Windows Server Datacenter 2016**
      - **SQL Server 2016 Developer on Windows Server Datacenter 2016**

   - **Bring-your-own-license (BYOL)**

      - **{BYOL} SQL Server 2016 Enterprise on Windows Server Datacenter 2016**
      - **{BYOL} SQL Server 2016 Standard on Windows Server Datacenter 2016**

   >[!IMPORTANT]
   >After you create the virtual machine, remove the pre-installed standalone SQL Server instance. You will use the pre-installed SQL Server media to create the SQL Server FCI after you configure the failover cluster and S2D.

   Alternatively, you can use Azure Marketplace images with just the operating system. Choose a **Windows Server 2016 Datacenter** image and install the SQL Server FCI after you configure the failover cluster and S2D. This image does not contain SQL Server installation media. Place the installation media in a location where you can run the SQL Server installation for each server.

1. After Azure creates your virtual machines, connect to each virtual machine with RDP.

   When you first connect to a virtual machine with RDP, the computer asks if you want to allow this PC to be discoverable on the network. Click **Yes**.

1. If you are using one of the SQL Server-based virtual machine images, remove the SQL Server instance.

   - In **Programs and Features**, right-click **Microsoft SQL Server 2016 (64-bit)** and click **Uninstall/Change**.
   - Click **Remove**.
   - Select the default instance.
   - Remove all features under **Database Engine Services**. Do not remove **Shared Features**. See the following picture:

      ![Remove Features](./media/virtual-machines-windows-portal-sql-create-failover-cluster/03-remove-features.png)

   - Click **Next**, and then click **Remove**.

1. <a name="ports"></a>Open the firewall ports.

   On each virtual machine, open the following ports on the Windows Firewall.

   | Purpose | TCP Port | Notes
   | ------ | ------ | ------
   | SQL Server | 1433 | Normal port for default instances of SQL Server. If you used an image from the gallery, this port is automatically opened.
   | Health probe | 59999 | Any open TCP port. In a later step, configure the load balancer [health probe](#probe) and the cluster to use this port.  

1. Add storage to the virtual machine. For detailed information, see [add storage](../../../storage/storage-premium-storage.md).

   Both virtual machines need at least two data disks.

   Attach raw disks - not NTFS formatted disks.
      >[!NOTE]
      >If you attach NTFS-formatted disks, you can only enable S2D with no disk eligibility check.  

   Attach a minimum of two Premium Storage (SSD disks) to each VM. We recommend at least P30 (1 TB) disks.

   Set host caching to **Read-only**.

   The storage capacity you use in production environments depends on your workload. The values described in this article are for demonstration and testing.

1. [Add the virtual machines to your pre-existing domain](virtual-machines-windows-portal-sql-availability-group-prereq.md#joinDomain).

After the virtual machines are created and configured, you can configure the failover cluster.

## Step 2: Configure the Windows Failover Cluster with S2D

The next step is to configure the failover cluster with S2D. In this step, you will do the following substeps:

1. Add Windows Failover Clustering feature
1. Validate the cluster
1. Create the failover cluster
1. Create the cloud witness
1. Add storage

### Add Windows Failover Clustering feature

1. To begin, connect to the first virtual machine with RDP using a domain account that is a member of local administrators, and has permissions to create objects in Active Directory. Use this account for the rest of the configuration.

1. [Add Failover Clustering feature to each virtual machine](virtual-machines-windows-portal-sql-availability-group-prereq.md#add-failover-clustering-features-to-both-sql-server-vms).

   To install Failover Clustering feature from the UI, do the following steps on both virtual machines.
   - In **Server Manager**, click **Manage**, and then click **Add Roles and Features**.
   - In **Add Roles and Features Wizard**, click **Next** until you get to **Select Features**.
   - In **Select Features**, click **Failover Clustering**. Include all required features and the management tools. Click **Add Features**.
   - Click **Next** and then click **Finish** to install the features.

   To install the Failover Clustering feature with PowerShell, run the following script from an administrator PowerShell session on one of the virtual machines.

   ```PowerShell
   $nodes = ("<node1>","<node2>")
   Invoke-Command  $nodes {Install-WindowsFeature Failover-Clustering -IncludeAllSubFeature -IncludeManagementTools}
   ```

For reference, the next steps follow the instructions under Step 3 of [Hyper-converged solution using Storage Spaces Direct in Windows Server 2016](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-3-configure-storage-spaces-direct).

### Validate the cluster

This guide refers to instructions under [validate cluster](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-31-run-cluster-validation).

Validate the cluster in the UI or with PowerShell.

To validate the cluster with the UI, do the following steps from one of the virtual machines.

1. In **Server Manager**, click **Tools**, then click **Failover Cluster Manager**.
1. In **Failover Cluster Manager**, click **Action**, then click **Validate Configuration...**.
1. Click **Next**.
1. On **Select Servers or a Cluster**, type the name of both virtual machines.
1. On **Testing options**, choose **Run only tests I select**. Click **Next**.
1. On **Test selection**, include all tests except **Storage**. See the following picture:

   ![Validate Tests](./media/virtual-machines-windows-portal-sql-create-failover-cluster/10-validate-cluster-test.png)

1. Click **Next**.
1. On **Confirmation**, click **Next**.

The **Validate a Configuration Wizard** runs the validation tests.

To validate the cluster with PowerShell, run the following script from an administrator PowerShell session on one of the virtual machines.

   ```PowerShell
   Test-Cluster –Node ("<node1>","<node2>") –Include "Storage Spaces Direct", "Inventory", "Network", "System Configuration"
   ```

After you validate the cluster, create the failover cluster.

### Create the failover cluster

This guide refers to [Create the failover cluster](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-32-create-a-cluster).

To create the failover cluster, you need:
- The names of the virtual machines that become the cluster nodes.
- A name for the failover cluster
- An IP address for the failover cluster. You can use an IP address that is not used on the same Azure virtual network and subnet as the cluster nodes.

The following PowerShell creates a failover cluster. Update the script with the names of the nodes (the virtual machine names) and an available IP address from the Azure VNET:

```PowerShell
New-Cluster -Name <FailoverCluster-Name> -Node ("<node1>","<node2>") –StaticAddress <n.n.n.n> -NoStorage
```   

### Create a cloud witness

Cloud Witness is a new type of cluster quorum witness stored in an Azure Storage Blob. This removes the need of a separate VM hosting a witness share.

1. [Create a cloud witness for the failover cluster](http://technet.microsoft.com/windows-server-docs/failover-clustering/deploy-cloud-witness).

1. Create a blob container.

1. Save the access keys and the container URL.

1. Configure the failover cluster cluster quorum witness. See, [Configure the quorum witness in the user interface].(http://technet.microsoft.com/windows-server-docs/failover-clustering/deploy-cloud-witness#to-configure-cloud-witness-as-a-quorum-witness) in the UI.

### Add storage

The disks for S2D need to be empty and without partitions or other data. To clean disks follow [the steps in this guide](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-34-clean-disks).

1. [Enable Store Spaces Direct \(S2D\)](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-35-enable-storage-spaces-direct).

   The following PowerShell enables storage spaces direct.  

   ```PowerShell
   Enable-ClusterS2D
   ```

   In **Failover Cluster Manager**, you can now see the storage pool.

1. [Create a volume](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct#step-36-create-volumes).

   One of the features of S2D is that it automatically creates a storage pool when you enable it. You are now ready to create a volume. The PowerShell commandlet `New-Volume` automates the volume creation process, including formatting, adding to the cluster, and creating a cluster shared volume (CSV). The following example creates an 800 gigabyte (GB) CSV.

   ```PowerShell
   New-Volume -StoragePoolFriendlyName S2D* -FriendlyName VDisk01 -FileSystem CSVFS_REFS -Size 800GB
   ```   

   After this command completes, an 800 GB volume is mounted as a cluster resource. The volume is at `C:\ClusterStorage\Volume1\`.

   The following diagram shows a cluster shared volume with S2D:

   ![ClusterSharedVolume](./media/virtual-machines-windows-portal-sql-create-failover-cluster/15-cluster-shared-volume.png)

## Step 3: Test failover cluster failover

In Failover Cluster Manager, verify that you can move the storage resource to the other cluster node. If you can connect to the failover cluster with **Failover Cluster Manager** and move the storage from one node to the other, you are ready to configure the FCI.

## Step 4: Create SQL Server FCI

After you have configured the failover cluster and all cluster components including storage, you can create the SQL Server FCI.

1. Connect to the first virtual machine with RDP.

1. In **Failover Cluster Manager**, make sure all cluster core resources are on the first virtual machine. If necessary, move all resources to this virtual machine.

1. Locate the installation media. If the virtual machine uses one of the Azure Marketplace images, the media is located at `C:\SQLServer_<version number>_Full`. Click **Setup**.

1. In the **SQL Server Installation Center**, click **Installation**.

1. Click **New SQL Server failover cluster installation**. Follow the instructions in the wizard to install the SQL Server FCI.

   The FCI data directories need to be on clustered storage. With S2D, it's not a shared disk, but a mount point to a volume on each server. S2D synchronizes the volume between both nodes. The volume is presented to the cluster as a cluster shared volume. Use the CSV mount point for the data directories.

   ![DataDirectories](./media/virtual-machines-windows-portal-sql-create-failover-cluster/20-data-dicrectories.png)

1. After you complete the wizard, Setup will install a SQL Server FCI on the first node.

1. After Setup successfully installs the FCI on the first node, connect to the second node with RDP.

1. Open the **SQL Server Installation Center**. Click **Installation**.

1. Click **Add node to a SQL Server failover cluster**. Follow the instructions in the wizard to install SQL server and add this server to the FCI.

   >[!NOTE]
   >If you used an Azure Marketplace gallery image with SQL Server, SQL Server tools were included with the image. If you did not use this image, install the SQL Server tools separately. See [Download SQL Server Management Studio (SSMS)](http://msdn.microsoft.com/library/mt238290.aspx).

## Step 5: Create Azure load balancer

On Azure virtual machines, clusters use a load balancer to hold an IP address that needs to be on one cluster node at a time. In this solution, the load balancer holds the IP address for the SQL Server FCI.

[Create and configure an Azure load balancer](virtual-machines-windows-portal-sql-availability-group-tutorial.md#configure-internal-load-balancer).

### Create the load balancer in the Azure portal

To create the load balancer:

1. In the Azure portal, go to the Resource Group with the virtual machines.

1. Click **+ Add**. Search the Marketplace for **Load Balancer**. Click **Load Balancer**.

1. Click **Create**.

1. Configure the load balancer with:

   - **Name**: A name that identifies the load balancer.
   - **Type**: The load balancer can be either public or private. A private load balancer can be accessed from within the same VNET. Most Azure applications can use a private load balancer. If your application needs access to SQL Server directly over the Internet, use a public load balancer.
   - **Virtual Network**: The same network as the virtual machines.
   - **Subnet**: The same subnet as the virtual machines.
   - **Private IP address**: The same IP address that you assigned to the SQL Server FCI cluster network resource.
   - **subscription**: Your Azure subscription.
   - **Resource Group**: Use the same resource group as your virtual machines.
   - **Location**: Use the same Azure location as your virtual machines.
   See the following picture:

   ![CreateLoadBalancer](./media/virtual-machines-windows-portal-sql-create-failover-cluster/30-load-balancer-create.png)

### Configure the load balancer backend pool

1. Return to the Azure Resource Group with the virtual machines and locate the new load balancer. You may have to refresh the view on the Resource Group. Click the load balancer.

1. On the load balancer blade, click **Backend pools**.

1. Click **+ Add** to add a backend pool.

1. Type a name for the backend pool.

1. Click **Add a virtual machine**.

1. On the **Choose virtual machines** blade, click **Choose an availability set**.

1. Choose the availability set that you placed the SQL Server virtual machines in.

1. On the **Choose virtual machines** blade, click **Choose the virtual machines**.

   Your Azure portal should look like the following picture:

   ![CreateLoadBalancerBackEnd](./media/virtual-machines-windows-portal-sql-create-failover-cluster/33-load-balancer-back-end.png)

1. Click **Select** on the **Choose virtual machines** blade.

1. Click **OK** twice.

### Configure a load balancer health probe

1. On the load balancer blade, click **Health probes**.

1. Click **+ Add**.

1. On the **Add health probe** blade, <a name="probe"></a>Set the health probe parameters:

   - **Name**: A name for the health probe.
   - **Protocol**: TCP.
   - **Port**: Set to an available TCP port. This port requires an open firewall port. Use the [same port](#ports) you set for the health probe at the firewall.
   - **Interval**: 5 Seconds.
   - **Unhealthy threshold**: 2 consecutive failures.

1. Click OK.

### Set load balancing rules

1. On the load balancer blade, click **Load balancing rules**.

1. Click **+ Add**.

1. Set the load balancing rules parameters:

   - **Name**: A name for the load balancing rules.
   - **Frontend IP address**: Use the IP address for the SQL Server FCI cluster network resource.
   - **Port**: Set for the SQL Server FCI TCP port. The default instance port is 1433.
   - **Backend port**: This value uses the same port as the **Port** value when you enable **Floating IP (direct server return)**.
   - **Backend pool**: Use the backend pool name that you configured earlier.
   - **Health probe**: Use the health probe that you configured earlier.
   - **Session persistence**: None.
   - **Idle timeout (minutes)**: 4.
   - **Floating IP (direct server return)**: Enabled

1. Click **OK**.

## Step 6: Configure cluster for probe

Set the cluster probe port parameter in PowerShell.

To set the cluster probe port parameter, update variables in the following script from your environment.

  ```PowerShell
   $ClusterNetworkName = "<Cluster Network Name>" # the cluster network name (Use Get-ClusterNetwork on Windows Server 2012 of higher to find the name).
   $IPResourceName = "IP Address Resource Name" # the IP Address cluster resource name.
   $ILBIP = "<10.0.0.x>" # the IP Address of the Internal Load Balancer (ILB). This is the static IP address for the load balancer you configured in the Azure portal.
   [int]$ProbePort = <59999>

   Import-Module FailoverClusters

   Get-ClusterResource $IPResourceName | Set-ClusterParameter -Multiple @{"Address"="$ILBIP";"ProbePort"=$ProbePort;"SubnetMask"="255.255.255.255";"Network"="$ClusterNetworkName";"EnableDhcp"=0}
   ```


## Step 7: Test FCI failover

Test failover of the FCI to validate cluster functionality. Do the following steps:

1. Connect to one of the SQL Server FCI cluster nodes with RDP.

1. Open **Failover Cluster Manager**. Click **Roles**. Notice which node owns the SQL Server FCI role.

1. Right-click the SQL Server FCI role.

1. Click **Move** and click **Best Possible Node**.

**Failover Cluster Manager** shows the role and its resources go offline. The resources then move and come online on the other node.

### Test connectivity

To test connectivity, log in to another virtual machine in the same virtual network. Open **SQL Server Management Studio** and connect to the SQL Server FCI name.

>[!NOTE]
>If necessary, you can [download SQL Server Management Studio](http://msdn.microsoft.com/library/mt238290.aspx).

## Limitations
On Azure virtual machines, Microsoft Distributed Transaction Coordinator (DTC) is not supported on FCIs because the RPC port is not supported by the load balancer.

## See Also

[Setup S2D with remote desktop (Azure)](http://technet.microsoft.com/windows-server-docs/compute/remote-desktop-services/rds-storage-spaces-direct-deployment)

[Hyper-converged solution with storage spaces direct](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/hyper-converged-solution-using-storage-spaces-direct).

[Storage Space Direct Overview](http://technet.microsoft.com/windows-server-docs/storage/storage-spaces/storage-spaces-direct-overview)

[SQL Server support for S2D](https://blogs.technet.microsoft.com/dataplatforminsider/2016/09/27/sql-server-2016-now-supports-windows-server-2016-storage-spaces-direct/)
