# Amazon RDS — Master Notes

> Consolidated, interview-ready notes on Amazon RDS. Formatting standardized: `##` for top-level topics, `###`/`####` for sub-sections, tables for comparisons.

## Table of Contents

**Foundational**
1. [Introduction to RDS](#1-introduction-to-rds)
2. [Creating Your First RDS Instance](#2-creating-your-first-rds-instance)
3. [VPC, Subnets, and Security Groups](#3-vpc-subnets-and-security-groups)
4. [Connecting to RDS](#4-connecting-to-rds)
5. [Basic Operations](#5-basic-operations)

**Intermediate — Administration & Performance**
6. [Storage and Scaling](#6-storage-and-scaling)
7. [Monitoring and Metrics](#7-monitoring-and-metrics)
8. [Backups and Snapshots](#8-backups-and-snapshots)
9. [Security and Access Control](#9-security-and-access-control)
10. [Maintenance and Patching](#10-maintenance-and-patching)

**Advanced — Optimization & High Availability**
11. [Multi-AZ Deployments](#11-multi-az-deployments)
12. [Read Replicas](#12-read-replicas)
13. [Performance Tuning](#13-performance-tuning)
14. [Cost Optimization](#14-cost-optimization)

**Expert — Architecture & Automation**
15. [Aurora Deep Dive](#15-aurora-deep-dive)
16. [Disaster Recovery & DR Planning](#16-disaster-recovery--dr-planning)
17. [Infrastructure as Code](#17-infrastructure-as-code)
18. [Audit and Compliance](#18-audit-and-compliance)
19. [Migration Strategies](#19-migration-strategies)
20. [Common Interview Questions](#20-common-interview-questions)

---

## 1. Introduction to RDS

### What is Amazon RDS?
**Amazon Relational Database Service (RDS)** is a **managed relational database** service that handles the undifferentiated heavy lifting of running a database — provisioning, patching, backups, failover, and scaling — so you can focus on schema design and queries instead of database administration.

### Supported Database Engines
| Engine | Notes |
|---|---|
| **MySQL** | Widely used open-source engine; Aurora MySQL is a compatible, higher-performance alternative. |
| **PostgreSQL** | Popular open-source engine with rich feature/extension support; Aurora PostgreSQL is its compatible alternative. |
| **MariaDB** | MySQL-compatible fork. |
| **Oracle** | Commercial engine; supports "Bring Your Own License" (BYOL) or License Included pricing. |
| **SQL Server** | Microsoft's commercial engine; also supports BYOL or License Included. |
| **Amazon Aurora** | AWS-built, MySQL/PostgreSQL-compatible engine with cloud-native architecture (see [Section 15](#15-aurora-deep-dive)). |

### Use Cases and Benefits
- **Managed operations**: AWS handles OS/DB patching, automated backups, and hardware provisioning.
- **High availability**: built-in **Multi-AZ** support for automatic failover ([Section 11](#11-multi-az-deployments)).
- **Scalability**: vertical scaling (bigger instance) and horizontal read scaling (read replicas) ([Sections 6, 12](#6-storage-and-scaling)).
- **Security**: encryption at rest/in transit, VPC isolation, IAM integration ([Section 9](#9-security-and-access-control)).
- **Typical use cases**: transactional (OLTP) application backends, content management systems, e-commerce platforms — any workload needing a traditional relational/SQL data model with ACID guarantees.

### RDS vs Self-Managed Database on EC2

| Aspect | Amazon RDS | Self-Managed on EC2 |
|---|---|---|
| OS/DB patching | Managed by AWS | You manage |
| Backups | Automated, built-in | You configure and manage |
| Failover | Built-in (Multi-AZ) | You build it yourself |
| OS-level access | ❌ No (no SSH to the DB host) | ✅ Full control |
| Custom DB engine versions/plugins | Limited to what RDS supports | Full flexibility |
| Best for | Most standard relational workloads | Edge cases needing OS-level access or unsupported engines/extensions |

---

## 2. Creating Your First RDS Instance

### Launching an RDS Instance via AWS Console
1. Navigate to **RDS Console → Create database**.
2. Choose a creation method: **Standard create** (full control over every setting) or **Easy create** (sensible defaults for quick setup).
3. Choose the **engine** (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, or Aurora) and version.
4. Choose a **template**: Production (Multi-AZ, provisioned IOPS defaults), Dev/Test, or Free Tier.
5. Set **DB instance identifier**, master username, and master password (or let AWS manage the password via **Secrets Manager integration**).
6. Choose **instance type** (see below) and **storage** (type and size — see [Section 6](#6-storage-and-scaling)).
7. Configure **connectivity**: VPC, subnet group, public accessibility (Yes/No), and security group(s) ([Section 3](#3-vpc-subnets-and-security-groups)).
8. Configure additional settings: initial database name, backup retention, monitoring, maintenance window, deletion protection.
9. Click **Create database** — provisioning typically takes several minutes.

### Choosing Engine, Instance Type, Storage

#### Instance Type (Class) Families
| Family | Optimized For | Example Use Case |
|---|---|---|
| `db.t` (Burstable) | Low-cost, variable CPU workloads | Dev/test, light production workloads |
| `db.m` (General Purpose) | Balanced compute/memory | Typical application databases |
| `db.r` (Memory Optimized) | High memory-to-vCPU ratio | Large in-memory working sets, analytics-heavy OLTP |
| `db.x` (High Memory) | Very large memory footprints | Large-scale enterprise workloads, SAP HANA-class needs |

#### Storage Type
See [Section 6](#6-storage-and-scaling) for full detail on gp2/gp3/io1/io2/magnetic.

---

## 3. VPC, Subnets, and Security Groups

### What is a Subnet Group in RDS?
An **RDS subnet group** is a **collection of subnets** (usually in different Availability Zones within a Region) that you define for your RDS database instances. When you create an RDS instance, you **must specify a subnet group** so that AWS knows where to place the instance within your VPC.

### Why Are Subnet Groups Needed?

1. **High Availability (Multi-AZ deployments)**:
   - RDS uses subnet groups to place primary and standby instances in **different Availability Zones** for fault tolerance.
   - This ensures that if one AZ goes down, the standby in another AZ can take over.

2. **VPC Integration**:
   - RDS instances run inside a VPC, and subnet groups define **which subnets** (and hence which AZs) the instance can use.
   - This allows you to control **network access**, **routing**, and **security**.

3. **Isolation and Security**:
   - You can place RDS instances in **private subnets** to restrict internet access.
   - Subnet groups help enforce **network segmentation** and **security boundaries**.

4. **Flexibility in Deployment**:
   - You can create different subnet groups for different environments (e.g., dev, test, prod).
   - This helps in managing resources and access control more effectively.

### Example Scenario

Suppose you have a VPC with three subnets in three different AZs:
- `subnet-a` in `us-east-1a`
- `subnet-b` in `us-east-1b`
- `subnet-c` in `us-east-1c`

You create an RDS subnet group including these three subnets. When you launch a Multi-AZ RDS instance, AWS will place the primary in one AZ (say `us-east-1a`) and the standby in another (say `us-east-1b`), using the subnets you defined.

> A subnet group needs subnets in **at least 2 AZs** — this is enforced even for a Single-AZ instance, since you might enable Multi-AZ later without re-architecting networking.

### Security Groups for RDS
- RDS instances are protected by **VPC security groups** (stateful, instance-level — same mechanism as EC2), not NACLs directly managed per-instance.
- A typical rule: allow inbound traffic on the database's port (e.g., `3306` for MySQL, `5432` for PostgreSQL) **only from the application tier's security group**, not from `0.0.0.0/0`.
- **Public accessibility** (a separate setting from security groups) controls whether the instance gets a publicly resolvable DNS name/public IP at all — best practice for production databases is to set this to **No**, keeping the instance in private subnets reachable only from within the VPC (or via VPN/Direct Connect/bastion for admin access).

---

## 4. Connecting to RDS

### Using a SQL Client
Common client tools: **DBeaver**, **pgAdmin** (PostgreSQL), **MySQL Workbench** (MySQL/MariaDB), **SQL Server Management Studio** (SQL Server), or the engine's native CLI (`psql`, `mysql`).

### Configuring Inbound Rules in Security Groups
- Add an inbound rule on the RDS instance's security group allowing TCP traffic on the database's port from the source that needs to connect (an application server's security group, a specific office IP range, or a bastion host's security group).
- Never open the database port to `0.0.0.0/0` in production — always scope to the minimum necessary source.

### Endpoint and Port Usage
- Every RDS instance gets a stable **DNS endpoint** (e.g., `mydb.c9akciq32.us-east-1.rds.amazonaws.com`) — applications should always connect via this hostname, never a raw IP, since the underlying IP can change (e.g., after failover).
- Default ports by engine: MySQL/MariaDB `3306`, PostgreSQL `5432`, Oracle `1521`, SQL Server `1433`, Aurora matches its compatible engine's default port.
- For a **Multi-AZ** deployment, the single endpoint automatically re-points to the new primary after a failover — application code doesn't need to change anything, just reconnect.

### IAM Database Authentication (Alternative to Passwords)
- RDS (MySQL/PostgreSQL) and Aurora support **IAM database authentication**, letting you connect using a short-lived auth token generated via IAM credentials instead of a traditional database password — removes the need to manage/rotate DB passwords for application connections.

---

## 5. Basic Operations

### Creating Databases and Tables
- An RDS **instance** can host multiple **databases/schemas**, depending on the engine (e.g., MySQL supports multiple databases per instance; PostgreSQL supports multiple databases, each with its own set of schemas/tables).
- Standard SQL DDL applies once connected: `CREATE DATABASE`, `CREATE TABLE`, etc. — RDS doesn't change standard SQL syntax, only how the underlying infrastructure is managed.

### Running Queries
- Standard SQL (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) works exactly as it would against any instance of that engine — RDS does not introduce new dialects.
- For read-heavy workloads, direct read traffic to **read replicas** instead of the primary where your application can tolerate replication lag ([Section 12](#12-read-replicas)).

---

## 6. Storage and Scaling

### Storage Types

| Type | Best For | Key Characteristics |
|---|---|---|
| **General Purpose SSD (gp3)** | Most workloads (current recommended default) | Baseline 3,000 IOPS and 125 MB/s throughput included regardless of volume size; IOPS and throughput can be provisioned independently of storage size, for an additional cost. |
| **General Purpose SSD (gp2)** | Legacy default, being superseded by gp3 | IOPS scale with volume size (3 IOPS/GB, burstable) — less flexible and often less cost-effective than gp3. |
| **Provisioned IOPS SSD (io1/io2)** | I/O-intensive workloads (large OLTP, SAP, etc.) | Lets you provision a specific, consistent IOPS level independent of storage size; io2 offers higher durability and a higher max IOPS-to-GB ratio than io1. |
| **Magnetic (Standard)** | Legacy, rarely used today | Lowest cost, lowest/least consistent performance — effectively deprecated for new workloads. |

### Storage Autoscaling
- RDS can automatically increase allocated storage when free space drops below a threshold, up to a maximum you define — avoids manual intervention and "disk full" outages, at the cost of less predictable billing.

### Vertical Scaling (Instance Resizing)
- Change the **instance class** (e.g., `db.m5.large` → `db.m5.xlarge`) to get more vCPU/RAM.
- Requires a brief outage (or a failover, if Multi-AZ) unless done during a maintenance window with minimal disruption settings — plan for a short downtime window regardless.

### Horizontal Scaling: Read Replicas
- Offload read traffic to one or more **read replicas**, which are asynchronously replicated copies of the primary — see full detail in [Section 12](#12-read-replicas).
- Vertical scaling increases a single instance's power; read replicas scale **read throughput** across multiple instances — these are complementary, not alternative, strategies.

---

## 7. Monitoring and Metrics

### Amazon CloudWatch Integration
- RDS automatically publishes standard metrics to CloudWatch: `CPUUtilization`, `FreeableMemory`, `FreeStorageSpace`, `DatabaseConnections`, `ReadIOPS`/`WriteIOPS`, `ReadLatency`/`WriteLatency`, `ReplicaLag` (for read replicas).
- Set **CloudWatch Alarms** on these (e.g., alert when `FreeStorageSpace` drops below a threshold, or `CPUUtilization` stays high) to catch problems before they cause outages.

### Enhanced Monitoring
- An optional feature providing **OS-level metrics** (CPU, memory, file system, process list) gathered directly from the instance's underlying host, at a much higher resolution (down to 1 second) than the default CloudWatch metrics (which are host-hypervisor-level and lower resolution).
- Delivered via a separate CloudWatch Logs stream, incurring a small additional cost.

### Performance Insights
- A dedicated dashboard visualizing database load, broken down by **SQL query, wait event, user, or host** — the primary tool for diagnosing *why* a database is slow (e.g., "80% of load is one query waiting on a lock") rather than just *that* it's slow.
- Free tier retains 7 days of history; a paid tier extends retention up to 2 years for long-term trend analysis.
- Available for most RDS engines and Aurora.

---

## 8. Backups and Snapshots

### 1. Automated Backups

#### What Are They?
- A **built-in feature** of Amazon RDS that automatically backs up your database.
- Includes:
  - **Daily snapshots**
  - **Transaction logs** for **Point-in-Time Recovery (PITR)**

#### Key Features
- **Retention Period**: configurable from **1 to 35 days**.
- **Point-in-Time Recovery**: you can restore your DB to any specific second within the retention window.
- **Enabled by Default**: when you create an RDS instance (unless explicitly disabled).
- **Storage Location**: stored in **Amazon S3**, managed by AWS (not directly accessible).
- **Encryption**: backups are encrypted using the **KMS key** associated with your DB instance.

#### Use Cases
- Disaster recovery.
- Automatic protection against data loss.
- Compliance with short-term data retention policies.

### 2. Manual Snapshots

#### What Are They?
- **User-initiated backups** of your RDS instance.
- Persist until you manually delete them.

#### Key Features
- **No Expiry**: stored indefinitely until deleted.
- **Cross-Region Copy**: can be copied to other AWS Regions for disaster recovery.
- **Sharing**: can be shared with other AWS accounts.
- **Storage Location**: stored in **Amazon S3**, managed by AWS.
- **Encryption**: encrypted using the **KMS key** selected during snapshot creation.

#### Use Cases
- Long-term archival.
- Pre-deployment safety.
- Migration across Regions or accounts.
- Compliance with long-term retention policies.

### Comparison Table

| Feature | Automated Backups | Manual Snapshots |
|---|---|---|
| **Created By** | AWS (automatically) | User (manually) |
| **Retention Period** | 1–35 days | Until manually deleted |
| **Point-in-Time Recovery** | ✅ Yes | ❌ No (only to snapshot time) |
| **Cross-Region Copy** | ❌ No (unless exported) | ✅ Yes |
| **Sharing Across Accounts** | ❌ No | ✅ Yes |
| **Storage Location** | Amazon S3 (managed by AWS) | Amazon S3 (managed by AWS) |
| **Encryption** | KMS (automated) | KMS (user-selected) |

### Restoration Options
- **Automated Backup Restore**: restore to a specific point in time; creates a **new** DB instance.
- **Snapshot Restore**: restore to the exact time the snapshot was taken; creates a **new** DB instance.

> **Interview tip:** Both restore paths create a **brand-new** RDS instance with a new endpoint — there's no "restore in place." This is a common gotcha: applications must be reconfigured to point at the new endpoint (or a DNS alias must be updated) after a restore.

### Encrypting an Unencrypted Instance
- You **cannot enable encryption on an existing unencrypted instance directly**. The standard path: take a snapshot → copy the snapshot with encryption enabled → restore a new instance from the encrypted snapshot copy.

---

## 9. Security and Access Control

### IAM Roles and Policies
- **IAM policies** control who can manage RDS *resources* (create/delete/modify instances, take snapshots) via the RDS API — this is separate from *database-level* authentication (see IAM Database Authentication in [Section 4](#4-connecting-to-rds)).
- RDS instances can be granted an **IAM role** for specific integrations, e.g., allowing SQL Server to access S3 for native backup/restore, or allowing Oracle to integrate with S3/SNS.

### Encryption at Rest and in Transit (KMS)
- **At rest**: enabled via **AWS KMS** at instance creation — encrypts the underlying storage, automated backups, snapshots, and read replicas. Cannot be toggled on after creation (see note in [Section 8](#8-backups-and-snapshots)).
- **In transit**: enforced via **SSL/TLS** connections between the client and the database — each engine provides a certificate bundle; applications can require SSL via connection parameters (and some engines support enforcing it server-side via a parameter group setting).

### Parameter Groups and Option Groups
| Construct | Purpose |
|---|---|
| **DB Parameter Group** | Controls database **engine configuration** — e.g., `max_connections`, `work_mem`, character set defaults. Some parameters apply immediately; others require a reboot. |
| **DB Option Group** | Enables and configures **engine-specific features/add-ons** not available by default — e.g., Oracle's native backup to S3, SQL Server's Transparent Data Encryption (TDE), or MariaDB audit plugins. |

- You cannot modify the default parameter/option group directly — always create a **custom group**, modify that, and associate it with your instance(s).

### Network-Level Security
- Security groups restricting inbound DB-port traffic ([Section 3](#3-vpc-subnets-and-security-groups)).
- Keeping instances **non-publicly-accessible** in private subnets as the default posture.

---

## 10. Maintenance and Patching

### Maintenance Windows
- A configurable weekly 30-minute window during which AWS applies pending **OS and database engine patches**, and in some cases other maintenance actions.
- You choose the window (day/time) to minimize business impact — AWS will not apply maintenance outside it unless the patch is a **critical security fix** that can't wait.

### Minor vs Major Version Upgrades

| Aspect | Minor Version Upgrade | Major Version Upgrade |
|---|---|---|
| Compatibility | Fully backward compatible | May include breaking changes |
| Can be automatic | ✅ Yes, if "Auto minor version upgrade" is enabled | ❌ No, always manual/explicit |
| Downtime | Brief (applied during maintenance window) | Brief for the instance, but test thoroughly first — app-level compatibility isn't guaranteed |
| Rollback | N/A (applied in place) | No automatic rollback — restore from snapshot/backup if needed |

### Automatic Failover (Multi-AZ)
- For Multi-AZ deployments, failover to the standby happens automatically during certain maintenance operations (e.g., instance-class changes, some patching), minimizing downtime to a typical failover window (usually under a minute to a couple of minutes) — detailed further in [Section 11](#11-multi-az-deployments).

---

## 11. Multi-AZ Deployments

### How Multi-AZ Works
- RDS provisions a **synchronous standby replica** in a different Availability Zone from the primary.
- Writes to the primary are **synchronously replicated** to the standby before being acknowledged as committed — ensuring **zero data loss (RPO ≈ 0)** on failover, at the cost of slightly higher write latency than a single-AZ deployment.
- The standby is **not readable** in the classic Multi-AZ (single standby) architecture — it exists purely for failover, unlike a read replica.

### Failover Process
Failover is triggered automatically when RDS detects:
- The primary instance becomes unavailable (AZ outage, instance failure, storage failure).
- A manual "reboot with failover" action.
- Certain maintenance operations (patching, instance-class modification).

During failover:
1. RDS automatically promotes the standby to be the new primary.
2. The **DNS endpoint** is updated to point to the new primary — application connection strings don't need to change.
3. Typical failover completes in **60–120 seconds**, though this varies by engine and workload.

### Monitoring Failover Events
- RDS Events (via the console, CLI, or **Amazon EventBridge**) emit notifications when a failover occurs — set up an **SNS topic subscription** via RDS Event Notifications to alert on-call teams automatically.
- CloudWatch metric `DatabaseConnections` dropping to zero briefly, paired with an RDS event log entry, confirms a failover happened.

### Multi-AZ with Two Readable Standbys (Newer Option)
- For some engines, RDS now also offers a **Multi-AZ DB cluster** deployment (as opposed to the classic single-standby instance): one writer + **two readable standbys**, using semi-synchronous replication, offering both faster failover (often under 35 seconds) and the ability to offload some read traffic to the standbys — a middle ground between classic Multi-AZ and Aurora.

---

## 12. Read Replicas

### Use Cases (Read Scaling, Disaster Recovery)
- **Read scaling**: offload read-heavy traffic (reporting, analytics, read-heavy API endpoints) from the primary to one or more read replicas, reducing load on the primary for writes.
- **Disaster recovery**: a cross-Region read replica can be **promoted** to a standalone, writable instance if the primary Region becomes unavailable — a DR strategy, though with higher RPO than Multi-AZ (replication is asynchronous).
- **Workload isolation**: route a specific heavy reporting job to a dedicated replica so it doesn't compete with production traffic on the primary.

### How It Works
- Replication is **asynchronous** (unlike Multi-AZ's synchronous replication) — meaning there can be **replication lag**, and a replica may briefly serve slightly stale data. Monitor this via the `ReplicaLag` CloudWatch metric.
- A single primary can have **up to 5 direct read replicas** (more for Aurora — see [Section 15](#15-aurora-deep-dive)), and replicas can themselves have their own read replicas (cascading replication) for some engines.

### Cross-Region Replication
- Read replicas can be created in a **different AWS Region** from the primary, useful for:
  - Disaster recovery against a full Region outage.
  - Serving read traffic closer to geographically distributed users, reducing latency.
- Cross-Region replication incurs data transfer charges and, since it's asynchronous over a longer network path, typically sees higher replication lag than same-Region replicas.

### Promoting Replicas
- A read replica can be **promoted** to become its own independent, standalone, writable DB instance.
- Promotion **breaks the replication link permanently** — this is a one-way operation; the promoted instance no longer receives updates from (or sends them to) the original primary.
- Common scenarios: disaster recovery failover, or splitting off a replica to become an independent instance for a new application/team.

### Read Replica vs Multi-AZ Standby

| Aspect | Read Replica | Multi-AZ Standby |
|---|---|---|
| Replication | Asynchronous | Synchronous |
| Readable | ✅ Yes | ❌ No (classic Multi-AZ) / ✅ Yes (Multi-AZ DB cluster) |
| Purpose | Read scaling, DR, workload isolation | High availability / automatic failover |
| Automatic failover target | ❌ No (must be manually promoted) | ✅ Yes |
| Can span Regions | ✅ Yes | ❌ No (same Region, different AZ) |

> **Interview tip:** This is one of the most common RDS interview questions — **Multi-AZ is for availability (HA)**, **read replicas are for scalability (and optionally DR)**. They solve different problems and are often used together: a Multi-AZ primary with one or more read replicas hanging off it.

---

## 13. Performance Tuning

### Query Optimization
- Use `EXPLAIN` (or the engine's equivalent) to understand a query's execution plan before optimizing blindly.
- Avoid `SELECT *` in production code paths — fetch only needed columns to reduce I/O and network transfer.
- Batch writes where possible instead of many small individual `INSERT`/`UPDATE` statements, to reduce round-trip and transaction overhead.

### Indexing Strategies
- Index columns used in `WHERE`, `JOIN`, and `ORDER BY` clauses — but remember every index adds write overhead (slower `INSERT`/`UPDATE`/`DELETE`), so don't over-index.
- Use **composite indexes** thoughtfully, ordering columns by selectivity and query pattern (leftmost-prefix matching applies for most engines' B-tree indexes).
- Periodically review and drop **unused indexes** — Performance Insights and engine-specific tools (e.g., PostgreSQL's `pg_stat_user_indexes`) help identify them.

### Parameter Tuning
Common tunable parameters (via a custom **DB Parameter Group**, [Section 9](#9-security-and-access-control)):

| Parameter | Engine | Purpose |
|---|---|---|
| `max_connections` | MySQL/PostgreSQL | Caps the number of simultaneous client connections — too high risks memory exhaustion, too low causes connection errors under load. |
| `work_mem` | PostgreSQL | Memory allocated per query operation (sorts, hash joins) before spilling to disk. |
| `innodb_buffer_pool_size` | MySQL | Size of the InnoDB cache for data/indexes in memory — one of the most impactful MySQL tuning parameters. |
| `shared_buffers` | PostgreSQL | Memory PostgreSQL uses for caching data pages. |

### Connection Pooling
- Opening a raw DB connection is relatively expensive; use a **connection pooler** (application-level pooling, or **Amazon RDS Proxy**) to reuse connections efficiently, especially important for serverless/Lambda workloads that can otherwise exhaust `max_connections` under concurrent scale-out.

### Amazon RDS Proxy
- A fully managed, highly available database proxy that sits between your application and RDS/Aurora, pooling and sharing database connections to improve scalability (critical for Lambda, which can otherwise open one connection per concurrent invocation).
- Also improves failover resilience by maintaining connections and transparently redirecting them during a Multi-AZ failover, and supports IAM authentication for the proxy connection itself.

---

## 14. Cost Optimization

### Instance Sizing
- Right-size using CloudWatch/Performance Insights data rather than guessing — an over-provisioned instance is pure waste; an under-provisioned one risks performance incidents.
- Use **burstable (`db.t`) instance classes** for genuinely intermittent workloads (dev/test, low-traffic apps) to save cost versus always-on general-purpose classes.

### Reserved Instances vs On-Demand

| Aspect | On-Demand | Reserved Instances (RI) |
|---|---|---|
| Commitment | None | 1 or 3 years |
| Cost | Highest per-hour rate | Up to ~60–70% cheaper than On-Demand for the same instance |
| Flexibility | Full — change/stop anytime | Locked in for the term (though RIs can sometimes be modified within a Region/engine family) |
| Best for | Unpredictable or short-term workloads, dev/test | Stable, predictable, long-running production workloads |

### Storage Autoscaling and Right-Sizing
- Use **storage autoscaling** ([Section 6](#6-storage-and-scaling)) instead of over-provisioning storage up front "just in case."
- Choose **gp3 over gp2** where possible — gp3 lets you provision IOPS/throughput independently of size, often costing less for the same performance target.

### Other Cost Levers
- **Stop (not just terminate) non-production instances** outside business hours — a stopped RDS instance (non-Aurora) doesn't incur compute charges, only storage (note: RDS auto-starts a stopped instance after 7 days if not manually restarted).
- **Delete unused manual snapshots** — they're billed indefinitely until removed.
- **Aurora Serverless v2** for spiky/intermittent workloads, to avoid paying for constant peak capacity ([Section 15](#15-aurora-deep-dive)).

---

## 15. Aurora Deep Dive

### Aurora vs RDS

| Aspect | Amazon Aurora | Standard RDS (MySQL/PostgreSQL) |
|---|---|---|
| Architecture | Cloud-native, distributed storage layer decoupled from compute, auto-replicated 6 ways across 3 AZs | Traditional engine architecture, storage tied to the instance (EBS-backed) |
| Performance | Up to ~5x MySQL / ~3x PostgreSQL throughput (per AWS benchmarks) | Standard engine performance |
| Storage | Auto-scales up to 128 TB, pay only for what's used | Must provision and manage storage size yourself |
| Read replicas | Up to **15**, with sub-10ms replica lag typical (shared storage layer) | Up to 5, asynchronous, can lag more |
| Failover | Typically under 30 seconds (replicas share the same storage, so failover is just a DNS/role change) | Classic Multi-AZ: 60–120 seconds |
| Cost | Generally higher per-hour than equivalent RDS instance class | Generally lower baseline cost |
| Compatibility | MySQL- and PostgreSQL-compatible (two separate Aurora flavors) | Native engine (including non-MySQL/PG engines like Oracle, SQL Server) |

### Aurora Serverless v2
- Automatically scales compute capacity **up and down in fine-grained increments** (measured in Aurora Capacity Units, ACUs) based on actual load, within a min/max range you configure.
- Ideal for **unpredictable or spiky workloads** (e.g., dev/test environments, infrequently used applications, or apps with large traffic swings) where paying for constant peak-provisioned capacity would be wasteful.
- Unlike Aurora Serverless v1, v2 scales nearly instantaneously and supports features like read replicas and Multi-AZ, making it suitable for production use, not just dev/test.

### Aurora Global Databases
- Lets a single Aurora cluster span **multiple AWS Regions**, with one primary Region handling writes and up to 5 secondary Regions providing **low-latency local reads** (typically sub-second cross-Region replication lag, via dedicated replication infrastructure separate from normal read replica replication).
- Secondary Regions can be **promoted** to take over as the new primary within about a minute in a Regional disaster recovery scenario — a much faster RTO than reconstructing a cross-Region topology manually.

---

## 16. Disaster Recovery & DR Planning

### Cross-Region Snapshots
- Manual snapshots can be copied to another Region (see [Section 8](#8-backups-and-snapshots)) to protect against a full Regional outage, and restored into a new instance there if needed.

### Automated Failover Strategies
- **Within a Region**: Multi-AZ provides automatic failover with near-zero RPO ([Section 11](#11-multi-az-deployments)).
- **Across Regions**: options include promoting a cross-Region read replica (manual, higher RPO) or using an **Aurora Global Database** (near-automatic, low RPO/RTO) ([Section 15](#15-aurora-deep-dive)).

### Backup Retention Policies
- Balance **retention length** (compliance/business requirements) against storage cost — automated backups up to 35 days, manual snapshots indefinitely (and cross-Region copies for geographic redundancy).
- Document and periodically **test restores** — an untested backup strategy is not a real DR plan; teams should periodically practice restoring from snapshot/PITR to verify RTO assumptions.

### RPO/RTO Quick Reference

| Strategy | Typical RPO | Typical RTO |
|---|---|---|
| Multi-AZ (same Region) | ~0 (synchronous) | ~1–2 minutes |
| Aurora Global Database | Seconds | ~1 minute |
| Cross-Region read replica (manual promotion) | Minutes (async lag) | Minutes to tens of minutes (manual action) |
| Snapshot restore (same or cross-Region) | Up to the last backup/snapshot | Longer (new instance provisioning + data restore time) |

---

## 17. Infrastructure as Code

### Automating RDS with Terraform, CloudFormation, or CDK
- **Terraform**: the `aws_db_instance` (and `aws_rds_cluster` for Aurora) resources define instance class, engine, storage, subnet group, parameter/option groups, etc., in declarative HCL.
- **CloudFormation**: the `AWS::RDS::DBInstance` / `AWS::RDS::DBCluster` resource types achieve the same declaratively in YAML/JSON, integrated natively with other AWS resources in a stack.
- **AWS CDK**: provides higher-level constructs (e.g., the `aws-rds` module) that generate CloudFormation under the hood, with sensible defaults and type-safe configuration in a general-purpose language (TypeScript, Python, etc.).

### Why IaC for RDS
- Ensures **consistent, repeatable** environments across dev/test/prod instead of manual console configuration drift.
- Enables **peer review** of infrastructure changes (e.g., a parameter group change) through the same pull-request process as application code.
- Critical for safely managing sensitive settings like deletion protection, backup retention, and parameter groups across many environments.

### Version Control and CI/CD Integration
- Store IaC definitions in the same repository (or a dedicated infra repo) under version control.
- Use a CI/CD pipeline to **plan** (preview) changes before applying them to production databases — a destructive accidental change (e.g., unintentionally recreating an instance) is one of the highest-impact IaC risks for a stateful resource like a database, so plan review and safeguards (e.g., Terraform's `prevent_destroy` lifecycle rule) are especially important here.

---

## 18. Audit and Compliance

### Logging with CloudTrail
- **CloudTrail** logs **management-plane API calls** against RDS — who created/deleted/modified an instance, changed a parameter group, or took a snapshot — for audit trails and security investigations (see also the S3/IAM master notes' CloudTrail sections for the general mechanism).
- Does **not** log the actual SQL queries run inside the database — that's covered separately below.

### Database Activity Streams
- A feature (available for Aurora, Oracle, and SQL Server) that streams **database-level activity** (queries, logins, DDL/DML operations) in near real-time to Amazon Kinesis, for ingestion into a SIEM or compliance monitoring tool.
- Provides a tamper-resistant audit trail of actual data-plane activity inside the database — distinct from CloudTrail, which only covers the RDS control-plane API.

### Compliance Frameworks
- RDS supports workloads subject to **HIPAA** (with a signed AWS Business Associate Addendum and following AWS's HIPAA eligibility guidance), **PCI DSS**, **SOC 1/2/3**, and other frameworks — but compliance is a **shared responsibility**: AWS secures the underlying infrastructure, while you're responsible for secure configuration (encryption, access control, patching cadence, logging) of your specific instances.

---

## 19. Migration Strategies

### AWS Database Migration Service (DMS)
- A managed service for migrating databases into (or between) RDS/Aurora/other targets with **minimal downtime**.
- Supports both **homogeneous migrations** (e.g., MySQL → MySQL) and **heterogeneous migrations** (e.g., Oracle → PostgreSQL), the latter requiring schema conversion first.
- Can perform a **one-time full load**, or **full load + Change Data Capture (CDC)** for near-zero-downtime migrations — CDC continuously replicates ongoing changes from the source until you're ready to cut over.

### Schema Conversion
- For heterogeneous migrations (different source and target engines), use the **AWS Schema Conversion Tool (SCT)** to automatically convert schema objects, stored procedures, and functions where possible, and flag objects requiring manual rework (e.g., proprietary Oracle PL/SQL constructs with no direct PostgreSQL equivalent).

### Cutover Planning
A typical low-downtime migration cutover sequence:
1. Run DMS full load + CDC to bring the target database up to date and keep it continuously in sync with the source.
2. Monitor replication lag until it's effectively zero.
3. Schedule a brief maintenance window: stop writes to the source, let final CDC changes apply to the target, verify consistency.
4. Cut application connection strings (or DNS) over to the new target database.
5. Keep the source running read-only for a rollback window before fully decommissioning it.

---

## 20. Common Interview Questions

1. **Multi-AZ vs Read Replicas — what's the difference and when would you use each?** → [Section 12](#12-read-replicas) — HA/failover (synchronous) vs. read scaling/DR (asynchronous); often combined.
2. **How does RDS achieve point-in-time recovery?** → Automated backups + continuous transaction log shipping to S3, replayed up to any second within the retention window ([Section 8](#8-backups-and-snapshots)).
3. **What happens to the application's connection string after a Multi-AZ failover?** → Nothing — the stable DNS endpoint is automatically re-pointed to the new primary ([Sections 4, 11](#11-multi-az-deployments)).
4. **How do you encrypt an existing unencrypted RDS instance?** → You can't in place — snapshot → copy with encryption enabled → restore a new instance ([Section 8](#8-backups-and-snapshots)).
5. **How would you scale a read-heavy application without touching the primary's write capacity?** → Add read replicas and route read traffic to them ([Section 12](#12-read-replicas)).
6. **gp3 vs gp2 — why would you pick one over the other?** → gp3 decouples IOPS/throughput from volume size, usually cheaper for the same performance ([Section 6](#6-storage-and-scaling)).
7. **How do you handle a Lambda function that needs to connect to RDS at high concurrency without exhausting `max_connections`?** → Amazon RDS Proxy for connection pooling ([Section 13](#13-performance-tuning)).
8. **Aurora vs standard RDS — when would you pick Aurora?** → Need for higher throughput, faster failover, more read replicas, or storage auto-scaling beyond what you'd want to manage manually ([Section 15](#15-aurora-deep-dive)).
9. **How would you migrate from on-prem Oracle to Aurora PostgreSQL with minimal downtime?** → AWS SCT for schema conversion, then AWS DMS full load + CDC, then a brief cutover window ([Section 19](#19-migration-strategies)).
10. **What's the difference between a DB Parameter Group and an Option Group?** → Engine configuration tuning vs. enabling optional engine features/add-ons ([Section 9](#9-security-and-access-control)).
11. **How do you audit the actual SQL queries run against a compliance-sensitive database?** → Database Activity Streams (not CloudTrail, which only covers the RDS management API) ([Section 18](#18-audit-and-compliance)).
12. **How would you design a globally distributed, low-latency-read application backed by a relational database?** → Aurora Global Database with secondary Regions for local reads ([Section 15](#15-aurora-deep-dive)).
