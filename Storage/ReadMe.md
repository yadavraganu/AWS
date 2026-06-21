## AWS Storage Options

* Block Storage (EBS / Instance Store): Direct OS mounting, high-speed transactional databases, raw virtual hard drives.
* Object Storage (S3): Unlimited flat scaling, internet-accessible, key-value storage for unstructured data.
* File Storage (EFS / FSx): Shared network file systems (NFS/SMB) accessible by thousands of instances simultaneously.
* Data Transfer & Hybrid: Edge computing devices and online synchronization tools bridging on-premises and AWS.

## 1. Block Storage: EBS vs. Instance Store

| Feature | Amazon EBS (Elastic Block Store) | Amazon EC2 Instance Store |
|---|---|---|
| Persistence | Persistent. Lives independently of the EC2 lifecycle. | Ephemeral. Wiped if stopped or host fails. |
| Connection | Network-attached (Storage Area Network). | Physically attached to the host blade. |
| Max IOPS | Up to 256,000 IOPS (io2 Block Express). | Millions of IOPS (Ultra-low NVMe latency). |
| Modification | Elastic. Scale capacity or IOPS on the fly. | Fixed size determined by the EC2 instance type. |
| Backups | Native automated, incremental S3 snapshots. | Requires manual OS/application-level scripting. |
| Use Cases | Boot volumes, relational databases (RDS, Postgres). | Caches, scratch space, replicated NoSQL. |

## EBS Volume Type Breakdown

* gp3 / gp2 (General Purpose SSD): Baseline 3,000 IOPS included. Best for boot volumes, virtual desktops, and dev environments.
* io2 / io1 (Provisioned IOPS SSD): Sub-millisecond latency. Best for IOPS-intensive critical databases (SAP HANA, Oracle).
* st1 (Throughput Optimized HDD): Low-cost magnetic. Best for sequential big data, MapReduce, log processing.
* sc1 (Cold HDD): Lowest block cost. Best for infrequently accessed legacy data.

## 2. Object Storage: Amazon S3 (Simple Storage Service)

* Architecture: Serverless, bucket-based object storage with 99.999999999% (11 9s) durability.
* Features: Object locking (WORM), versioning, lifecycle rules, and server-side KMS encryption.

## S3 Storage Classes

* S3 Standard: Default class. Millisecond access, high throughput. Best for active data and web assets.
* S3 Intelligent-Tiering: Automatically moves data between frequent/infrequent tiers. Best for data with unknown access patterns.
* S3 Standard-IA (Infrequent Access): Lower storage cost, high retrieval fee. Best for long-lived data needed quickly when requested.
* S3 One Zone-IA: Stored in a single AZ (30% cheaper, less resilient). Best for reproducible secondary backups.
* S3 Glacier Instant Retrieval: Millisecond retrieval. Best for archives accessed a few times a year.
* S3 Glacier Flexible Retrieval: Retrieval takes 1 minute to 5 hours. Best for compliance or medical records.
* S3 Glacier Deep Archive: Cheapest storage option. Retrieval takes 12 hours. Best for multi-year regulatory archives.

## 3. File Storage: EFS vs. FSx Network File Systems

* Amazon EFS (Elastic File System): Linux-native (NFSv4). Serverless, autoscaling, mountable across thousands of EC2 instances and on-premises servers via Direct Connect.
* Amazon FSx for Windows File Server: Windows-native (SMB). Fully integrated with Active Directory, Microsoft ACLs, and shadow copies.
* Amazon FSx for Lustre: Ultra-high-performance parallel file system. Best for high-performance computing (HPC), AI/ML training, and financial modeling.
* Amazon FSx for NetApp ONTAP / OpenZFS: Managed enterprise storage arrays. Best for migrating existing complex enterprise file storage layouts directly to AWS without rewriting code.

## 4. Data Transfer & Edge Connectivity

* AWS Storage Gateway: Hybrid cloud tool making S3 appear as a local file share (File Gateway), local volume (Volume Gateway), or local tape library (Tape Gateway).
* AWS DataSync: Automated service to accelerate moving data between on-premises storage arrays and S3, EFS, or FSx over the internet or Direct Connect.
* AWS Snow Family (Snowcone, Snowball, Snowmobile): Physical ruggedized hardware devices shipped to your data center to securely move petabytes of offline data into AWS.

