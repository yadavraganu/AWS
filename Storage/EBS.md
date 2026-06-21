## AWS EBS (Elastic Block Store)

AWS Elastic Block Store (EBS) provides **durable, block-level storage volumes** for use with Amazon EC2 instances. Think of it as a virtual hard drive in the cloud.

### Core Concepts & Essentials

* **Architecture:** EBS volumes are network-attached storage (NAS) that persist independently from the life of an EC2 instance.
* **Availability:** Automatically replicated within its **Availability Zone (AZ)** to protect against component failure. It does *not* automatically replicate across different AZs.
* **Multi-Attach:** Some SSD-based EBS volumes can be attached to multiple EC2 instances concurrently (must be in the same AZ).
* **Encryption:** Uses AWS KMS (AES-256). Encrypting a volume encrypts its data at rest, I/O in transit, and all subsequent snapshots.

### EBS Volume Types Compared

EBS volumes are divided into two main categories: **SSD-backed** (for transactional workloads, high IOPS) and **HDD-backed** (for throughput-intensive workloads, large streaming data).

| Volume Type | API Name | Max Size | Max IOPS | Max Throughput | Ideal Use Cases |
| ----------- | -------- | -------- | -------- | -------------- | --------------- |
| **gp3** (General Purpose SSD) | `gp3` | 16 TiB | 16,000 | 1,000 MiB/s | Default choice. System boot volumes, virtual desktops, development environments. |
| **gp2** (General Purpose SSD) | `gp2` | 16 TiB | 16,000 | 250 MiB/s | Older generation. IOPS scales with size ($3 \text{ IOPS per GiB}$). Has burst buckets. |
| **io2 Block Express** (Provisioned IOPS) | `io2` | 64 TiB | 256,000 | 4,000 MiB/s | Sub-millisecond latency. Mission-critical databases (SAP HANA, Oracle, SQL Server). |
| **io1** (Provisioned IOPS SSD) | `io1` | 16 TiB | 64,000 | 1,000 MiB/s | Legacy high-performance SSD. |
| **st1** (Throughput Optimized HDD) | `st1` | 16 TiB | 500 | 500 MiB/s | Frequently accessed, throughput-intensive workloads like Big Data, MapReduce, Log processing. **Cannot be a boot volume.** |
| **sc1** (Cold HDD) | `sc1` | 16 TiB | 250 | 250 MiB/s | Less frequently accessed data, lowest cost. Archive/Backup storage. **Cannot be a boot volume.** |

### Snapshots & Data Protection

* **Point-in-Time Backups:** Snapshots are **incremental** backups, meaning only the blocks that have changed since your last snapshot are saved.
* **Storage Location:** Snapshots are stored in **Amazon S3** (but managed behind the scenes; you don't see them in your S3 buckets).
* **Cross-Region Copying:** You can copy snapshots across AWS Regions to create geographical backups or ease regional migrations.
* **Fast Snapshot Restore (FSR):** Eliminates the latency of I/O operations on blocks data when restoring a volume from a snapshot (removes the need to "warm" the volume but additional charges on enabled AZs).
* **EBS Data Lifecycle Manager (DLM):** Automates the creation, retention, and deletion of EBS snapshots via tags.

### EBS vs. Instance Store (Ephemeral Storage)

It is crucial to know the difference between network-attached EBS and physically attached Instance Stores.

| Feature | EBS Volume | Instance Store |
| --- | --- | --- |
| **Type** | Network-attached (Virtual) | Physically attached to the host |
| **Data Persistence** | Survives instance **Stop** and **Terminate** (if configured) | Data is lost if the instance is **Stopped** or **Terminated** |
| **Performance** | High, but subject to network bandwidth | Extremely high IOPS, very low latency |
| **Use Case** | Databases, OS boot volumes, long-term storage | Caches, buffers, scratch data, temporary state |

### Elastic Volumes (Modifying on the Fly)

Without detaching the volume or stopping the instance, you can dynamically:

1. Increase volume size.
2. Change the volume type (e.g., upgrading `gp2` to `gp3`).
3. Adjust IOPS or Throughput performance.

⚠️ **Note:** After modifying a volume, you must wait at least **6 hours** before applying another modification to the same volume. You also need to extend the file system inside the OS to recognize the new space.


### Monitoring & Performance Optimization

* **Amazon CloudWatch Metrics:**
* `VolumeReadBytes` / `VolumeWriteBytes`: Throughput performance.
* `VolumeReadOps` / `VolumeWriteOps`: Total IOPS.
* `VolumeQueueLength`: Number of read/write operations waiting to be issued. A high queue length usually means you need more IOPS.
* **EBS-Optimized Instances:** Ensure your EC2 instance type supports "EBS Optimization." This provides dedicated network bandwidth solely for EBS I/O traffic, preventing contention with your standard network traffic.
