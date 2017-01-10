---
ms.assetid: 342173ca-4e10-44f4-b2c9-02a6c26f7a4a
title: Planning volumes in Storage Spaces Direct
ms.prod: windows-server-threshold
ms.author: cosdar
ms.manager: eldenc
ms.technology: storage-spaces
ms.topic: article
author: cosmosdarwin
ms.date: 01/11/2016
---

# Planning volumes in Storage Spaces Direct

> Applies To: Windows Server 2016

This topic provides guidance for how to plan volumes in Storage Spaces Direct to meet the performance and capacity needs of your workloads, including choosing their filesystem, resiliency type, and size.

## Review: What are volumes

Volumes are the datastores where you put the files your workloads need, such as VHD or VHDX files for Hyper-V virtual machines. Volumes combine the drives in the storage pool to introduce the fault tolerance, scalability, and performance benefits of Storage Spaces Direct.

   >[!NOTE]
   > Throughout documentation for Storage Spaces Direct, we use term "volume" to refer jointly to the volume and the virtual disk under it, including functionality provided by other built-in Windows features such as Cluster Shared Volumes (CSV) and ReFS. Understanding these implementation-level distinctions is not necessary to plan and deploy Storage Spaces Direct successfully.

![what-are-volumes](media/plan-volumes/what-are-volumes.png)

All volumes are accessible by all servers in the cluster at the same time. Once created, they show up at **C:\ClusterStorage\** on all servers.

![csv-folder-screenshot](media/plan-volumes/csv-folder-screenshot.png)

## Choosing how many volumes to create

We recommend making the number of volumes a multiple of the number of servers in your cluster. For example, if you have 4 servers, you will experience more consistent performance with 8 total volumes than with 7 or 9. This allows the cluster to distribute volume "ownership" (one server handles metadata orchestration for each volume) evenly among servers.

We recommend limiting the total number of volumes to 32 per cluster.

## Choosing the filesystem

We recommend using the new [Resilient File System (ReFS)](../refs/refs-overview.md) for Storage Spaces Direct.

ReFS is the premier filesystem purpose-built for virtualization and offers many advantages, including dramatic performance accelerations and built-in protection against data corruption. However, it does not yet support certain features, such as data deduplication.

If your workload requires a feature that ReFS doesn't support yet, you can use NTFS instead.

   >[!TIP]
   > Volumes with different filesystems can coexist in the same cluster.

## Choosing the resiliency type

Volumes in Storage Spaces Direct provide resiliency to protect against hardware problems, such as drive failures, and to enable continuous availability throughout server maintenance, such as during software updates. If you have four or more servers, you can choose between several resiliency types.

   >[!NOTE]
   > Which resiliency types you can choose is independent of which types of drives you have.

### With two servers

The only option for clusters with two servers is two-way mirroring. This keeps two copies of all data, one copy on the drives in each server. Its storage efficiency is 50% – to write 1 TB of data, you need at least 2 TB of physical storage capacity in the storage pool. Two-way mirroring can safely tolerate one hardware failure (drive or server) at a time.

![two-way-mirror](media/plan-volumes/two-way-mirror.png)

If you have more than two servers, we recommend using one of the following resiliency types instead.

### With three servers

