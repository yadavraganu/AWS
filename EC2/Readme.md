## EC2 Tenancy Overview
Amazon EC2 offers three primary tenancy options that determine how your virtual machines are distributed across physical hardware: Shared (Default) Tenancy, Dedicated Instances, and Dedicated Hosts.
Choosing the right option depends on your specific compliance, licensing, and budget requirements.
| Tenancy Option | Hardware Isolation | License Visibility (Cores/Sockets) | Key Use Case | Cost |
|---|---|---|---|---|
| Shared (Default) | Multi-tenant | No | Standard cloud workloads | Lowest (Pay for usage) |
| Dedicated Instances | Single-tenant (Account level) | No | Corporate compliance & isolation | Higher (Instance fee + regional fee) |
| Dedicated Hosts | Single-tenant (Physical server level) | Yes | Bring Your Own License (BYOL) | Highest (Pay for the whole host) |
### Shared Tenancy (Default)
This is the default setting for almost all Amazon EC2 instances.
* **How it works:** Your instances run on physical hardware that is shared with other AWS accounts.
* **Isolation:** Complete programmatic and security isolation is managed by the AWS hypervisor.
* **Best for:** Cost-optimized applications, development environments, and standard production workloads.
### Dedicated Instances
This option isolates your hardware at the AWS account level.

* **How it works:** Instances run on hardware dedicated to your specific AWS account. Other instances on that same physical machine will only belong to you.
* **Caveat:** You do not have visibility or control over instance placement on the hardware. If you stop and restart the instance, it may move to a different dedicated server.
* **Best for:** Satisfying strict corporate compliance or regulatory frameworks requiring physical tenant isolation.
### Dedicated Hosts
This option provides a physical server fully dedicated to your use with advanced visibility.

* **How it works:** You lease the entire physical server. You gain full visibility into the physical sockets and cores.
* **Placement control**: You can map instances directly to specific hosts using a Host ID, ensuring they stay on the exact same hardware even after restarts.
* **Best for:** Bring Your Own License (BYOL) software models—like Microsoft Windows Server or SQL Server—that require licensing tied to physical hardware sockets or cores.
### Important Considerations

* **VPC Tenancy:** When creating an Amazon VPC, you can set its default tenancy. A "Dedicated" VPC forces all instances launched within it to use Dedicated tenancy unless specified otherwise.
* **Tenancy Switching:** You can switch an instance's tenancy between Dedicated and Dedicated Host while it is stopped. However, you cannot convert an existing instance from Dedicated/Host back to a Shared default tenancy.
Here are comprehensive reference notes and comparison tables covering all standard Amazon EC2 purchasing options, updated to reflect current AWS cloud architecture practices.

## Amazon EC2 Purchasing Options

### 1. No-Commitment Models (Pay-As-You-Go)

#### On-Demand Instances
* **How it works:** You pay for compute capacity by the second (minimum 60 seconds) with no upfront payments or long-term contracts. You can launch, stop, or terminate them at will.
* **Capacity Guarantee:** None. Availability depends on current data center supply.
* **Example:** A marketing team launches a web server on a Monday morning to test a new campaign application for a week, then terminates the instance on Friday afternoon. They are billed exactly for the hours the instance ran.


#### Spot Instances
* **How it works:** Allows you to bid on or utilize spare, unused AWS compute capacity at massive discounts. However, AWS can reclaim these instances with a strict **2-minute warning notification** if they need the capacity back for On-Demand users.
* **Capacity Guarantee:** None. This is the highest-risk model for application uptime.
* **Example:** A data engineering team runs a massive Apache Spark batch job that processes 50TB of logs. The job is fault-tolerant and automatically checkpoints its progress. By using Spot instances, they finish the calculation at a fraction of the standard cost, accepting that a few nodes might drop out and restart during execution.

### 2. Commitment-Based Savings (1-Year or 3-Year Terms)

