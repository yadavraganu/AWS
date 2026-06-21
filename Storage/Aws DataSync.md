# What Is AWS DataSync?

AWS DataSync is a managed online data transfer service that automates, accelerates, and secures the movement of file and object data between on-premises storage systems, AWS storage services, edge locations, and even other cloud providers. It handles infrastructure setup, encryption, and data integrity validation so you can focus on migrating, replicating, or processing your data rather than managing transfer workflows.

## Key Concepts

- Agent  
  A lightweight VM appliance you deploy on-premises or in your VPC that reads and writes data during transfers. Agents communicate securely with the DataSync service over TLS to orchestrate tasks.

- Location  
  A logical endpoint representing where your data lives—this can be an NFS/SMB file server, HDFS cluster, S3 bucket, EFS file system, FSx file server, or storage in another cloud.

- Task  
  The core resource that defines a data transfer workflow. A task ties together a source location, a destination location, filter rules, and settings for validation, encryption, and bandwidth usage.

- Task Execution  
  An individual run of a task on DataSync’s fully managed infrastructure, moving data according to your configuration while performing automatic retries and integrity checks.

## Supported Storage Systems

| Category                  | Examples                                                                 |
|---------------------------|--------------------------------------------------------------------------|
| On-Premises File Storage  | NFS, SMB, HDFS                                                          |
| AWS Storage Services      | S3, EFS, FSx for Windows File Server, FSx for Lustre, FSx for OpenZFS, FSx for NetApp ONTAP |
| Other Cloud Providers     | Google Cloud Storage, Azure Blob & Files, Wasabi, Backblaze B2, Cloudflare R2, Oracle Cloud, IBM Cloud, Alibaba OSS, NAVER Cloud |
| Edge & Snowball Devices   | S3-compatible storage on Snowball Edge                                   |

This broad interoperability lets you build hybrid-cloud architectures or migrate entire repositories with minimal coding.

## Common Use Cases

- Migrate Active Datasets  
  Rapidly move terabytes to petabytes of on-premises or edge data into Amazon S3, EFS, or FSx, with built-in encryption and validation.

- Archive Cold Data  
  Shift infrequently accessed data to cost-effective long-term storage classes like S3 Glacier Flexible Retrieval or Deep Archive to free up local capacity.

- Replicate for Disaster Recovery  
  Continuously copy data into standby file systems (EFS, FSx) or cross-region S3 buckets to support failover and business continuity.

- Hybrid-Cloud Processing  
  Transfer logs, media, or big-data inputs into AWS for machine learning, analytics, or video rendering workloads, then export results back on-premises or to another cloud.

## How AWS DataSync Works

1. You deploy an agent in your environment (on-premises VM or EC2 in a VPC).  
2. You define “locations” for your source and destination storage endpoints.  
3. You create a task that links those locations, sets filters (include/exclude patterns), and configures transfer options (encryption, bandwidth limits, metadata preservation).  
4. When you start a task execution, DataSync orchestrates transfers over TLS, automatically scales to use network capacity, validates checksums, and retries failed transfers.  
5. You monitor progress and metrics via the DataSync console or CloudWatch.  

All data remains in the AWS backbone when transferring between AWS Regions or services—never traversing the public Internet except for on-premises or other-cloud transfers.

## Benefits

- Fully Managed  
  No servers to provision or software to maintain; DataSync handles scaling, scheduling, and retries automatically.  

- Secure by Default  
  Data in transit is encrypted with TLS, and you can optionally encrypt at rest using AWS KMS keys.  

- High Performance  
  Optimized protocol and multi-threaded transfers can move up to hundreds of MB/s per agent, leveraging available bandwidth.  

- Data Integrity  
  End-to-end checksum validation ensures bytes on the destination match the source.  

- Cost-Effective  
  Pay only for the amount of data processed and transferred, with no upfront fees or minimums.  