With three servers, you should use three-way mirroring for better fault tolerance and performance. Three-way mirroring keeps three copies of all data, one copy on the drives in each server. Its storage efficiency is 33.3% – to write 1 TB of data, you need at least 3 TB of physical storage capacity in the storage pool. Three-way mirroring can safely tolerate [at least two hardware problems (drive or server) at a time](storage-spaces-fault-tolerance.md#examples). For example, if you're rebooting one server when suddenly another drive or server fails, all data remains safe and continuously accessible. 

![three-way-mirror](media/plan-volumes/three-way-mirror.png)

### With four or more servers

With four or more servers, you can choose for each volume whether to use three-way mirroring, dual parity (often called "erasure coding"), or mix the two.

Dual parity provides the same fault tolerance as three-way mirroring but with better storage efficiency. With four servers, its storage efficiency is 50.0% – to store 2 TB of data, you need 4 TB of physical storage capacity in the storage pool. This increases to 66.7% storage efficiency with seven servers, and continues up to 80.0% storage efficiency. The tradeoff is that parity encoding is more compute-intensive, which can limit its performance. For more details, see Fault tolerance and storage efficiency.

![dual-parity](media/plan-volumes/dual-parity.png)

Which resiliency type to use depends on the needs of your workload.

#### When performance matters most

Workloads that have strict latency requirements or that need lots of mixed random IOPS, such as SQL Server databases or performance-sensitive Hyper-V virtual machines, should run on volumes that use mirroring to maximize performance.

   >[!TIP]
   > Mirroring is faster than any other resiliency type. We use mirroring for nearly all our performance showcases.

#### When capacity matters most

Workloads that write infrequently, such as data warehouses or "cold" storage, should run on volumes that use dual parity to maximize storage efficiency. Certain other workloads, such as traditional file servers, virtual desktop infrastructure (VDI), or others that don’t create lots of fast-drifting random IO traffic and/or don’t require the best performance may also use dual parity, at your discretion. Parity inevitably increases CPU utilization and IO latency, particularly on writes, compared to mirroring.

#### When data is written in bulk

Workloads that write in large, sequential passes, such as archival or backup targets, have another option that is new in Windows Server 2016: one volume can mix mirroring and dual parity. Writes land first in the mirrored portion and are gradually moved into the parity portion later. This accelerates ingestion and reduces resource utilization when large writes arrive by allowing the compute-intensive parity encoding to happen over a longer time. When sizing the portions, consider that the quantity of writes that happen at once (such as one daily backup) should comfortably fit in the mirror portion. For example, if you ingest 100 GB once daily, consider using mirroring for 150 GB to 200 GB, and dual parity for the rest.

The resulting storage efficiency depends on the proportions you choose. See [this demo](https://www.youtube.com/watch?v=-LK2ViRGbWs&t=36m55s) for some examples.

   >[!TIP]
   > Volumes with different resiliency types can coexist in the same cluster.

### About deployments with NVMe, SSD, and HDD

In deployments with two types of drives, the faster drives provide caching while the slower drives provide capacity. This happens automatically – for more information, see [Understanding the cache in Storage Spaces Direct](understand-the-cache.md). In such deployments, all volumes ultimately reside on the same type of drives – the capacity drives.

In deployments with all three types of drives, only the fastest drives (NVMe) provide caching, leaving two types of drives (SSD and HDD) to provide capacity. For each volume, you can choose whether it resides entirely on the SSD tier, entirely on the HDD tier, or whether it spans the two.

   >[!IMPORTANT]
   > We recommend using the SSD tier to place your most performance-sensitive workloads on all-flash.

## Choosing the size of volumes

Volumes in Storage Spaces Direct can be any size up to 32 TB.

#### Size versus footprint

The size of a volume refers to its usable capacity, the amount of data it can store. This is provided by the **-Size** parameter of the **New-Volume** cmdlet and then appears in the **Size** property when you run the **Get-Volume** cmdlet. This is distinct from volume's **Footprint**, the total physical storage capacity it occupies on the storage pool. The footprint depends on its resiliency type.

For example, volumes that use three-way mirroring have a footprint three times their size.

The footprints of your volumes need to fit in the storage pool.

![size-versus-footprint](media/plan-volumes/size-versus-footprint.png)

#### Reserve capacity

Leaving some capacity in the storage pool unallocated gives volumes space to repair "in-place" after drives fail, improving data safety and performance. If there is sufficient capacity, an immediate, in-place, parallel repair can restore volumes to full resiliency even before the failed drives are replaced. This happens automatically.

We recommend reserving the equivalent of one capacity drive per server, up to 4 drives. You may reserve more at your discretion, but this minimum recommendation guarantees an immediate, in-place, parallel repair can succeed after the failure of any drive.

![reserve](media/plan-volumes/reserve.png)

For example, if you have 2 servers and you are using 1 TB capacity drives, set aside 2 x 1 = 2 TB of the pool as reserve. If you have 3 servers and 1 TB capacity drives, set aside 3 x 1 = 3 TB as reserve. If you have 4 or more servers and 1 TB capacity drives, set aside 4 x 1 = 4 TB as reserve.

   >[!NOTE]
   > In clusters with drives of all three types (NVMe + SSD + HDD), we recommend reserving the equivalent of one SSD plus one HDD per server, up to 4 drives of each maximum.

## Example: Capacity planning

Consider four servers, each with some cache and 16 x 2 TB drives for capacity.

```
4 servers x 16 drives each x 2 TB each = 128 TB
```

From this 128 TB, we set aside four drives, or 8 TB, so that in-place repairs can happen without any rush to replace drives after they fail. This leaves 120 TB of physical storage capacity in the pool with which we can create volumes.

```
128 TB – (4 x 2 TB) = 120 TB
```

Suppose we need our deployment to host some highly active Hyper-V virtual machines, but we also have lots of cold storage – old files and backups we need to retain. Because we have four servers, let's create four volumes.

Let's put the virtual machines on the first two volumes, *Volume1* and *Volume2*. We choose ReFS as the filesystem (for the faster creation and checkpoints) and three-way mirroring for resiliency to maximize performance. Let's put the cold storage on the other two volumes, *Volume 3* and *Volume 4*. We choose NTFS as the filesystem (for data deduplication) and dual parity for resiliency to maximize capacity.

We aren't required to make all volumes the same size, but for simplicity, let's – for example, we can make them all 12 TB.

*Volume1* and *Volume2* will each occupy 12 TB x 33.3% efficiency = 36 TB of physical storage capacity.

*Volume3* and *Volume4* will each occupy 12 TB x 50.0% efficiency = 24 TB of physical storage capacity.

```
36 TB + 36 TB + 24 TB + 24 TB = 120 TB
```

The four volumes fit exactly on the physical storage capacity available in our pool. Perfect!

![example](media/plan-volumes/example.png)

   >[!TIP]
   > You don't need to create all the volumes right away. You can leave some of your storage pool unallocated to extend volumes or create new volumes later.

## Usage

See [Creating volumes in Storage Spaces Direct](create-volumes.md).

### See also

- [Storage Spaces Direct overview](storage-spaces-direct-overview.md)
- [Choosing drives for Storage Spaces Direct](choosing-drives-and-resiliency-types.md)
- [Fault tolerance and storage efficiency](storage-spaces-fault-tolerance.md)