#### Compute Savings Plans
* **How it works:** You commit to a consistent hourly spend (e.g., $15/hour) for a 1 or 3-year term.
* **Flexibility:** Maximum. It automatically applies to any EC2 instance family, size, zone, or operating system. It also seamlessly extends its discount to **AWS Fargate** and **AWS Lambda** container/serverless workloads.
* **Example:** A growing startup expects their baseline infrastructure costs to always be at least $50/hour. They commit to a Compute Savings Plan. Six months later, they migrate half their microservices from standard EC2 instances to AWS Fargate containers in a different AWS region. The Savings Plan automatically shifts the discount to cover the new Fargate usage.


#### EC2 Instance Savings Plans
* **How it works:** You commit to a consistent hourly spend on a specific **instance family** within a specific **AWS region** (e.g., `m6g` instances in `us-east-1`).
* **Flexibility:** Moderate. You can change sizes, OS types, or tenancies within that specific family, but you cannot transfer the discount to a different family or region.
* **Example:** A financial institution runs a core database cluster on `r6i` memory-optimized instances in London (`eu-west-2`). They know this architecture won't change for years. They lock in an EC2 Instance Savings Plan to maximize their discount percentage for that specific family.


#### Reserved Instances (RIs) — Legacy Model
* *Note: Standard and Convertible RIs are older mechanisms largely replaced by Savings Plans, but they are still fully supported and common on AWS certification exams.*
* **Standard RIs:** Offers the highest discount (up to 72%) but requires locking into a specific instance attribute profile. Can provide an actual capacity reservation if scoped to a specific Availability Zone (AZ).
* **Convertible RIs:** Offers a lower discount (up to 54%) but allows you to exchange the reservation for a different instance family, OS, or tenancy later on, provided the new reservation is of equal or greater value.

### 3. Capacity Reservations & Dedicated Physical Hardware

#### On-Demand Capacity Reservations (ODCR)
* **How it works:** You reserve specific compute capacity in a chosen Availability Zone. Unlike Savings Plans, this is **not a discount tool**—you are charged the standard On-Demand rate regardless of whether your instances are running or sitting idle.
* **Example:** An e-commerce platform prepares for a massive Black Friday sales event. To ensure that flash-sale microservices can scale out instantly without hitting data center capacity limits, they create a Capacity Reservation a few days in advance. They pay for the idle capacity until the event begins and instances spin up into those reserved slots.


#### EC2 Capacity Blocks for ML
* **How it works:** Allows you to reserve highly scarce, sought-after GPU instances (like NVIDIA H100s) for a targeted, future time window (ranging from 1 to 14 days).
* **Example:** An AI research team schedules an upcoming 3-day sprint to fine-tune a large language model. Knowing these specific GPU chips face high global demand, they book an EC2 Capacity Block two weeks ahead of time to guarantee the cluster is available the exact hour their sprint starts.


#### Dedicated Hosts
* **How it works:** A physical, bare-metal server fully dedicated to your single-tenant use.
* **Example:** A corporation has legacy corporate software licensed explicitly by the number of physical CPU sockets or cores on a machine. They lease an EC2 Dedicated Host to map their existing licenses directly to the physical hardware, maintaining strict compliance while moving to the cloud.

### Side-by-Side Comparison Tables

#### Table 1: Cost vs. Risk Flexibility (Standard Models)

| Dimension | On-Demand | Spot Instances | Compute Savings Plans | EC2 Instance Savings Plans |
| --- | --- | --- | --- | --- |
| **Max Discount** | 0% (Baseline) | **Up to 90%** | Up to 66% | **Up to 72%** |
| **Commitment Term** | None (Hourly/Per-second) | None | 1 or 3 Years | 1 or 3 Years |
| **Can AWS Terminate It?** | No | **Yes (2-minute warning)** | No | No |
| **Region Flexibility** | High | High | **Fully Flexible** | Locked to single region |
| **Family Flexibility** | High | High | **Fully Flexible** | Locked to instance family |
| **Extends to Fargate/Lambda?** | No | No | **Yes** | No |

#### Table 2: Dedicated Hardware & Capacity Assurances

