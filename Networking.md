## Table of Contents

1. [What is Amazon VPC?](#1-what-is-amazon-vpc)
2. [CIDR Blocks](#2-cidr-blocks)
3. [Subnets](#3-subnets)
4. [Internet Gateway (IGW)](#4-internet-gateway-igw)
5. [NAT Gateway](#5-nat-gateway)
6. [Route Tables](#6-route-tables)
7. [Security Groups](#7-security-groups)
8. [Network ACLs (NACLs)](#8-network-acls-nacls)
9. [Security Groups vs NACLs](#9-security-groups-vs-nacls)
10. [VPC Peering](#10-vpc-peering)
11. [AWS Transit Gateway](#11-aws-transit-gateway)
12. [VPC Peering vs Transit Gateway vs PrivateLink](#12-vpc-peering-vs-transit-gateway-vs-privatelink)
13. [VPC Endpoints](#13-vpc-endpoints)
14. [Interface Endpoints (AWS PrivateLink)](#14-interface-endpoints-aws-privatelink)
15. [Gateway Endpoints](#15-gateway-endpoints)
16. [Elastic IPs](#16-elastic-ips)
17. [Elastic Network Interfaces (ENIs)](#17-elastic-network-interfaces-enis)
18. [Bastion Hosts](#18-bastion-hosts)
19. [VPC Flow Logs](#19-vpc-flow-logs)
20. [DNS in a VPC](#20-dns-in-a-vpc)
21. [IPv6 in a VPC](#21-ipv6-in-a-vpc)
22. [Egress-Only Internet Gateway](#22-egress-only-internet-gateway)
23. [Site-to-Site VPN vs AWS Direct Connect](#23-site-to-site-vpn-vs-aws-direct-connect)
24. [VPC Sharing (RAM)](#24-vpc-sharing-ram)
25. [Multi-AZ & High-Availability Design](#25-multi-az--high-availability-design)
26. [Common Interview Questions](#26-common-interview-questions)

---

## 1. What is Amazon VPC?

Amazon VPC is a **logically isolated network** within the AWS cloud where you can launch AWS resources (like EC2 instances). It gives you full control over your virtual networking environment, including IP address ranges, subnets, route tables, gateways, and security settings.

- Every AWS account gets a **Default VPC** per Region (with a public subnet in each AZ, an IGW, and permissive default settings) — most production workloads instead use a **custom VPC** designed deliberately.
- A VPC is **Regional** — it spans all Availability Zones in that Region, but a **subnet** lives in exactly one AZ.
- Up to **5 VPCs per Region** by default (soft limit, increasable via a quota request).

### Core Concepts of VPC

| Component | Purpose |
|---|---|
| **CIDR Block** | Defines the VPC's IP address range |
| **Subnets** | Divide the VPC into smaller networks, each tied to one AZ |
| **Internet Gateway (IGW)** | Enables internet access for public subnets |
| **NAT Gateway** | Lets private subnets reach the internet outbound only |
| **Route Tables** | Control traffic routing within/out of the VPC |
| **Security Groups** | Stateful, instance-level virtual firewall |
| **Network ACLs** | Stateless, subnet-level firewall |
| **VPC Peering** | Private connection between two VPCs |
| **AWS Transit Gateway** | Hub connecting many VPCs and on-prem networks |
| **VPC Endpoints** | Private connectivity to AWS services without the internet |

---

## 2. CIDR Blocks

A **CIDR block** (Classless Inter-Domain Routing) defines the range of IP addresses available to your VPC and its subnets.

- Example: `10.0.0.0/16` gives you 65,536 IP addresses (the whole second, third, and fourth octets are usable).
- Allowed VPC CIDR size: **/16 (65,536 IPs) to /28 (16 IPs)**.
- You can add **secondary CIDR blocks** to a VPC later if you run out of address space (useful when scaling beyond the original plan).
- Best practice: use **private IP ranges** per RFC 1918 — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

### Reserved IPs per Subnet
AWS reserves **5 IP addresses** in every subnet, which reduces usable IPs:

| IP | Reserved For |
|---|---|
| `.0` | Network address |
| `.1` | VPC router |
| `.2` | DNS server (Amazon-provided DNS) |
| `.3` | Reserved for future use |
| `.255` (last address) | Network broadcast address (not supported in VPC, but still reserved) |

> **Interview tip:** A `/24` subnet has 256 total addresses but only **251 usable** — this "5 reserved IPs" gotcha is a very common question.

### Planning CIDR Ranges
- Plan **non-overlapping** CIDR ranges across VPCs from the start — overlapping ranges block VPC Peering and complicate Transit Gateway routing later.
- Leave room to grow: don't carve every subnet as tightly as possible if you plan to add more AZs, subnets, or services later.

---

## 3. Subnets

A **subnet** is a logical subdivision of a VPC — a range of IP addresses within the VPC's larger CIDR block. Subnets organize and isolate resources for security and management. **Each subnet must reside entirely within a single Availability Zone (AZ)** — a key design consideration for high availability.

### Types of Subnets

#### Public Subnet
A subnet whose route table has a direct route to an **Internet Gateway (IGW)**.

- Instances can have public IP addresses (auto-assigned or via Elastic IP).
- Used for public-facing resources: web servers, public load balancers, bastion hosts, NAT Gateways.
- Traffic destined for the internet routes through the IGW.

#### Private Subnet
A subnet whose route table does **not** have a direct route to an IGW. Resources cannot be directly accessed from the internet.

- Instances only have private IP addresses.
- Used for backend services, databases, and application servers.
- Outbound internet access (e.g., for updates) goes indirectly through a **NAT Gateway** in a public subnet.

### Subnet Design Best Practices
- Spread subnets across **multiple AZs** (typically 2–3) for high availability.
- Use a consistent tiering pattern per AZ: public subnet (load balancers/NAT), private-app subnet (app servers), private-data subnet (databases) — a common "3-tier" layout.
- Size subnets generously up front (e.g., `/24` per tier per AZ) — resizing a subnet after resources are launched is disruptive.

---

## 4. Internet Gateway (IGW)

An **Internet Gateway** is a horizontally scaled, redundant, and highly available VPC component that enables communication between instances in your VPC and the public internet.

### Key Points
- **One IGW per VPC** — it must be created and then attached to the VPC.
- Performs **NAT translation** for instances with public IPs (maps the private IP to the assigned public/Elastic IP for internet-bound traffic).
- Has no bandwidth constraints or availability risk to manage — it's a managed, highly available AWS resource.
- A subnet becomes "public" only when its **route table** has a route sending `0.0.0.0/0` (or the relevant destination) to the IGW — attaching an IGW to the VPC alone does not make any subnet public.

---

## 5. NAT Gateway

A **NAT Gateway** allows instances in a **private subnet** to initiate outbound connections to the internet (e.g., for software updates or calling external APIs) **without** exposing them to unsolicited inbound connections from the internet.

### How It Works
1. Deployed in a **public subnet** (it needs its own route to the IGW) and assigned an **Elastic IP**.
2. The private subnet's route table sends `0.0.0.0/0` traffic to the NAT Gateway.
3. The NAT Gateway performs **source NAT (SNAT)** — rewriting the private instance's source IP to its own Elastic IP for outbound traffic, and reversing that mapping for the return traffic.
4. Unsolicited inbound connections initiated **from** the internet are still blocked — only replies to requests the private instance made get through.

### Key Characteristics
- **Managed by AWS**: highly available within a single AZ, auto-scales bandwidth (up to 100 Gbps).
- **AZ-scoped**: a NAT Gateway lives in one AZ. For high availability across AZs, deploy **one NAT Gateway per AZ** with each AZ's private subnets routing to their own AZ's NAT Gateway (avoids cross-AZ data transfer charges and the single-AZ failure risk).
- **Billed** per hour it's provisioned, plus a per-GB data processing charge — this is a common real-world cost driver worth flagging in interviews.

### NAT Gateway vs NAT Instance

| Aspect | NAT Gateway | NAT Instance (legacy, EC2-based) |
|---|---|---|
| Management | Fully managed by AWS | Self-managed EC2 instance |
| Availability | Highly available within its AZ | Single point of failure unless scripted for failover |
| Bandwidth | Scales automatically up to 100 Gbps | Limited by instance type |
| Cost | Hourly + per-GB charge | EC2 instance cost (can be cheaper at low/steady traffic) |
| Security Group | Not directly attachable | Can attach security groups, run custom software (e.g., proxy/inspection) |
| Recommended | ✅ Yes, for almost all cases | ❌ Legacy, rarely recommended today |

---

## 6. Route Tables

A **Route Table** contains a set of rules ("routes") that determine where network traffic from a subnet is directed. Every subnet must be associated with exactly one route table (either an explicit one, or the VPC's **main route table** by default).

### How It Works
- Each route specifies a **destination CIDR** and a **target** (e.g., `local`, an IGW, a NAT Gateway, a VPC peering connection, a Transit Gateway, or a VPC endpoint).
- Every route table automatically includes a `local` route for the VPC's own CIDR block, enabling communication between all resources inside the VPC — this route cannot be removed.
- The **most specific matching route** wins when multiple routes could apply (longest prefix match).

### Example Route Table (Public Subnet)

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | `igw-xxxxxxxx` |

### Example Route Table (Private Subnet)

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | `nat-xxxxxxxx` |

### Key Points
- A subnet with a route table pointing `0.0.0.0/0` to an IGW is a **public subnet**; pointing it to a NAT Gateway (or having no default route at all) makes it **private**.
- One route table can be associated with **multiple subnets**, but a subnet has only **one** route table at a time.
- The **main route table** is the default for any subnet not explicitly associated with another — best practice is to leave the main route table without an internet route and explicitly associate public subnets with a custom route table that does.

---

## 7. Security Groups

**Security Groups** are virtual firewalls that control **inbound and outbound traffic** to AWS resources — primarily EC2 instances. They operate at the **instance level** (technically, the ENI level), not the subnet level.

### Key Characteristics
- **Stateful**: if you allow inbound traffic, the response is automatically allowed outbound (and vice versa) — you don't need a matching outbound rule for return traffic.
- **Instance-level**: applied directly to EC2 instances or other supported resources (RDS, ELB, Lambda ENIs, etc.).
- **Allow rules only**: you can only specify what traffic is *allowed*; everything else is implicitly denied. There is no explicit "deny" rule in a security group.
- **Multiple groups**: you can assign multiple security groups to a single instance — the **union** of all their rules applies.
- Security groups can reference **other security groups** as the source/destination (not just CIDR ranges) — very useful for tiered architectures (e.g., "allow port 3306 from the app-tier security group").

### Inbound Rules
Define what traffic is allowed to **reach** your instance.
- Allow SSH from your IP: `TCP port 22` from `203.0.113.0/32`
- Allow HTTP from anywhere: `TCP port 80` from `0.0.0.0/0`

### Outbound Rules
Define what traffic your instance is allowed to **send out**.
- Allow all outbound traffic: `0.0.0.0/0` on all ports (the default for a new security group).

### Where to Use Security Groups

| Resource Type | Use Case Example |
|---|---|
| **EC2 Instances** | Control access via SSH, HTTP, HTTPS, etc. |
| **RDS Databases** | Allow access only from specific EC2 security groups or IP ranges |
| **Elastic Load Balancers** | Define allowed traffic to backend instances |
| **Lambda (VPC-enabled)** | Control access to other VPC resources |
| **ECS Tasks (Fargate)** | Secure communication between containers |

---

## 8. Network ACLs (NACLs)

**Network ACLs** are **stateless** firewalls that operate at the **subnet level**, evaluating traffic entering or leaving every resource in the subnets they're associated with.

### Key Characteristics
- **Stateless**: return traffic is **not** automatically allowed — you must explicitly define both inbound **and** outbound rules for a full round-trip (e.g., allow inbound on port 80, and allow outbound on ephemeral ports 1024–65535 for the responses).
- **Subnet-level**: applies to every instance in the associated subnet(s), not per-instance.
- **Numbered rules, evaluated in order**: rules are processed by rule number (lowest first); the first rule that matches the traffic is applied, and evaluation stops there.
- **Supports both Allow and Deny rules** — unlike security groups, NACLs can explicitly deny specific traffic (e.g., block a known malicious IP range).
- Every VPC comes with a **default NACL** that allows all inbound and outbound traffic; custom NACLs **deny all** traffic by default until you add rules.
- A subnet is associated with exactly **one** NACL at a time, but one NACL can be associated with **multiple subnets**.

### Example NACL Rules

| Rule # | Type | Protocol | Port Range | Source/Dest | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 200 | Inbound | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW |
| * | Inbound | All | All | `0.0.0.0/0` | DENY (default catch-all) |

---

## 9. Security Groups vs NACLs

| Aspect | Security Groups | Network ACLs |
|---|---|---|
| Level | Instance (ENI) level | Subnet level |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions explicitly) |
| Rule types | Allow only | Allow **and** Deny |
| Evaluation | All rules evaluated; if any rule matches, traffic is allowed | Rules evaluated **in order** by rule number; first match wins |
| Scope | Applies only to the resources it's attached to | Applies to **all** resources in the associated subnet(s) |
| Default behavior | Deny all inbound, allow all outbound (new SG) | Default NACL allows all; custom NACL denies all until rules are added |
| Typical use | Primary, fine-grained control (per-app/tier) | Secondary layer — subnet-wide blocklists, defense in depth |

> **Interview tip:** Use **security groups as the primary control** (they map naturally to application tiers) and **NACLs as a coarse, secondary layer** — e.g., to explicitly block a known-bad CIDR range at the subnet boundary regardless of any security group.

---

## 10. VPC Peering

**VPC Peering** is a networking connection between two VPCs that enables routing traffic between them using **private IP addresses**. Useful for communication between VPCs in the same or different AWS accounts and Regions.

### When to Use VPC Peering
Use it when:
1. You need **private communication** between two VPCs.
2. VPCs are in the **same Region or different Regions**.
3. You want **low-latency, high-bandwidth** communication.
4. You don't need **transitive routing** (VPC A → VPC B → VPC C is **not** supported over peering).
5. You want to avoid NAT gateways or VPNs for internal traffic.

### How to Set Up VPC Peering
1. **Create a Peering Connection** — VPC Console → Peering Connections → Create Peering Connection; choose requester and accepter VPCs.
2. **Accept the Peering Request** — the owner of the accepter VPC must accept it.
3. **Update Route Tables** — add routes in both VPCs pointing the peer's CIDR to the peering connection.
4. **Update Security Groups** — allow inbound traffic from the peer VPC's CIDR block (or reference its security groups if in the same account/region).

### When to Avoid VPC Peering
Avoid it if:
1. You need **transitive routing** between multiple VPCs → use **AWS Transit Gateway** instead.
2. You have **many VPCs** to connect (hub-and-spoke) → peering becomes an unmanageable full-mesh at scale (N VPCs need N(N-1)/2 peering connections).
3. You need **centralized egress/ingress** → Transit Gateway or a centralized NAT/firewall VPC is better suited.
4. You want **fine-grained service-level access** (only specific services, not the whole network) → use **AWS PrivateLink** instead.

### Why Overlapping IP Ranges Are Not Allowed
- VPC Peering uses route tables to forward traffic between VPCs.
- If both VPCs have overlapping IP ranges (e.g., both `10.0.0.0/16`), AWS cannot determine which VPC a packet is meant for.
- This causes routing conflicts and unpredictable behavior — peering connections between VPCs with overlapping CIDRs simply cannot be created.

### Best Practices
- Plan VPC CIDR blocks carefully to avoid overlap.
- Use non-overlapping private IP ranges, e.g.:
  - VPC A: `10.0.0.0/16`
  - VPC B: `10.1.0.0/16`
- Consider smaller subnets if IP space is limited.

---

## 11. AWS Transit Gateway

**AWS Transit Gateway (TGW)** is a highly scalable, managed hub that connects multiple VPCs and on-premises networks through a single gateway, using a **hub-and-spoke** model instead of a full mesh.

### Why Use It
- Solves the peering **scalability problem**: instead of N(N-1)/2 peering connections for N VPCs, each VPC attaches once to the TGW.
- Supports **transitive routing** — VPC A can reach VPC C through the TGW even without a direct connection, unlike VPC Peering.
- Can also attach **VPNs** and **Direct Connect** gateways, centralizing all hybrid and inter-VPC connectivity in one place.

### Key Concepts
- **Attachments**: VPCs, VPNs, Direct Connect gateways, and peering connections to other Transit Gateways (including cross-Region) all attach to the TGW.
- **Transit Gateway Route Tables**: separate from VPC route tables; control which attachments can route to which other attachments — used to segment traffic (e.g., isolate a "shared services" VPC from being reachable by a "sandbox" VPC).
- **Cross-Region peering**: Transit Gateways in different Regions can be peered together to extend the hub-and-spoke model globally.

### VPC Peering vs Transit Gateway

| Aspect | VPC Peering | Transit Gateway |
|---|---|---|
| Topology | Point-to-point (mesh at scale) | Hub-and-spoke |
| Transitive routing | ❌ Not supported | ✅ Supported |
| Scalability | Poor beyond a handful of VPCs | Scales to thousands of attachments |
| Cost | No hourly charge, only data transfer | Hourly charge per attachment + data processing |
| On-prem connectivity | Not directly (needs separate VPN/DX per VPC) | Centralizes VPN/Direct Connect for all VPCs |
| Best for | A few VPCs needing direct, simple connectivity | Many VPCs, hybrid networks, centralized routing/segmentation |

---

## 12. VPC Peering vs Transit Gateway vs PrivateLink

A common interview question is choosing the right connectivity option:

| Requirement | Best Fit |
|---|---|
| Connect 2–3 VPCs directly, no transitive routing needed | **VPC Peering** |
| Connect dozens/hundreds of VPCs, need transitive routing, centralize VPN/DX | **Transit Gateway** |
| Expose **one specific service** (not the whole network) to many consumer VPCs/accounts, including third parties | **AWS PrivateLink (Interface Endpoints)** |
| Need full network-level reachability (all ports/protocols) between environments | Peering or Transit Gateway |
| Need to avoid exposing your VPC's internal topology to the consumer at all | **PrivateLink** — consumers only see the service endpoint, never your VPC's CIDR or resources |

---

## 13. VPC Endpoints

**VPC Endpoints** provide **private connectivity** between your VPC and supported AWS services (or other VPCs/services) **without traversing the public internet**, and without needing an IGW, NAT Gateway, or public IP.

### Types

| Type | Powered By | Supported Services | Cost |
|---|---|---|---|
| **Gateway Endpoint** | Route table entries | Amazon S3, DynamoDB only | Free |
| **Interface Endpoint** | AWS PrivateLink (ENI with private IP) | Most other AWS services, third-party/custom services via NLB | Hourly + per-GB data processing charge |

See [Section 14](#14-interface-endpoints-aws-privatelink) and [Section 15](#15-gateway-endpoints) for details on each.

---

## 14. Interface Endpoints (AWS PrivateLink)

An **Interface Endpoint** is a type of VPC Endpoint that lets you privately connect your VPC to supported AWS services, third-party services, or your own services — **without using public IPs** or traversing the internet.

### Key Features
- **Powered by AWS PrivateLink**: uses private IPs within your VPC.
- **Creates an Elastic Network Interface (ENI)** in your subnet, with a private IP address.
- Traffic stays entirely within the AWS network.
- Supports services like: Amazon S3, DynamoDB (via interface endpoint as an alternative to the gateway endpoint), Secrets Manager, CloudWatch, KMS, and custom services hosted behind a Network Load Balancer.

### How It Works
1. Create an **Interface Endpoint** in a subnet of your VPC.
2. AWS provisions an **ENI** with a private IP in that subnet.
3. Your VPC resources use this ENI (via its private DNS name) to communicate with the target service.
4. No need for a NAT Gateway, Internet Gateway, or public IPs.

### Use Cases

| Use Case | Benefit |
|---|---|
| Accessing AWS services privately | Avoids public internet exposure |
| Connecting to third-party SaaS | Secure, scalable integration via PrivateLink |
| Hosting internal services | Share securely across accounts/VPCs |
| Compliance-sensitive workloads | Meets data residency and security requirements |

### Security Benefits
- Traffic never leaves the AWS backbone.
- Access controlled with **Security Groups** (interface endpoints support them, unlike gateway endpoints) and **IAM/endpoint policies**.
- Helps meet compliance and audit requirements.

---

## 15. Gateway Endpoints

A **Gateway Endpoint** is the other type of VPC Endpoint, supporting only **Amazon S3** and **DynamoDB**.

### How It Works
- Implemented as a **route table target** (not an ENI) — you add a route in your subnet's route table pointing traffic for S3/DynamoDB to the gateway endpoint.
- No hourly or data processing charges — it's **free**.
- Cannot be extended with a **Security Group** (since it's not an ENI) — access is controlled via an **endpoint policy** and the target service's own resource policies (e.g., an S3 bucket policy scoped with `aws:SourceVpce`).

### Gateway vs Interface Endpoint

| Aspect | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| Mechanism | Route table entry | ENI with private IP (PrivateLink) |
| Supported services | S3, DynamoDB only | Most AWS services + custom/third-party via NLB |
| Cost | Free | Hourly + per-GB charge |
| Security Group support | ❌ No | ✅ Yes |
| Reachable from on-premises (via DX/VPN) | ❌ No | ✅ Yes |
| Access control | Endpoint policy + resource policy | Endpoint policy + Security Groups |

---

## 16. Elastic IPs

An **Elastic IP (EIP)** is a static, public IPv4 address that you allocate to your AWS account and can associate with an EC2 instance, NAT Gateway, or network interface.

### Key Points
- Unlike an auto-assigned public IP (which changes if the instance stops/starts), an EIP **stays the same** until you explicitly release it — useful for DNS records or firewall allowlists that need a stable IP.
- You can **remap** an EIP from one instance to another quickly (e.g., for failover).
- **AWS charges for EIPs that are allocated but not associated with a running resource**, or associated with a stopped instance — to discourage hoarding scarce IPv4 addresses.
- A NAT Gateway requires an EIP to provide a stable outbound address for the private subnets behind it.

---

## 17. Elastic Network Interfaces (ENIs)

An **Elastic Network Interface (ENI)** is a virtual network card that can be attached to an EC2 instance, representing a network endpoint with its own private IP(s), MAC address, and security groups.

### Key Points
- An instance can have **multiple ENIs**, each potentially in a different subnet (within the same AZ) — useful for dual-homed instances (e.g., a management network + a data network).
- ENIs can be **detached from one instance and attached to another** in the same AZ, preserving the IP — useful for fast failover architectures.
- Interface Endpoints, Lambda VPC networking, and ECS/EKS networking are all implemented under the hood using ENIs.

---

## 18. Bastion Hosts

A **Bastion Host** (jump box) is a hardened EC2 instance placed in a **public subnet** that provides a single, controlled entry point for administrators to reach instances in **private subnets** via SSH/RDP, without exposing those private instances directly to the internet.

### How It Works
1. The bastion host sits in a public subnet with a security group that only allows SSH/RDP from known administrator IP ranges.
2. Administrators SSH/RDP into the bastion first.
3. From the bastion, they hop to private instances, whose security groups allow SSH/RDP **only from the bastion's security group**.

### Modern Alternative: AWS Systems Manager Session Manager
- **Session Manager** (part of AWS Systems Manager) lets you connect to instances (public or private) via the AWS Console/CLI **without opening any inbound ports at all**, without a bastion host, and without managing SSH keys — access is controlled entirely through IAM.
- Increasingly preferred over bastion hosts because it removes an entire class of attack surface (no open SSH port anywhere) and centralizes audit logging via CloudTrail.

---

## 19. VPC Flow Logs

**VPC Flow Logs** capture information about the **IP traffic** going to and from network interfaces in your VPC — used for troubleshooting connectivity issues and for security monitoring.

### What's Captured
Each flow log record includes: source/destination IP and port, protocol, number of packets/bytes, start/end time, and whether the traffic was **ACCEPT**ed or **REJECT**ed (e.g., by a security group or NACL).

### Levels You Can Enable Flow Logs At
- **VPC level** — captures all ENIs in the VPC.
- **Subnet level** — captures all ENIs in the subnet.
- **ENI level** — captures a single network interface.

### Destinations
- **Amazon CloudWatch Logs** — for real-time monitoring/alerting.
- **Amazon S3** — for long-term storage and analysis (e.g., queried with Athena).
- **Amazon Kinesis Data Firehose** — for streaming to other analytics/SIEM tools.

### Use Cases
- Diagnosing **why traffic is being blocked** (e.g., a REJECT record reveals whether a security group or NACL is the culprit — flow logs don't say which specifically, but combined with config review they narrow it down fast).
- Detecting unusual or malicious traffic patterns (e.g., port scanning, data exfiltration).
- Feeding network traffic data into SIEM/security analytics pipelines.

> **Interview tip:** Flow Logs record **metadata about connections**, not the packet payload/contents — they won't show you what data was sent, only that a connection was accepted or rejected.

---

## 20. DNS in a VPC

Amazon VPC provides DNS resolution and hostname assignment for resources inside it, controlled by two VPC attributes:

| Attribute | Purpose |
|---|---|
| `enableDnsSupport` | Whether the VPC's Amazon-provided DNS server (at the `.2` address in each subnet) resolves domain names at all. |
| `enableDnsHostnames` | Whether instances receive public/private **DNS hostnames** in addition to IP addresses. |

### Key Points
- The **Amazon-provided DNS Resolver** lives at the base of the VPC's CIDR range plus two (e.g., `10.0.0.2` for a `10.0.0.0/16` VPC), or more generally at the reserved `.2` address of each subnet.
- Private hosted zones in **Route 53** let you create custom internal DNS names resolvable only within one or more associated VPCs — commonly used for service discovery in private subnets.
- **Interface Endpoints** rely on VPC DNS (specifically `enableDnsHostnames`/private DNS on the endpoint) so that the standard AWS service hostname (e.g., `s3.amazonaws.com`) transparently resolves to the endpoint's private IP instead of the public one.

---

## 21. IPv6 in a VPC

VPCs can optionally support **IPv6** alongside IPv4 (dual-stack), or in some cases IPv6-only subnets.

### Key Points
- AWS assigns a **/56 IPv6 CIDR block** to the VPC (from Amazon's pool, or you can bring your own), from which you carve **/64 subnets**.
- IPv6 addresses in AWS are **all globally unique/public by design** — there's no equivalent of private RFC 1918 space for IPv6.
- Because every IPv6 address is inherently reachable in principle, an **Egress-Only Internet Gateway** (see next section) is used to allow outbound-only IPv6 traffic, mirroring what a NAT Gateway does for IPv4.
- Security groups and NACLs still apply to IPv6 traffic exactly as they do for IPv4 — having a globally routable address does **not** bypass firewall rules.

---

## 22. Egress-Only Internet Gateway

An **Egress-Only Internet Gateway (EIGW)** is the IPv6 equivalent of a NAT Gateway: it allows instances in a VPC to initiate **outbound-only** IPv6 traffic to the internet while preventing the internet from initiating inbound IPv6 connections to those instances.

### Key Points
- Used **only for IPv6** traffic — NAT Gateways handle IPv4 outbound-only traffic instead.
- Stateful in the sense that it tracks outbound connections and allows their return traffic, but blocks unsolicited inbound connections.
- Since IPv6 addresses have no concept of "private" like IPv4 (see [Section 21](#21-ipv6-in-a-vpc)), an EIGW is the mechanism that gives IPv6 resources the same outbound-only behavior a NAT Gateway gives IPv4 resources in a private subnet.

---

## 23. Site-to-Site VPN vs AWS Direct Connect

Both connect an on-premises network to a VPC, but with different trade-offs.

| Aspect | Site-to-Site VPN | AWS Direct Connect |
|---|---|---|
| Connection type | Encrypted tunnel over the **public internet** | Dedicated **private physical** network connection |
| Setup time | Minutes to hours | Weeks to months (physical cross-connect required) |
| Bandwidth | Typically up to ~1.25 Gbps per tunnel | 1 Gbps to 100 Gbps (dedicated or hosted connections) |
| Latency/Consistency | Variable (depends on internet conditions) | Consistent, predictable low latency |
| Cost | Lower, pay-as-you-go | Higher fixed cost, but often cheaper at high sustained volume |
| Encryption | Built-in (IPsec) | Not encrypted by default (can layer VPN over DX for encryption) |
| Best for | Quick setup, backup connectivity, lower/variable bandwidth needs | Large, steady data transfer; latency-sensitive or compliance-driven workloads |

### Common Pattern
- Use **Direct Connect as the primary** path for consistent, high-bandwidth hybrid connectivity, with a **Site-to-Site VPN as a failover** path if the DX connection goes down.
- Both can attach to a **Transit Gateway** to centralize hybrid connectivity across many VPCs.

---

## 24. VPC Sharing (RAM)

**VPC Sharing**, via **AWS Resource Access Manager (RAM)**, lets a central "owner" account share one or more subnets from its VPC with other accounts in the same AWS Organization.

### Key Points
- Participant accounts can launch their own resources (EC2, RDS, etc.) directly into the shared subnets, **as if the subnet were their own** — while the VPC, subnet, and route tables remain centrally owned and managed.
- Reduces the need to create and peer/connect many separate VPCs — useful for a shared "landing zone" networking model where a central networking team owns the VPC and application teams just deploy into it.
- Participant accounts **cannot** modify the shared VPC's core networking (route tables, IGW, NACLs) — only the owning account can.
- Billing for data transfer/resources still applies per the participant account's own usage; the owner only manages the network layer.

---

## 25. Multi-AZ & High-Availability Design

A quick checklist for a well-architected, highly available VPC design (a common whiteboard-style interview prompt):

1. **Span at least 2–3 Availability Zones** with a public + private subnet pair in each.
2. **One NAT Gateway per AZ** (not just one for the whole VPC) so an AZ failure doesn't take down outbound internet access for the other AZs.
3. Place **Auto Scaling Groups / ELB targets across all AZs** so instance loss in one AZ doesn't take the app down.
4. Use **Multi-AZ RDS** (synchronous standby in a second AZ) for database resilience, placed in private data subnets.
5. Use **Transit Gateway** (not point-to-point peering) once you have more than a handful of VPCs to connect.
6. Enable **VPC Flow Logs** for visibility, and **CloudTrail** for API-level auditing of network changes.
7. Reserve extra CIDR space per subnet/VPC up front — resizing later is disruptive.

---

## 26. Common Interview Questions

1. **What's the difference between a security group and a NACL?** → [Section 9](#9-security-groups-vs-nacls) — stateful/instance-level vs. stateless/subnet-level.
2. **How does a private subnet get internet access?** → Via a **NAT Gateway** in a public subnet ([Section 5](#5-nat-gateway)).
3. **What makes a subnet "public"?** → Its route table has a route to an **Internet Gateway** ([Sections 4, 6](#4-internet-gateway-igw)).
4. **How do you connect 20 VPCs together with transitive routing?** → **AWS Transit Gateway**, not VPC Peering ([Sections 10–11](#10-vpc-peering)).
5. **How do you access S3 from a private subnet without a NAT Gateway?** → A **Gateway Endpoint** for S3 ([Section 15](#15-gateway-endpoints)).
6. **How do you expose a single internal service to another team's VPC without full network peering?** → **AWS PrivateLink** / Interface Endpoint with an NLB ([Sections 12, 14](#12-vpc-peering-vs-transit-gateway-vs-privatelink)).
7. **Why can't you peer two VPCs with the same CIDR range?** → Overlapping IP ranges create ambiguous routing ([Section 10](#10-vpc-peering)).
8. **How many usable IPs are in a /24 subnet?** → 251 (5 are reserved by AWS) ([Section 2](#2-cidr-blocks)).
9. **How do you SSH into a private instance securely?** → A bastion host, or (preferred today) **Systems Manager Session Manager** with no open inbound ports ([Section 18](#18-bastion-hosts)).
10. **How do you debug why traffic is being blocked between two instances?** → **VPC Flow Logs**, checking for `REJECT` records, then reviewing security groups and NACLs ([Section 19](#19-vpc-flow-logs)).
11. **NAT Gateway vs NAT Instance — which would you recommend and why?** → NAT Gateway, for its managed HA and bandwidth scaling ([Section 5](#5-nat-gateway)).
12. **Site-to-Site VPN vs Direct Connect — when would you use each?** → [Section 23](#23-site-to-site-vpn-vs-aws-direct-connect).
13. **How does IPv6 outbound-only access work if there's no "private" IPv6 range?** → **Egress-Only Internet Gateway** ([Sections 21–22](#21-ipv6-in-a-vpc)).
