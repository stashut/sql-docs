---
title: Create an FCI with a premium file share
description: "Use a premium file share (PFS) to create a failover cluster instance (FCI) with SQL Server on Azure virtual machines."
author: AbdullahMSFT
ms.author: amamun
ms.reviewer: mathoma, randolphwest
ms.date: 10/02/2023
ms.service: virtual-machines-sql
ms.subservice: hadr
ms.topic: how-to
ms.custom:
  - na
  - devx-track-azurepowershell
editor: monicar
tags: azure-service-management
---
# Create an FCI with a premium file share (SQL Server on Azure VMs)

[!INCLUDE [appliesto-sqlvm](../../includes/appliesto-sqlvm.md)]

This article explains how to create a failover cluster instance (FCI) with SQL Server on Azure Virtual Machines (VMs) by using a [premium file share](/azure/storage/files/storage-how-to-create-file-share).

Premium file shares are SSD backed and provide consistently low-latency file shares that are fully supported for use with failover cluster instances for SQL Server 2012 or later on Windows Server 2012 or later. Premium file shares give you greater flexibility, allowing you to resize and scale a file share without any downtime.

To learn more, see an overview of [FCI with SQL Server on Azure VMs](failover-cluster-instance-overview.md) and [cluster best practices](hadr-cluster-best-practices.md).

> [!NOTE]  
> It's now possible to lift and shift your failover cluster instance solution to SQL Server on Azure VMs using Azure Migrate. See [Migrate failover cluster instance](../../migration-guides/virtual-machines/sql-server-failover-cluster-instance-to-sql-on-azure-vm.md) to learn more.

## Prerequisites

Before you complete the instructions in this article, you should already have:

- An Azure subscription.
- An account that has permissions to create objects on both Azure virtual machines and in Active Directory.
- [Two or more prepared Azure Windows virtual machines](failover-cluster-instance-prepare-vm.md) in an [availability set](/azure/virtual-machines/windows/tutorial-availability-sets#create-an-availability-set) or different [availability zones](/azure/virtual-machines/windows/create-portal-availability-zone#confirm-zone-for-managed-disk-and-ip-address).
- A [premium file share](/azure/storage/files/storage-how-to-create-file-share) to be used as the clustered drive, based on the storage quota of your database for your data files.
- The latest version of [PowerShell](/powershell/azure/install-az-ps).

[!INCLUDE[tip-for-multi-subnet-ag](../../includes/virtual-machines-fci-multi-subnet.md)]

## Mount premium file share

To mount your premium file share, follow these steps:

1. Sign in to the [Azure portal](https://portal.azure.com), and go to your storage account.
1. Go to **File shares** under **Data storage**, and then select the premium file share you want to use for your SQL storage.
1. Select **Connect** to bring up the connection string for your file share.
1. In the dropdown list, select the drive letter you want to use, choose **Storage account key** as the authentication method, and then copy the code block to a text editor, such as Notepad.

   :::image type="content" source="media/failover-cluster-instance-premium-file-share-manually-configure/premium-file-storage-commands.png" alt-text="Screenshot showing how to copy the PowerShell command from the file share connect portal.":::

1. Use Remote Desktop Protocol (RDP) to connect to the SQL Server VM with the **account that your SQL Server FCI will use for the service account**.
1. Open an administrative PowerShell command console.
1. Run the command that you copied earlier to your text editor from the File share portal.
1. Go to the share by using either File Explorer or the **Run** dialog box (Windows + R on your keyboard). Use the network path `\\storageaccountname.file.core.windows.net\filesharename`. For example, `\\sqlvmstorageaccount.file.core.windows.net\sqlpremiumfileshare`
1. Create at least one folder on the newly connected file share to place your SQL data files into.
1. Repeat these steps on each SQL Server VM that will participate in the cluster.

  > [!IMPORTANT]  
  > Consider using a separate file share for backup files to save the input/output operations per second (IOPS) and space capacity of this share for data and log files. You can use either a Premium or Standard File Share for backup files.

## Create Windows Failover Cluster

The steps to create your Windows Server Failover Cluster differ between single subnet and multi-subnet environments. To create your cluster, follow the steps in the tutorial for either a [multi-subnet scenario](availability-group-manually-configure-tutorial-multi-subnet.md#add-failover-cluster-feature) or a [single subnet scenario](availability-group-manually-configure-tutorial-single-subnet.md#create-the-cluster). Though these tutorials create an availability group, the steps to create the cluster are the same for a failover cluster instance. 

## Configure quorum

The cloud witness is the recommended quorum solution for this type of cluster configuration for SQL Server on Azure VMs.

If you have an even number of votes in the cluster, configure the [quorum solution](hadr-cluster-quorum-configure-how-to.md) that best suits your business needs. For more information, see [Quorum with SQL Server VMs](hadr-windows-server-failover-cluster-overview.md#quorum).

## Validate cluster

Validate the cluster on one of the virtual machines by using the Failover Cluster Manager UI or PowerShell.

To validate the cluster by using the UI, do the following on one of the virtual machines:

1. In **Server Manager**, select **Tools**, and then select **Failover Cluster Manager**.
1. Right-click the cluster in **Failover Cluster Manager**, select **Validate Cluster** to open the **Validate a Configuration Wizard**.
1. On the **Validate a Configuration Wizard**, select **Next**.
1. On the **Select Servers or a Cluster** page, enter the names of both virtual machines.
1. On the **Testing options** page, select **Run only tests I select**.
1. Select **Next**.
1. On the  **Test Selection** page, select all tests except for **Storage** and **Storage Spaces Direct**: 

   :::image type="content" source="media/failover-cluster-instance-premium-file-share-manually-configure/cluster-validation.png" alt-text="Screenshot showing how to select cluster validation tests.":::

1. Select **Next**.
1. On the **Confirmation** page, select **Next**.  The **Validate a Configuration** wizard runs the validation tests.


To validate the cluster by using PowerShell, run the following script from an administrator PowerShell session on one of the virtual machines:

```powershell
Test-Cluster –Node ("<node1>","<node2>") –Include "Inventory", "Network", "System Configuration"
```

## Test cluster failover

Test the failover of your cluster. In **Failover Cluster Manager**, right-click your cluster, select **More Actions** > **Move Core Cluster Resource** > **Select node**, and then select the other node of the cluster. Move the core cluster resource to every node of the cluster, and then move it back to the primary node. If you can successfully move the cluster to each node, you're ready to install SQL Server.

:::image type="content" source="media/failover-cluster-instance-premium-file-share-manually-configure/test-cluster-failover.png" alt-text="Screenshot showing how to test cluster failover by moving the core resource to the other nodes.":::

## Create SQL Server FCI

After you've configured the failover cluster and all cluster components, including storage, you can create the SQL Server FCI.

### Create first node in the SQL FCI

To create the first node in the SQL Server FCI, follow these steps:

1. Connect to the first virtual machine by using RDP or Bastion.

1. In **Failover Cluster Manager**, make sure that all the core cluster resources are on the first virtual machine. If necessary, move all resources to this virtual machine.

1. If the version of the operating system is Windows Server 2019 and the Windows Cluster was created using the default [**Distributed Network Name (DNN)**](https://blogs.windows.com/windows-insider/2018/08/14/announcing-windows-server-2019-insider-preview-build-17733/), then the FCI installation for SQL Server 2017 and below fails with the error `The given key was not present in the dictionary`.

    During installation, SQL Server setup queries for the existing Virtual Network Name (VNN) and doesn't recognize the Windows Cluster DNN. The issue has been fixed in SQL Server 2019 setup. For SQL Server 2017 and below, follow these steps to avoid the installation error:

    - In Failover Cluster Manager, connect to the cluster, right-click  **Roles** and select **Create Empty Role**.
    - Right-click the newly created empty role, select **Add Resource**, and select **Client Access Point**.
    - Enter any name and complete the wizard to create the **Client Access Point**.
    - After the SQL Server FCI installation completes, the role containing the temporary **Client Access Point** can be deleted.

1. Locate the installation media. If the virtual machine uses one of the Azure Marketplace images, the media is located at `C:\SQLServer_<version number>_Full`.

1. Select **Setup**.

1. In the **SQL Server Installation Center**, select **Installation**.

1. Select **New SQL Server failover cluster installation**, and then follow the instructions in the wizard to install the SQL Server FCI.

1. On the **Cluster Network Configuration** page, the IP you provide varies depending on if your SQL Server VMs were deployed to a single subnet, or multiple subnets.

   1. For a **single subnet environment**, provide the IP address that you plan to add to the [Azure Load Balancer](failover-cluster-instance-vnn-azure-load-balancer-configure.md)
   1. For a **multi-subnet environment**, provide the secondary IP address in the subnet of the _first_ SQL Server VM that you previously designated as the [IP address of the failover cluster instance network name](failover-cluster-instance-prepare-vm.md#assign-secondary-ip-addresses):

   :::image type="content" source="media/failover-cluster-instance-azure-shared-disk-manually-configure/sql-install-cluster-network-secondary-ip-vm-1.png" alt-text="Screenshot of the secondary IP address in the subnet of the first VM.":::

1. In **Database Engine Configuration**, the data directories need to be on the premium file share. Enter the full path of the share, in this format: `\\storageaccountname.file.core.windows.net\filesharename\foldername`. A warning appears, telling you that you've specified a file server as the data directory. This warning is expected. Ensure that the user account you used to access the VM via RDP when you persisted the file share is the same account that the SQL Server service uses to avoid possible failures.

   :::image type="content" source="media/failover-cluster-instance-premium-file-share-manually-configure/use-file-share-as-data-directories.png" alt-text="Screenshot showing to use file share as SQL data directories.":::

1. After you complete the steps in the wizard, Setup installs a SQL Server FCI on the first node.

### Add additional nodes the SQL FCI

To add an additional node to the SQL Server FCI, follow these steps:

1. After FCI installation succeeds on the first node, connect to the second node by using RDP or Bastion.

1. Open the **SQL Server Installation Center**, and then select **Installation**.

1. Select **Add node to a SQL Server failover cluster**. Follow the instructions in the wizard to install SQL Server and add the node to the FCI.

1. For a multi-subnet scenario, in **Cluster Network Configuration**, enter the secondary IP address in the subnet of the _second_ SQL Server VM that you previously designated as the [IP address of the failover cluster instance network name](failover-cluster-instance-prepare-vm.md#assign-secondary-ip-addresses)

   :::image type="content" source="media/failover-cluster-instance-azure-shared-disk-manually-configure/sql-install-cluster-network-secondary-ip-vm-2.png" alt-text="Screenshot entering the secondary IP address in the subnet of the second SQL Server VM subnet.":::

   After selecting **Next** in **Cluster Network Configuration**, setup shows a dialog box indicating that SQL Server Setup detected multiple subnets as in the example image. Select **Yes** to confirm.

   :::image type="content" source="media/failover-cluster-instance-azure-shared-disk-manually-configure/sql-install-multi-subnet-confirmation.png" alt-text="Screenshot showing multi subnet confirmation.":::

1. After you complete the instructions in the wizard, setup adds the second SQL Server FCI node.

1. Repeat these steps on any other nodes that you want to add to the SQL Server failover cluster instance.

> [!NOTE]  
> Azure Marketplace gallery images come with SQL Server Management Studio installed. If you didn't use a marketplace image [Download SQL Server Management Studio (SSMS)](/sql/ssms/download-sql-server-management-studio-ssms).

## Register with SQL IaaS Agent extension

To manage your SQL Server VM from the portal, register it with the [SQL IaaS Agent extension](sql-agent-extension-manually-register-single-vm.md). 

> [!NOTE]
> At this time, SQL Server failover cluster instances on Azure virtual machines registered with the SQL IaaS Agent extension only support a [limited](failover-cluster-instance-overview.md#limited-extension-support) number of features available through basic registration, and not those that require the agent, such as automated backup, patching, Microsoft Entra authentication and advanced portal management. See the [table of benefits](sql-server-iaas-agent-extension-automate-management.md#feature-benefits) to learn more. 

Register a SQL Server VM with PowerShell (-LicenseType can be `PAYG` or `AHUB`):

```powershell-interactive
# Get the existing compute VM
$vm = Get-AzVM -Name <vm_name> -ResourceGroupName <resource_group_name>

# Register SQL VM with SQL IaaS Agent extension
New-AzSqlVM -Name $vm.Name -ResourceGroupName $vm.ResourceGroupName -Location $vm.Location `
   -LicenseType <license_type>
```

## Configure connectivity

If you deployed your SQL Server VMs in multiple subnets, skip this step. If you deployed your SQL Server VMs to a single subnet, then you need to configure an additional component to route traffic to your FCI. You can configure a virtual network name (VNN) with an Azure Load Balancer, or a distributed network name for a failover cluster instance. [Review the differences between the two](hadr-windows-server-failover-cluster-overview.md#virtual-network-name-vnn) and then deploy either a [distributed network name](failover-cluster-instance-distributed-network-name-dnn-configure.md) or a [virtual network name and Azure Load Balancer](failover-cluster-instance-vnn-azure-load-balancer-configure.md) for your failover cluster instance.

## Limitations

- FILESTREAM isn't supported for a failover cluster with a premium file share. To use filestream, deploy your cluster by using [Storage Spaces Direct](failover-cluster-instance-storage-spaces-direct-manually-configure.md) or [Azure shared disks](failover-cluster-instance-azure-shared-disks-manually-configure.md) instead.
- Database Snapshots aren't currently supported with [Azure Files due to sparse files limitations](/rest/api/storageservices/features-not-supported-by-the-azure-file-service).
- Since database snapshots aren't supported, CHECKDB for user databases falls back to CHECKDB WITH TABLOCK. TABLOCK limits the checks that are performed - DBCC CHECKCATALOG isn't run on the database, and Service Broker data isn't validated.
- DBCC CHECKDB on `master` and `msdb` database isn't supported.
- Databases that use the in-memory OLTP feature aren't supported on a failover cluster instance deployed with a premium file share. If your business requires in-memory OLTP, consider deploying your FCI with [Azure shared disks](failover-cluster-instance-azure-shared-disks-manually-configure.md) or [Storage Spaces Direct](failover-cluster-instance-storage-spaces-direct-manually-configure.md) instead.
- Microsoft Distributed Transaction Coordinator (MSDTC) is not supported by SQL Server on Azure VM failover cluster instances deployed to premium file shares. Review [FCI limitations](failover-cluster-instance-overview.md#limitations) for more information.
- Microsoft Distributed Transaction Coordinator (MSDTC) is supported on Azure virtual machines starting with Windows Server 2019 and later when deployed to dedicated Clustered Shared Volumes (CSVs) and using a [standard load balancer](/azure/load-balancer/load-balancer-overview). MSDTC is not supported on Windows Server 2016 and earlier.

[!INCLUDE [virtual-machines-fci-limitations](../../includes/virtual-machines-fci-limitations.md)]


## Related content

- [Create an FCI with Azure shared disks (SQL Server on Azure VMs)](failover-cluster-instance-azure-shared-disks-manually-configure.md)
- [Create an FCI with Storage Spaces Direct (SQL Server on Azure VMs)](failover-cluster-instance-storage-spaces-direct-manually-configure.md)
- [Windows Server Failover Cluster with SQL Server on Azure VMs](hadr-windows-server-failover-cluster-overview.md)
- [Failover cluster instances with SQL Server on Azure Virtual Machines](failover-cluster-instance-overview.md)
- [Failover cluster instance overview](/sql/sql-server/failover-clusters/windows/always-on-failover-cluster-instances-sql-server)
- [HADR configuration best practices (SQL Server on Azure VMs)](hadr-cluster-best-practices.md)