| Dimension | Capacity Reservations (ODCR) | Capacity Blocks for ML | Dedicated Hosts | Dedicated Instances |
| --- | --- | --- | --- | --- |
| **Primary Goal** | Assured AZ capacity | Securing rare GPU hardware | Software Licensing (BYOL) | Hardware Isolation |
| **Cost Profile** | Standard On-Demand Rate | Premium block rate | Flat rate per physical host | On-Demand rate + fee |
| **Capacity Guaranteed?** | **Yes** (In chosen AZ) | **Yes** (For specified dates) | **Yes** (You own the blade) | Yes (While running) |
| **Hardware Tenancy** | Shared or Dedicated | Shared (Dedicated Cluster) | **Physical Single-Tenant** | Host-level Isolation |
| **Term Commitment** | None (Cancel anytime) | Fixed future window (1-14 days) | None, 1 Year, or 3 Years | Hourly |

Here is a complete, production-grade set of technical notes for AWS EC2 Placement Groups. This version corrects the multi-AZ nuances, adds strict AWS constraints, and includes best practices to avoid common deployment failures.

## AWS EC2 Placement Groups

### 1. Cluster Placement Group

* **Core Concept:** Packs instances close together inside a single network segment to achieve ultra-low latency and high network throughput ($10\text{ Gbps}$ to $100\text{ Gbps}+$ depending on instance type).
* **Scope:** Restricted to a **single Availability Zone (AZ)** within the same VPC. It cannot span multiple AZs.
* **Best Used For:** * Tightly-coupled High-Performance Computing (HPC) applications.
* Big data clusters requiring low-latency node communication (e.g., specialized MPI jobs).
* High-throughput database replications.

* **Downside/Risks:** * **Single Point of Failure:** Shares underlying hardware racks; a rack failure impacts multiple instances.
* **Capacity Restrictions:** Higher likelihood of encountering `InsufficientInstanceCapacity` errors during launch or restarts if the specific network segment runs out of hardware.

### 2. Spread Placement Group

* **Core Concept:** Maximizes physical isolation by placing each individual instance on a distinct, separate hardware rack (each with its own independent power and network source).
* **Scope:** **Can span multiple Availability Zones** within the same Region.
* **Hard Constraints:** Limited to a maximum of **7 running instances per Availability Zone** per placement group.
* **Best Used For:** * Critical, small-scale workloads where instances must not fail simultaneously.
* Primary and secondary database instances.
* Core application nodes handling high-availability routing.
* **Downside:** Highly limited in scale; attempting to launch an 8th instance in the same AZ within that group will fail.

### 3. Partition Placement Group

* **Core Concept:** Divides the placement group into logical segments called "partitions." AWS ensures that no two partitions share the same hardware racks, power, or network sources.
* **Scope:** **Can span multiple Availability Zones** within the same Region.
* **Hard Constraints:** Limited to a maximum of **7 partitions per Availability Zone**. There is no hard limit on the number of instances *inside* a partition (subject only to your account limits).
* **Best Used For:** Large, distributed, and replicated data workloads that need to be topology-aware:
* **Apache Kafka:** Splitting brokers across different partitions.
* **Hadoop / HDFS:** Ensuring NameNodes and DataNodes don't share underlying hardware.
* **Cassandra / NoSQL:** Mapping database rings across separate infrastructure partitions.


* **Downside:** Instances are isolated at the partition level, not the individual instance level. A failure on a specific rack impacts only the instances assigned to that single partition.

### Key Rules & Constraints

* **Cost:** Creation and usage are completely **free**; you only pay standard rates for the EC2 instances launched inside them.
* **Naming:** Names must be unique within your AWS account for the specific Region.
* **Instance Type Restrictions:** * Not all instance types support placement groups. Burstable performance instances (e.g., `t2`, `t3`) are **explicitly blocked** from Cluster placement groups.
* Mixing different instance types in a Cluster group is strongly discouraged. It increases the probability of launch failures due to capacity fragmentation.
* **Immutability:** You **cannot merge** placement groups. However, you can move an existing instance into a placement group, or move it out, provided the instance is in a `stopped` state.
* **The "All-at-Once" Rule:** When launching instances into a Cluster placement group, **launch them all in a single CLI/API request** or CloudFormation stack. If you try to launch them incrementally over time, AWS may run out of capacity on that specific network segment, leading to failures.
* **Handling Capacity Blocks:** If you stop an instance inside a Cluster group and later try to restart it, it might fail if the host rack has filled up in the interim. You will either have to wait for capacity to free up or move the instance out of the group.
