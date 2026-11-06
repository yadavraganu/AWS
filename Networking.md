## Table of Contents

1. [How to Think About AWS Networking](#1-how-to-think-about-aws-networking)
2. [Amazon VPC Fundamentals](#2-amazon-vpc-fundamentals)
3. [IP Addressing and CIDR](#3-ip-addressing-and-cidr)
4. [Subnets and Availability Zones](#4-subnets-and-availability-zones)
5. [Route Tables and Routing Decisions](#5-route-tables-and-routing-decisions)
6. [Internet Gateway and Public IPv4](#6-internet-gateway-and-public-ipv4)
7. [NAT Gateway and Outbound Connectivity](#7-nat-gateway-and-outbound-connectivity)
8. [Security Groups](#8-security-groups)
9. [Network ACLs](#9-network-acls)
10. [VPC Traffic: Put the Pieces Together](#10-vpc-traffic-put-the-pieces-together)
11. [VPC Endpoints and AWS PrivateLink](#11-vpc-endpoints-and-aws-privatelink)
12. [Connecting VPCs: Peering and Transit Gateway](#12-connecting-vpcs-peering-and-transit-gateway)
13. [Hybrid Connectivity: VPN and Direct Connect](#13-hybrid-connectivity-vpn-and-direct-connect)
14. [DNS, Route 53 Resolver, and DHCP](#14-dns-route-53-resolver-and-dhcp)
15. [IPv6 Networking](#15-ipv6-networking)
16. [Elastic IPs and Elastic Network Interfaces](#16-elastic-ips-and-elastic-network-interfaces)
17. [Load Balancers and Network Placement](#17-load-balancers-and-network-placement)
18. [Remote Administration: Bastion and Session Manager](#18-remote-administration-bastion-and-session-manager)
19. [Network Observability and Troubleshooting](#19-network-observability-and-troubleshooting)
20. [Advanced Network Security and Governance](#20-advanced-network-security-and-governance)
21. [High Availability, Resilience, and Cost Design](#21-high-availability-resilience-and-cost-design)
22. [Reference Architectures](#22-reference-architectures)
23. [Practical Troubleshooting Runbook](#23-practical-troubleshooting-runbook)
24. [Common Misconceptions and Corrections](#24-common-misconceptions-and-corrections)
25. [Interview Questions and Answers](#25-interview-questions-and-answers)
26. [Glossary and Further Learning](#26-glossary-and-further-learning)

---

# 1. How to Think About AWS Networking

AWS networking is easier to understand when you separate five questions:

1. **Addressing:** What IP address does a resource use?
2. **Placement:** Which VPC, subnet, and Availability Zone contains it?
3. **Routing:** Where does a packet go next?
4. **Permission:** Is the packet allowed by security controls?
5. **Name resolution:** How does an application turn a DNS name into an IP address?

A network connection generally succeeds only when all relevant layers are configured correctly. A route does not grant permission, and a security rule does not create a route.

## 1.1 The core traffic path

For an EC2 instance communicating with another system, reason in this order:

- Resolve the destination name through DNS, if the application uses a name.
- Check the source resource's network interface and security groups.
- Check the subnet's associated route table for the destination.
- Check the target gateway or connection and the return route.
- Check the destination-side security group and NACLs.
- Check host firewall, application listener, TLS, and application-level authorization.

For a stateful security group, response traffic for an allowed connection is automatically tracked. NACLs are stateless, so both directions must be allowed.

## 1.2 Important network layers

| Layer | AWS concept | Main question |
|---|---|---|
| Network boundary | VPC | Which logically isolated network? |
| IP segmentation | CIDR and subnet | Which address range and AZ? |
| Forwarding | Route table | Where should this destination go? |
| Stateful filtering | Security group | Which connections are allowed? |
| Stateless filtering | Network ACL | Which packets cross the subnet boundary? |
| Internet connectivity | IGW, NAT, Egress-only IGW | How does traffic reach or leave the internet? |
| Private service connectivity | VPC endpoints / PrivateLink | How can a service be reached privately? |
| Multi-network connectivity | Peering / Transit Gateway | How do networks communicate? |
| Hybrid connectivity | VPN / Direct Connect | How does on-premises connect to AWS? |
| Name resolution | Route 53 Resolver | What IP address does a name map to? |
| Visibility | Flow Logs and analyzers | What is happening and why? |

---

# 2. Amazon VPC Fundamentals

An **Amazon Virtual Private Cloud (VPC)** is a logically isolated virtual network in an AWS account. You define its IP address ranges, subnets, route tables, gateways, and network security.

## 2.1 Regional scope

- A VPC is a **Regional** resource. It can span multiple Availability Zones (AZs) in one AWS Region.
- A subnet belongs to **exactly one AZ**.
- A VPC does not automatically span Regions. To connect networks in different Regions, use supported connectivity such as inter-Region VPC peering or Transit Gateway peering.
- Many VPC components are Regional, but some components are AZ-scoped. NAT Gateways and subnets are examples of AZ-scoped resources.

## 2.2 Default VPC and custom VPC

AWS can create a default VPC in a Region. A typical default VPC includes:

- A subnet in each AZ in that Region.
- An attached Internet Gateway.
- A main route table with a default route to the IGW.
- Default DNS settings.

This makes it convenient to launch an instance quickly, but it is not a substitute for a deliberate production network design.

A **custom VPC** gives the team control over CIDR allocation, subnet tiers, routing, and isolation.

## 2.3 VPC components

| Component | Responsibility |
|---|---|
| CIDR block | Defines the VPC's IP address range |
| Subnet | Allocates a smaller address range in one AZ |
| Route table | Selects the next hop for traffic |
| Internet Gateway | Enables internet routing for eligible IPv4/IPv6 resources |
| NAT Gateway | Enables outbound IPv4 connections from private networks |
| Security group | Stateful firewall attached to network interfaces/resources |
| Network ACL | Stateless filter applied at subnet boundaries |
| VPC endpoint | Private path to supported services |
| VPC peering | Direct private routing between two VPCs |
| Transit Gateway | Central hub for multiple networks |
| Flow Logs | Records metadata about IP traffic |

## 2.4 VPC limits and quotas

AWS quotas vary by Region and can change. Do not memorize a quota as an immutable architectural fact. Check **Service Quotas** and the current VPC documentation for the account and Region being designed. Many quotas are adjustable; some service constraints are not.

---

# 3. IP Addressing and CIDR

An IP address identifies a network interface or endpoint. AWS VPC designs use IPv4, IPv6, or both.

## 3.1 CIDR notation

CIDR means **Classless Inter-Domain Routing**. The suffix after `/` indicates how many leading bits identify the network.

Examples:

| CIDR | Total IPv4 addresses | Intuition |
|---|---:|---|
| `10.0.0.0/16` | 65,536 | Large VPC range |
| `10.0.1.0/24` | 256 | Common subnet size |
| `10.0.1.0/26` | 64 | Smaller subnet |
| `10.0.1.0/28` | 16 | Very small subnet |

Formula for an IPv4 CIDR:

`Number of addresses = 2^(32 - prefix length)`

For example, `/24` has `2^(32-24) = 256` total addresses.

## 3.2 Private IPv4 ranges

The RFC 1918 private ranges are:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

These are commonly used for VPCs. Private addresses are not directly routed over the public internet.

AWS does not require every VPC IPv4 CIDR to be RFC 1918, but public or overlapping ranges require careful planning. A VPC's address range is not advertised to the public internet merely because it uses a globally routable range.

## 3.3 VPC and subnet CIDR sizing

For a VPC's primary IPv4 CIDR, AWS supports sizes from `/16` through `/28`. Subnet IPv4 CIDRs also have AWS-specific size constraints. Check current AWS documentation before relying on edge sizes or secondary CIDR behavior.

A common layout:

```text
VPC: 10.20.0.0/16

AZ-a
  Public subnet:       10.20.1.0/24
  Private app subnet:  10.20.11.0/24
  Private data subnet: 10.20.21.0/24

AZ-b
  Public subnet:       10.20.2.0/24
  Private app subnet:  10.20.12.0/24
  Private data subnet: 10.20.22.0/24
```

The exact CIDRs are examples, not mandatory AWS conventions.

## 3.4 Five IPv4 addresses reserved in every subnet

AWS reserves five IPv4 addresses in each subnet:

| Address | Purpose |
|---|---|
| First address | Network address |
| Second address | VPC router |
| Third address | Amazon-provided DNS |
| Fourth address | Reserved for future use |
| Last address | Reserved; historically associated with broadcast behavior, which VPC does not support |

Therefore, a `/24` has 256 total IPv4 addresses and **251 usable addresses** for resources.

A `/28` has 16 total addresses and 11 usable addresses.

This reservation applies to IPv4 subnet addressing. Do not apply the five-address rule to IPv6.

## 3.5 Plan for growth and avoid overlapping ranges

When connecting networks, overlapping CIDRs cause routing ambiguity or prevent supported connections.

Plan address space across:

- AWS accounts and Regions.
- Development, test, staging, and production.
- On-premises networks.
- Acquired companies or partner networks.
- EKS pod/service networking, if applicable.
- Future VPCs and shared services.

For a multi-account organization, central IP Address Manager (IPAM) can help allocate and track CIDRs.

## 3.6 Secondary CIDRs

A VPC can have secondary IPv4 CIDR blocks added when the primary range is insufficient, subject to AWS constraints. Adding address space later can help, but it does not remove the need to plan for non-overlap and routing.

---

# 4. Subnets and Availability Zones

A **subnet** is a range of IP addresses inside a VPC and is associated with one AZ. Subnets are often used to separate workloads by exposure, function, or security boundary.

## 4.1 Public, private, and isolated subnets

These labels are determined primarily by routing, not by the subnet's name.

### Public subnet

A subnet is conventionally called public when its associated route table has a route to an Internet Gateway, typically:

```text
Destination     Target
10.20.0.0/16    local
0.0.0.0/0       igw-xxxxxxxx
```

A resource also needs a suitable public IPv4 address or IPv6 address, applicable security rules, and a listening service to be reachable from the internet.

**Important:** An IGW route alone does not automatically assign a public IP or make every resource publicly reachable.

Typical resources: internet-facing load balancers, NAT Gateways, and carefully controlled public endpoints.

### Private subnet

A private subnet does not have a direct route to an IGW for general internet access. It may have a default route to a NAT Gateway:

```text
Destination     Target
10.20.0.0/16    local
0.0.0.0/0       nat-xxxxxxxx
```

Typical resources: application servers, containers, private APIs, and workers.

### Isolated subnet

An isolated subnet has no route to an external network such as an IGW, NAT Gateway, VPN, or Transit Gateway. It may still communicate with resources through the VPC's local route and configured endpoints.

Typical resources: databases or workloads with deliberately restricted egress.

## 4.2 A common three-tier design

| Tier | Common placement | Purpose |
|---|---|---|
| Public | Public subnets in multiple AZs | Internet-facing load balancer, NAT Gateway |
| Application | Private subnets in multiple AZs | EC2, ECS, EKS worker nodes, application services |
| Data | Private/isolated subnets in multiple AZs | RDS, caches, internal data stores |

The tier is a design convention. AWS does not enforce the meaning of “public,” “app,” or “data” from a subnet name.

## 4.3 Subnet design practices

- Use multiple AZs for production workloads that need AZ resilience.
- Create separate subnet tiers when they need different routing or security boundaries.
- Leave address-space room for scaling and managed services.
- Remember that a subnet cannot be moved to another AZ after creation.
- Some managed services require free IP capacity in selected subnets for scaling and maintenance.
- Avoid placing every workload in one large subnet merely because security groups exist; subnet boundaries can simplify routing and governance.

---

# 5. Route Tables and Routing Decisions

A **route table** is a set of routes that tells AWS where to send traffic based on the destination IP address.

A route contains:

- **Destination:** an IPv4 CIDR, IPv6 CIDR, or prefix list.
- **Target:** where traffic should be forwarded, such as `local`, IGW, NAT Gateway, peering connection, Transit Gateway, or a VPC endpoint.

## 5.1 Main route and subnet associations

- Every VPC has a main route table.
- Every subnet is associated with exactly one route table at a time.
- A subnet without an explicit association uses the main route table.
- One route table can be associated with multiple subnets.
- A route table can also be associated with certain gateways for edge routing use cases.

For production, explicitly associate purpose-built route tables with subnets and keep the main route table conservative.

## 5.2 The local route

A VPC route table includes a `local` route for the VPC's address ranges. It enables routing between resources in the VPC, subject to security controls.

Example:

| Destination | Target |
|---|---|
| `10.20.0.0/16` | `local` |
| `0.0.0.0/0` | `igw-123456` |

The local route is normally present automatically. Advanced designs can use supported route replacement features, but that is not a beginner default.

## 5.3 Longest prefix match

When multiple routes match a destination, AWS uses the most specific matching route (the route with the longest prefix).

Example:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | NAT Gateway |
| `10.30.0.0/16` | Transit Gateway |
| `10.30.5.0/24` | VPC peering |

Traffic for `10.30.5.10` matches all three, but `/24` is the most specific, so the peering route wins.

Do not assume that a default route overrides a more specific private-network route.

## 5.4 Example route tables

**Public subnet**

| Destination | Target |
|---|---|
| `10.20.0.0/16` | `local` |
| `0.0.0.0/0` | Internet Gateway |

**Private application subnet**

| Destination | Target |
|---|---|
| `10.20.0.0/16` | `local` |
| `0.0.0.0/0` | NAT Gateway |
| S3 managed prefix list | S3 gateway endpoint |

**Isolated data subnet**

| Destination | Target |
|---|---|
| `10.20.0.0/16` | `local` |
| S3 managed prefix list | S3 gateway endpoint, if required |

## 5.5 Route table troubleshooting

When traffic fails, inspect:

1. Which route table is actually associated with the source subnet?
2. Is there a route matching the destination?
3. Is the route target available and in the correct state?
4. Does the destination have a return route?
5. Is a more-specific route taking precedence?
6. Are security groups and NACLs allowing the flow?

A route table is not a firewall. A valid route only determines the forwarding path.

---

# 6. Internet Gateway and Public IPv4

An **Internet Gateway (IGW)** is a horizontally scaled, redundant, highly available VPC component that allows communication between a VPC and the internet.

## 6.1 What an IGW does

- It is attached to a VPC.
- It is the route-table target for internet-bound traffic.
- For IPv4, it performs the necessary one-to-one NAT mapping between a resource's private IPv4 address and its public IPv4 address.
- For IPv6, addresses are globally routable; the IGW provides the internet path, while security rules control access.

An IGW is not a substitute for public IP assignment, routes, or firewall rules.

## 6.2 Requirements for inbound IPv4 internet access

A typical EC2 instance needs all of the following:

1. A public IPv4 address or Elastic IP associated with its network interface.
2. A subnet route table with `0.0.0.0/0` targeting the IGW.
3. A security group inbound rule allowing the required protocol and port from the intended source.
4. NACL rules that permit the request and response.
5. A host firewall and application that accept the connection.

If any one is missing, the connection may fail.

## 6.3 Public IP assignment

A subnet has a setting to auto-assign public IPv4 addresses to eligible network interfaces at launch. Launch configuration can override this behavior. An automatically assigned public IPv4 address is generally not retained when an instance is stopped and started; an Elastic IP is stable until disassociated or released.

AWS charges for public IPv4 addresses, including addresses associated with running resources. Check current pricing before designing around many public addresses.

---

# 7. NAT Gateway and Outbound Connectivity

A **NAT Gateway** allows resources in a private IPv4 subnet to initiate connections to destinations outside the VPC, commonly the public internet, without accepting unsolicited inbound connections initiated from those destinations.

## 7.1 Public NAT Gateway packet path

A public NAT Gateway is created in a public subnet and uses an Elastic IP.

Example:

```text
Private EC2 (10.20.11.10)
       |
       | Route: 0.0.0.0/0 -> NAT Gateway
       v
NAT Gateway (public subnet, Elastic IP)
       |
       | Public subnet route: 0.0.0.0/0 -> IGW
       v
Internet Gateway
       |
       v
Internet destination
```

The NAT Gateway translates the source address/port for outbound IPv4 connections and maps response traffic back to the originating private resource.

## 7.2 Configuration checklist

1. Create a public subnet.
2. Attach an IGW to the VPC.
3. Add `0.0.0.0/0 -> IGW` to the public subnet route table.
4. Create a **public NAT Gateway** in that public subnet and allocate an Elastic IP.
5. Add `0.0.0.0/0 -> NAT Gateway` to the private subnet route table.
6. Ensure security groups and NACLs permit the required flow.

A NAT Gateway does not make a private EC2 instance publicly addressable.

## 7.3 Public NAT Gateway versus private NAT Gateway

- **Public NAT Gateway:** Used for internet-bound IPv4 traffic through an IGW. It uses an Elastic IP.
- **Private NAT Gateway:** Used to connect to other private networks, such as networks reachable through a Transit Gateway or virtual private gateway. It does not provide internet access through an IGW.

Do not assume every NAT Gateway is an internet gateway replacement.

## 7.4 Availability and scaling

A NAT Gateway is an AZ-scoped resource. It is managed by AWS and scales automatically. AWS documentation currently describes 5 Gbps initial bandwidth scaling up to 100 Gbps and packet processing scaling from 1 million to 10 million packets per second; these limits should be rechecked in the current documentation.

For resilient designs, create one NAT Gateway per AZ and route each AZ's private subnets to the NAT Gateway in the same AZ. This avoids depending on a single AZ's NAT Gateway and can reduce cross-AZ data-transfer charges.

A single NAT Gateway shared across AZs may be a deliberate cost trade-off for non-critical environments, but it creates an AZ dependency and cross-AZ traffic costs.

## 7.5 NAT Gateway costs

Common cost components include:

- Hourly charge while provisioned.
- Per-GB data processing charge.
- Applicable data transfer charges, including cross-AZ transfer where relevant.
- Public IPv4 address charges.

Reduce unnecessary NAT processing by using gateway endpoints for S3 and DynamoDB and interface endpoints for frequently used supported AWS services where the endpoint economics make sense.

## 7.6 NAT Gateway versus NAT instance

| Aspect | NAT Gateway | NAT instance |
|---|---|---|
| Management | AWS managed | Customer managed EC2 |
| Scaling | Managed scaling | Instance-size and architecture dependent |
| Availability | Managed within one AZ | Customer builds failover |
| Security groups | Not attached like an EC2 instance | Can use security groups |
| Customization | Limited | Full OS/software control |
| Operations | Low operational burden | Patching, monitoring, scaling, failover |
| Typical choice | Default for most workloads | Specialized or legacy cases |

---

# 8. Security Groups

A **security group (SG)** is a stateful virtual firewall associated with network interfaces and supported AWS resources.

## 8.1 Key properties

- **Stateful:** Return traffic for an allowed connection is automatically permitted.
- **Allow rules only:** There are no explicit deny rules. Unmatched traffic is denied.
- **Rules are additive:** If multiple SGs are attached to a network interface, their allowed rules are combined.
- **ENI-level enforcement:** SGs are associated with network interfaces, even when the console presents them as attached to an instance.
- **References:** Rules can reference other security groups in supported network relationships.

## 8.2 Inbound and outbound rules

An inbound rule controls traffic that can reach the resource. An outbound rule controls traffic initiated by the resource.

Example web/application/database pattern:

| Security group | Direction | Protocol/port | Source/destination |
|---|---|---|---|
| ALB-SG | Inbound | TCP 443 | Approved client CIDRs |
| ALB-SG | Outbound | TCP 8080 | App-SG |
| App-SG | Inbound | TCP 8080 | ALB-SG |
| App-SG | Outbound | TCP 5432 | DB-SG |
| DB-SG | Inbound | TCP 5432 | App-SG |

This is safer than opening a database port to `0.0.0.0/0`.

## 8.3 Security group references

Instead of hard-coding an application subnet's CIDR, an inbound rule can allow traffic from another SG. This expresses an identity-like network relationship: allow the relevant port from interfaces associated with that source SG.

SG references have topology and Region constraints. For example, references can be used within the same VPC and in supported peering or Transit Gateway relationships. Cross-Region VPC peering does not support peer SG references; use CIDR-based rules there. Confirm current support for the specific connection type.

## 8.4 Default behavior

A newly created SG normally has no inbound allow rules and a default allow-all outbound rule. Always review outbound rules against the workload's actual egress requirements.

## 8.5 Good practices

- Use separate SGs for load balancers, application tiers, databases, and administration.
- Reference SGs instead of broad CIDRs where supported.
- Avoid unrestricted SSH/RDP from the internet.
- Keep rule descriptions meaningful.
- Review unused and overly broad rules periodically.
- Remember that SGs do not filter traffic to the Amazon-provided DNS resolver in the same way as ordinary destinations; use the relevant VPC DNS controls and Route 53 Resolver features.

---

# 9. Network ACLs

A **Network Access Control List (NACL)** is a stateless filter applied at the subnet boundary.

## 9.1 Key properties

- Stateless: inbound and outbound directions are evaluated independently.
- Supports both ALLOW and DENY rules.
- Rules have numbers and are evaluated from lowest to highest.
- The first matching rule is applied.
- Each subnet is associated with one NACL at a time.
- One NACL can be associated with multiple subnets.
- The default NACL allows all inbound and outbound IPv4 traffic (and IPv6 where configured); a newly created custom NACL denies traffic until rules are added.

## 9.2 Ephemeral ports and return traffic

For TCP, the client usually connects from an ephemeral source port to a server's listening destination port. The server's response has the reverse source/destination ports.

Example: client `10.20.11.10:53000` connects to server `10.20.21.20:443`.

- Request: source port `53000`, destination port `443`.
- Response: source port `443`, destination port `53000`.

Because a NACL is stateless, it must allow both packet directions. The required ephemeral port range depends on the client operating system and connection pattern. `1024–65535` is a commonly used broad example, not a universal rule to apply blindly.

## 9.3 Example: web server in a subnet

If clients connect to a web server on TCP 443:

- Inbound NACL: allow TCP destination port 443 from the intended client CIDR.
- Outbound NACL: allow TCP destination ephemeral ports for response traffic to those clients.
- Also consider outbound connections initiated by the server and their return traffic.

The exact rules must account for both directions and all required flows.

## 9.4 Security group versus NACL

| Aspect | Security group | NACL |
|---|---|---|
| Scope | Network interface/resource | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny |
| Evaluation | All rules contribute; matching allow permits | Ordered; first matching rule wins |
| Return traffic | Automatically tracked | Must be allowed separately |
| Typical role | Primary workload firewall | Coarse subnet boundary control |

Use SGs for most workload-level access control. Use NACLs when subnet-wide stateless filtering or explicit deny behavior is useful. Avoid complex NACL rule sets unless there is a clear requirement.

---

# 10. VPC Traffic: Put the Pieces Together

This section combines the concepts into concrete packet paths.

## 10.1 Public EC2 instance to the internet

Requirements:

- Public IP or Elastic IP on the instance's network interface.
- Subnet route `0.0.0.0/0 -> IGW`.
- Outbound SG permission.
- NACL permits outbound and return traffic.
- Host firewall and application allow the connection.

For an internet client to initiate a connection to the instance, also allow the inbound port in the SG and NACL and ensure the application is listening.

## 10.2 Private EC2 instance to the internet

Requirements:

- Private subnet route `0.0.0.0/0 -> public NAT Gateway`.
- NAT Gateway in a public subnet.
- Public subnet route `0.0.0.0/0 -> IGW`.
- NAT Gateway has an Elastic IP.
- SG and NACL permit the connection and return traffic.

The private instance does not need a public IP.

## 10.3 Private EC2 instance to S3

Preferred common design:

- Create an S3 **gateway endpoint**.
- Associate it with the route table used by the private subnet.
- Use an endpoint policy and S3/IAM policies to control access.
- Ensure DNS and application configuration use the expected S3 endpoint behavior.

This avoids sending S3 traffic through a NAT Gateway. A gateway endpoint does not create an ENI and does not have a security group.

## 10.4 Application tier to database tier

- App and DB resources use private subnets.
- App SG outbound allows the database port to DB-SG.
- DB-SG inbound allows the database port from App-SG.
- VPC local routing provides the in-VPC path.
- NACLs must allow both directions if they are restrictive.
- The database must be listening and its own authentication must succeed.

## 10.5 VPC to on-premises

A typical hybrid path uses Site-to-Site VPN or Direct Connect, often attached through a Transit Gateway for multiple VPCs. Routing must exist on both sides, and the on-premises firewall and AWS security controls must permit the traffic.

---

# 11. VPC Endpoints and AWS PrivateLink

A **VPC endpoint** provides a private path from a VPC to supported AWS services or endpoint services without requiring a public internet route for that traffic.

There are multiple endpoint types. Do not treat all VPC endpoints as interchangeable.

## 11.1 Gateway endpoints

Gateway endpoints support:

- Amazon S3
- Amazon DynamoDB

Characteristics:

- Added as a route-table target.
- Do not create an ENI in your subnet.
- No hourly endpoint charge or endpoint data-processing charge.
- Not protected by a security group.
- Use endpoint policies plus IAM and service resource policies for authorization.
- Not directly reachable from on-premises through VPN or Direct Connect.
- Gateway endpoints are Regional and are used by resources in the VPC through associated route tables.

For many VPC workloads accessing S3 or DynamoDB, a gateway endpoint is a cost-effective first choice.

## 11.2 Interface endpoints

An **interface endpoint** is powered by AWS PrivateLink. AWS creates endpoint network interfaces (ENIs) with private IP addresses in selected subnets.

Characteristics:

- Supports many AWS services and supported partner/customer services.
- Security groups can be associated with endpoint ENIs.
- Endpoint policies can restrict access.
- Private DNS can make standard service DNS names resolve to endpoint private addresses, when supported and configured.
- Usually billed per endpoint-AZ-hour and data processed; consult current pricing.
- Can be accessed from on-premises over VPN or Direct Connect when routing and DNS are configured appropriately.

For resilience, deploy interface endpoint ENIs in multiple AZs when the workload requires AZ-level availability.

## 11.3 Gateway versus interface endpoint

| Aspect | Gateway endpoint | Interface endpoint |
|---|---|---|
| Mechanism | Route-table target | ENI with private IP |
| Services | S3 and DynamoDB | Many AWS services and supported endpoint services |
| Security group | No | Yes |
| Endpoint policy | Yes | Yes |
| Hourly/data processing endpoint fee | No | Generally yes |
| On-premises access | No | Supported with suitable connectivity and DNS |
| Common use | In-VPC S3/DynamoDB access | Private access to service APIs or PrivateLink services |

S3 and DynamoDB can also support interface endpoints. These are useful in scenarios such as on-premises access or access from another VPC, but they are not automatically a replacement for the free gateway endpoint.

## 11.4 AWS PrivateLink service publishing

A service provider can expose a service privately to consumers using an endpoint service, commonly fronted by a Network Load Balancer. Consumers create interface endpoints in their VPCs.

Benefits:

- Consumer does not need a route to the provider's entire VPC.
- Provider does not need to expose its VPC CIDR.
- Access is service-oriented rather than full network connectivity.
- Useful across accounts and for third-party service access.

PrivateLink is not the same as VPC peering: it exposes a service endpoint, not general IP reachability to the provider network.

## 11.5 Endpoint policies versus IAM and resource policies

An endpoint policy is an additional policy layer. It does not replace IAM identity policies or the service's resource policy.

A request may be denied by any applicable authorization layer. For S3, for example, consider IAM, bucket policy, endpoint policy, and any organization-level controls.

---

# 12. Connecting VPCs: Peering and Transit Gateway

## 12.1 VPC Peering

VPC Peering creates a one-to-one network connection between two VPCs, allowing resources to communicate using private IP addresses.

It can connect VPCs:

- In the same AWS account or different accounts.
- In the same Region or different Regions, where supported.

### Setup at a high level

1. Create a peering request from one VPC.
2. The owner of the accepter VPC accepts it.
3. Add routes in the relevant subnet route tables on both sides.
4. Update SGs and NACLs as required.
5. Verify DNS resolution requirements if private DNS names are involved.

### Limitations

- VPC CIDR ranges must not overlap for a peering connection.
- Peering is **not transitive**. If A peers with B and B peers with C, A cannot use B as a router to reach C through those peering connections.
- A peer connection does not automatically add routes.
- A peering connection does not provide centralized firewalling or routing.
- Security-group references have constraints; cross-Region peer SG references are not supported.

### When to use

Use peering for a small number of VPCs that need direct, straightforward private connectivity and do not require transitive routing.

## 12.2 AWS Transit Gateway

**AWS Transit Gateway (TGW)** is a Regional network hub that connects VPCs and supported network attachments such as VPN and Direct Connect gateway integrations.

It provides a hub-and-spoke model.

### Why use TGW?

Without a hub, connecting every VPC to every other VPC requires a full mesh. For `N` VPCs, the number of pairwise connections is:

`N × (N - 1) / 2`

For 10 VPCs, that is 45 connections. With a hub, each VPC generally needs one attachment to the TGW, subject to architecture and routing needs.

### Core concepts

- **Attachment:** Connects a VPC, VPN, Direct Connect gateway, or supported network to TGW.
- **TGW route table:** Determines which attachments can reach which destination networks.
- **Association:** Each attachment is associated with one TGW route table at a time.
- **Propagation:** An attachment can advertise its routes into one or more TGW route tables.
- **Static route:** An administrator-defined route.
- **Blackhole route:** A route that intentionally drops matching traffic.
- **Peering attachment:** Connects Transit Gateways, including across Regions; routes across TGW peering are configured with static routes.

### VPC attachment routing

A VPC attachment alone is not enough. You generally need:

1. A VPC subnet route pointing the destination CIDR to the TGW.
2. A TGW route table route pointing the destination to the correct attachment.
3. A return route in the destination network.
4. Security controls permitting the traffic.

### Segmentation example

- Production VPC attachments associate with a production TGW route table.
- Development VPC attachments associate with a development TGW route table.
- Shared-services routes are selectively propagated or configured.
- No route is added between development and production unless required.

This is how TGW route tables can enforce network segmentation at the routing layer. They do not replace security groups or firewalls.

## 12.3 Peering versus Transit Gateway versus PrivateLink

| Requirement | Suitable option |
|---|---|
| Two VPCs need simple direct private connectivity | VPC Peering |
| Many VPCs need centralized, transitive routing | Transit Gateway |
| VPCs need segmented hub-and-spoke connectivity | Transit Gateway with multiple route tables |
| Consumer needs one provider service, not the whole network | PrivateLink |
| On-premises network needs centralized access to many VPCs | Transit Gateway with VPN/DX integration |
| CIDRs overlap and only a service needs to be shared | Consider PrivateLink, if the service pattern is supported |

## 12.4 Connectivity is not authorization

Peering or TGW provides a network path. It does not automatically permit application traffic. SGs, NACLs, firewalls, host firewalls, and application authorization still matter.

---

# 13. Hybrid Connectivity: VPN and Direct Connect

Hybrid networking connects AWS VPCs to corporate data centers, offices, colocation facilities, or other external networks.

## 13.1 AWS Site-to-Site VPN

Site-to-Site VPN creates IPsec VPN tunnels between AWS and a customer gateway device.

Typical components:

- **Customer Gateway (CGW):** AWS representation of the customer-side VPN device and its public IP/BGP configuration.
- **Virtual Private Gateway (VGW):** AWS gateway attached to one VPC.
- **Transit Gateway:** Alternative central termination point for VPN connections.
- **VPN tunnels:** Redundant IPsec tunnels are provided for availability.
- **BGP:** Dynamic routing can exchange routes when configured; static routing is also possible in supported configurations.

VPN traffic uses the internet as its underlying transport, but the IPsec tunnel encrypts the traffic.

## 13.2 AWS Direct Connect (DX)

Direct Connect provides a dedicated network connection from a customer network or colocation facility to AWS.

Common concepts:

- **Dedicated connection:** Physical connection provisioned for one customer.
- **Hosted connection:** Capacity provided through a Direct Connect Partner.
- **Virtual interface (VIF):** Logical interface used to access supported AWS network destinations.
- **Private VIF:** Commonly used to access VPCs through a virtual private gateway or Direct Connect gateway.
- **Transit VIF:** Used to reach Transit Gateways through a Direct Connect gateway.
- **Public VIF:** Used to reach supported AWS public services over the Direct Connect connection.
- **Direct Connect gateway:** Enables connectivity patterns across supported Regions/accounts and multiple gateways, subject to allowed prefixes and service constraints.

Direct Connect is not encrypted by default. Use application-layer encryption or an appropriate VPN overlay when encryption in transit is required.

## 13.3 VPN versus Direct Connect

| Aspect | Site-to-Site VPN | Direct Connect |
|---|---|---|
| Transport | IPsec over internet | Dedicated or partner-provided private connection |
| Setup | Usually faster | Provisioning may take longer |
| Performance | Depends on internet and tunnel configuration | More predictable network path |
| Encryption | IPsec | Not encrypted by default |
| Use | Rapid setup, backup, smaller/variable traffic | Consistent private connectivity and sustained traffic |
| Resilience | Use redundant tunnels and devices | Use redundant connections/locations and optionally VPN backup |

Do not describe Direct Connect as inherently encrypted or as a complete substitute for redundant network design.

## 13.4 Common hybrid design

A common enterprise pattern is Direct Connect for primary connectivity and Site-to-Site VPN as backup. Both can integrate with Transit Gateway, but routing, BGP preferences, allowed prefixes, and failover behavior must be explicitly designed and tested.

## 13.5 AWS Client VPN

AWS Client VPN is a managed remote-access VPN service that allows individual users/devices to connect to AWS and, with suitable routes and authorization rules, reach VPC or on-premises networks.

It is different from Site-to-Site VPN:

- Site-to-Site connects networks.
- Client VPN connects remote users/devices.

---

# 14. DNS, Route 53 Resolver, and DHCP

DNS translates hostnames into IP addresses. A workload can have perfect routing and security rules but still fail if DNS resolution is incorrect.

## 14.1 VPC DNS attributes

Two important VPC attributes:

| Attribute | Meaning |
|---|---|
| `enableDnsSupport` | Enables DNS resolution through the Amazon-provided Route 53 Resolver |
| `enableDnsHostnames` | Enables DNS hostnames for resources where applicable |

Some services and private DNS endpoint configurations require the relevant DNS attributes to be enabled.

## 14.2 Amazon-provided DNS resolver

The Route 53 Resolver is available inside a VPC. Its IPv4 address can be reached at:

- The primary IPv4 CIDR of the VPC plus two (for example, `10.20.0.2` for `10.20.0.0/16`).
- The link-local address `169.254.169.253`.

It is **not** a different `.2` address in every subnet. The VPC resolver is a managed service.

## 14.3 Route 53 private hosted zones

A private hosted zone contains DNS records resolvable from associated VPCs. It is useful for internal service names such as:

```text
orders.internal.example.com -> 10.20.11.25
```

A private hosted zone must be associated with the VPCs that should resolve its records, directly or through supported association workflows.

## 14.4 Route 53 Resolver endpoints and rules

For hybrid DNS:

- **Inbound Resolver endpoint:** Lets DNS resolvers outside the VPC query Route 53 Resolver for names associated with AWS private DNS.
- **Outbound Resolver endpoint:** Lets VPC workloads forward selected DNS queries to external DNS resolvers.
- **Resolver rules:** Define which DNS domains are forwarded and to which targets.
- **Resolver rule sharing:** Rules can be shared across accounts using supported AWS mechanisms.

Plan endpoint subnets, security groups, routing, and high availability across AZs.

## 14.5 DHCP option sets

A VPC can use a DHCP option set to provide network configuration such as domain name, domain-name servers, and NTP settings to instances. AWS-provided DNS is the default in common VPC configurations. Custom DNS can be configured where required, but ensure that instances can reach those DNS servers and that forwarding rules are correct.

## 14.6 DNS troubleshooting

Check:

1. Is `enableDnsSupport` enabled?
2. Is the expected DNS hostname behavior enabled?
3. Does the queried name exist in the right public/private hosted zone?
4. Is the private hosted zone associated with the VPC?
5. Are Resolver rules forwarding the query to the intended server?
6. Can the instance reach the custom DNS server on UDP/TCP 53?
7. Is a PrivateLink endpoint's private DNS configured and supported?

---

# 15. IPv6 Networking

IPv6 provides a much larger address space and can coexist with IPv4.

## 15.1 Dual-stack and IPv6-only

- **Dual-stack:** Resources/network paths support IPv4 and IPv6.
- **IPv6-only:** Workloads use IPv6 where supported by the service and application.
- IPv6 support must be checked for each AWS service and workload dependency.

AWS commonly allocates a `/56` IPv6 CIDR to a VPC, from which `/64` subnet ranges are assigned. Exact allocation choices depend on the IPv6 addressing option.

## 15.2 IPv6 addresses and security

AWS IPv6 addresses are globally unique and globally routable in principle. That does **not** mean every address is reachable from the internet: routes, security groups, NACLs, and other controls still apply.

IPv6 does not use IPv4 NAT as a general requirement for internet connectivity.

## 15.3 Egress-only Internet Gateway

An **Egress-Only Internet Gateway (EIGW)** allows IPv6 instances to initiate outbound internet connections while preventing the internet from initiating unsolicited inbound connections through that gateway.

Typical route:

```text
Destination     Target
::/0            eigw-xxxxxxxx
```

An EIGW is stateful and IPv6-only. It is not an IPv4 NAT Gateway.

## 15.4 IPv4 and IPv6 route examples

| Purpose | IPv4 route | IPv6 route |
|---|---|---|
| Internet-facing subnet | `0.0.0.0/0 -> IGW` | `::/0 -> IGW` |
| Outbound-only private IPv4 | `0.0.0.0/0 -> NAT GW` | — |
| Outbound-only IPv6 | — | `::/0 -> EIGW` |

For IPv6 inbound internet reachability, the IGW route and security controls must be configured appropriately.

---

# 16. Elastic IPs and Elastic Network Interfaces

## 16.1 Elastic IP address

An **Elastic IP (EIP)** is a static public IPv4 address allocated to an AWS account. It can be associated with supported resources such as a NAT Gateway or an EC2 network interface.

Uses:

- Stable public IPv4 address for allowlists.
- Remapping to another supported resource during failover.
- Stable public egress address for a NAT Gateway.

Important:

- EIPs are IPv4 addresses.
- AWS charges for public IPv4 addresses under current pricing rules, including associated and unassociated addresses.
- Avoid using an EIP when a managed load balancer or DNS-based design is more appropriate.
- An EIP does not automatically make a resource reachable; routes and security controls are still required.

## 16.2 Elastic Network Interface (ENI)

An **ENI** is a virtual network interface in a subnet. It has attributes such as:

- Private IPv4 addresses.
- Optional public IPv4/EIP association.
- MAC address.
- Security groups.
- Subnet and AZ placement.

An EC2 instance can have multiple ENIs, subject to instance-type limits. ENIs attached to one instance must be in the same AZ. A detachable ENI can be moved to another instance in the same AZ, preserving network-interface identity and private IP configuration, subject to attachment constraints.

ENIs are also used by many managed AWS services, including interface endpoints and VPC-enabled services.

---

# 17. Load Balancers and Network Placement

Load balancers are a key part of VPC architecture. They distribute incoming traffic across healthy targets.

## 17.1 Application Load Balancer (ALB)

- Operates at Layer 7 (HTTP/HTTPS).
- Supports host-based and path-based routing.
- Common for web applications and APIs.
- Can be internet-facing or internal.
- Uses subnets in multiple AZs for availability.

Example: route `/orders/*` to an orders target group and `/customers/*` to a customers target group.

## 17.2 Network Load Balancer (NLB)

- Operates primarily at Layer 4 (TCP, UDP, TLS).
- Designed for high-performance, low-latency network traffic and static IP use cases.
- Commonly used when clients need TCP/UDP connectivity or when publishing a PrivateLink endpoint service.
- Can be internet-facing or internal.

## 17.3 Gateway Load Balancer (GWLB)

- Used to deploy, scale, and manage virtual network appliances such as firewalls and intrusion detection systems.
- Uses GENEVE encapsulation to steer traffic through appliances.
- Common in centralized inspection architectures.

## 17.4 Public versus internal load balancer

- **Internet-facing:** Receives traffic from clients through public-facing addresses and public subnet placement.
- **Internal:** Receives traffic from clients with network reachability inside private networks.

A public load balancer does not mean its targets must be public. A common secure pattern is a public ALB in public subnets forwarding to application targets in private subnets.

## 17.5 Example security-group chain

```text
Internet client
   |
   v
ALB security group: allow TCP 443 from approved clients
   |
   v
Application security group: allow app port only from ALB-SG
   |
   v
Database security group: allow DB port only from App-SG
```

---

# 18. Remote Administration: Bastion and Session Manager

## 18.1 Bastion host

A bastion host (jump host) is a hardened server used as a controlled administrative entry point to private resources.

Traditional flow:

1. Administrator connects to the bastion.
2. Bastion connects to the private instance.
3. Private instance allows administration traffic only from the bastion SG.

If used, restrict source IPs, harden and patch the host, log sessions, rotate credentials, and avoid using it as a general-purpose server.

## 18.2 AWS Systems Manager Session Manager

Session Manager can provide shell access to managed instances without opening inbound SSH/RDP ports or maintaining a bastion host.

Requirements include:

- SSM Agent and supported instance configuration.
- IAM permissions for the operator and instance.
- Network access from the instance to the required Systems Manager endpoints, either through internet/NAT or suitable VPC interface endpoints.
- Logging configuration if session records must be retained.

Session Manager is often preferred, but it is not “zero networking”: the managed node still needs outbound connectivity to the Systems Manager service endpoints.

---

# 19. Network Observability and Troubleshooting

## 19.1 VPC Flow Logs

VPC Flow Logs capture metadata about IP traffic to and from network interfaces. They can be created at VPC, subnet, or ENI scope.

Destinations include:

- CloudWatch Logs.
- Amazon S3.
- Amazon Data Firehose.

Typical record fields include source/destination addresses and ports, protocol, packet/byte counts, timestamps, and action (`ACCEPT` or `REJECT`).

### What Flow Logs are good for

- Identifying accepted or rejected flows.
- Investigating source/destination IPs and ports.
- Detecting unexpected traffic patterns.
- Feeding data into Athena or a SIEM pipeline when stored in S3.

### What Flow Logs do not provide

- They do not capture packet payloads.
- They are not a packet capture tool.
- They are not a real-time packet-by-packet stream.
- Some traffic is excluded from Flow Logs, including traffic to the Amazon DNS resolver, DHCP, instance metadata, and other documented exceptions.
- A `REJECT` record does not by itself identify exactly which security control caused the rejection.

## 19.2 Reachability Analyzer

VPC Reachability Analyzer performs configuration-based analysis of network reachability between supported source and destination resources. It evaluates network configuration such as routes and security controls and can identify a blocking component.

It is not a packet capture and does not prove that an application process is healthy or listening.

## 19.3 Network Access Analyzer

Network Access Analyzer helps identify network paths that match specified access requirements, such as paths that expose resources to the internet or violate intended segmentation.

Use it for network security posture and architecture validation.

## 19.4 CloudTrail

CloudTrail records AWS API activity. Use it to investigate who changed a route table, security group, subnet, VPC endpoint, or other networking resource and when the change occurred.

Flow Logs answer “what IP traffic was observed?” CloudTrail answers “what AWS API action changed the configuration?”

## 19.5 VPC Traffic Mirroring

Traffic Mirroring copies traffic from supported network interfaces to monitoring appliances for deeper packet inspection. It is distinct from Flow Logs because it can provide packet-level visibility, subject to supported configurations and capture design.

## 19.6 A practical troubleshooting sequence

When an application cannot connect:

1. **DNS:** Does the hostname resolve to the expected IP?
2. **Address:** Is the destination IP correct and reachable from this network?
3. **Route:** Does the source subnet route table contain a matching route?
4. **Return route:** Can the destination send traffic back?
5. **Security group:** Does the source permit egress and destination permit ingress?
6. **NACL:** Are both directions permitted?
7. **Gateway/attachment:** Is IGW, NAT, endpoint, peering, TGW, VPN, or DX configured and healthy?
8. **Host:** Is the OS firewall allowing the port?
9. **Application:** Is the service listening on the correct interface and port?
10. **TLS/application:** Are certificates, protocols, authentication, and authorization correct?

---

# 20. Advanced Network Security and Governance

## 20.1 AWS Network Firewall

AWS Network Firewall is a managed, stateful network firewall service for VPC traffic inspection. It supports rule groups and centralized inspection designs.

Common use cases:

- Domain/IP/protocol filtering.
- Threat-signature inspection.
- Egress control.
- Centralized inspection across multiple VPCs.

A firewall deployment requires careful routing so that traffic actually traverses the firewall endpoints. Merely creating a firewall does not inspect traffic automatically.

## 20.2 AWS Firewall Manager

Firewall Manager helps centrally configure and manage supported firewall and security policies across accounts and resources in AWS Organizations. It is a governance layer, not a packet-forwarding device.

## 20.3 AWS IP Address Manager (IPAM)

VPC IPAM helps plan, track, monitor, and manage IP address space across accounts and Regions. It is useful for preventing overlapping CIDRs and maintaining allocation standards.

## 20.4 Managed prefix lists

A prefix list is a named set of CIDR blocks that can be referenced in route tables and security group rules. AWS-managed prefix lists represent AWS-maintained ranges for supported services; customer-managed prefix lists represent ranges maintained by your organization.

They reduce repeated CIDR entry and simplify policy changes.

## 20.5 VPC sharing with AWS RAM

VPC sharing allows a VPC owner account to share subnets with participant accounts using AWS Resource Access Manager (RAM), typically within an AWS Organization.

- The VPC owner controls the VPC and shared subnet network configuration.
- Participants can deploy supported resources into shared subnets.
- Ownership and permissions are split: participants manage their resources; the owner manages shared network infrastructure.
- Sharing a subnet is not the same as peering two VPCs.

## 20.6 AWS PrivateLink versus VPC Lattice

- **PrivateLink:** Private connectivity to a specific service endpoint; consumers do not need general network access to the provider VPC.
- **VPC Lattice:** Application-layer service networking for connecting, securing, and monitoring services across VPCs and accounts, with service discovery and policy features.

Choose based on whether the requirement is private endpoint connectivity or broader application service-to-service networking.

---

# 21. High Availability, Resilience, and Cost Design

## 21.1 Multi-AZ baseline

A common production VPC layout includes:

- At least two AZs, and often three where service support and workload requirements justify it.
- Public and private subnet tiers in each AZ.
- Load balancer targets distributed across AZs.
- One NAT Gateway per AZ when private IPv4 egress must survive an AZ failure independently.
- Multi-AZ database deployment where supported and required.
- Interface endpoints deployed across the AZs needed by the workload.
- Monitoring and tested failover procedures.

Do not assume “resources in two AZs” automatically means the application is highly available. Application state, load balancing, dependencies, routing, and failover behavior all matter.

## 21.2 Cost factors

Important networking cost drivers include:

- NAT Gateway hourly and data-processing charges.
- Cross-AZ data transfer.
- Public IPv4 address charges.
- Interface endpoint hourly and data-processing charges.
- Transit Gateway attachment and data-processing charges.
- Site-to-Site VPN and Direct Connect port/data charges.
- Data transfer out to the internet or other Regions.
- Firewall and traffic inspection processing.
- Flow Logs storage and query costs.

## 21.3 Cost optimization examples

- Use S3 and DynamoDB gateway endpoints for in-VPC access where suitable.
- Keep workloads close to their AZ-local NAT Gateway and data dependencies where possible.
- Avoid unnecessary cross-AZ data paths.
- Review whether an interface endpoint's fixed and processing costs are justified by traffic volume and security requirements.
- Remove unused NAT Gateways, endpoints, EIPs, and TGW attachments.
- Use centralized egress only when its security and operational benefits justify the routing complexity and data-transfer costs.
- Monitor NAT bytes, endpoint usage, and cross-AZ traffic.

## 21.4 Resilience checklist

- No single NAT Gateway should be an unnoticed dependency for all AZs.
- Route tables should be documented and reviewed.
- Hybrid connectivity should have redundant tunnels/links and tested route failover.
- DNS resolver endpoints should be deployed with availability in mind.
- Security policy changes should be audited.
- Capacity and IP address headroom should be monitored.
- Run failure simulations and verify that traffic takes the intended alternate path.

---

# 22. Reference Architectures

## 22.1 Highly available three-tier web application

```text
                         Internet
                            |
                           IGW
                            |
                +-----------+-----------+
                |                       |
           Public AZ-a             Public AZ-b
             ALB node                ALB node
                |                       |
        +-------+-------+       +-------+-------+
        |               |       |               |
   Private App-a    Private App-b (application targets)
        |               |
   Private DB-a     Private DB-b (Multi-AZ database pattern)
```

Supporting design:

- ALB spans public subnets in multiple AZs.
- App targets live in private subnets.
- App SG allows traffic only from ALB SG.
- DB SG allows traffic only from app SG.
- NAT Gateway per AZ provides outbound IPv4 where required.
- S3 gateway endpoint provides private S3 access.
- Flow Logs and CloudTrail provide network and configuration visibility.

## 22.2 Private data engineering platform

A typical data platform may contain:

- Private ingestion workers (EC2, ECS, EKS, or managed service ENIs where applicable).
- Private orchestration and transformation services.
- Private database and metadata services.
- S3 gateway endpoint for object storage access.
- Interface endpoints for required service APIs such as ECR, CloudWatch Logs, KMS, STS, and Systems Manager, subject to service support and application needs.
- NAT Gateway only for destinations that genuinely require public IPv4 egress.
- Separate data and application subnets where distinct routing or security boundaries are useful.

Remember: an S3 endpoint solves the network path to S3. It does not grant S3 permissions. IAM, bucket policy, endpoint policy, KMS key policy, and organization controls still apply.

## 22.3 Multi-account hub-and-spoke network

```text
                   On-premises network
                           |
                      VPN / DX
                           |
                    Transit Gateway
                    /      |      \
                   /       |       \
              Prod VPC   Dev VPC   Shared Services VPC
```

Design considerations:

- Use TGW route tables to segment environments.
- Propagate only intended routes.
- Use centralized inspection or egress only with deliberate route design.
- Use RAM to share supported networking resources across accounts.
- Use IPAM to allocate non-overlapping CIDRs.
- Use CloudTrail, Flow Logs, and centralized security monitoring.

## 22.4 Private service sharing between organizations

```text
Consumer VPC
    |
Interface Endpoint (private IP)
    |
AWS PrivateLink
    |
Provider Endpoint Service
    |
Network Load Balancer
    |
Provider application
```

The consumer receives access to a service endpoint rather than general routing into the provider VPC.

---

# 23. Practical Troubleshooting Runbook

## Scenario 1: EC2 in a public subnet cannot access the internet

Check in this order:

1. Does the instance have a public IPv4 address or EIP?
2. Does the subnet's associated route table contain `0.0.0.0/0 -> IGW`?
3. Is the IGW attached to the correct VPC?
4. Does the SG allow outbound traffic?
5. Does the NACL allow outbound and response traffic?
6. Does the host firewall allow the connection?
7. Does DNS resolve the destination correctly?

## Scenario 2: Private EC2 cannot download packages

1. Is the private subnet route `0.0.0.0/0 -> NAT Gateway` present?
2. Is the NAT Gateway in a public subnet?
3. Does the NAT subnet route `0.0.0.0/0 -> IGW`?
4. Is the NAT Gateway available and associated with an Elastic IP if it is public?
5. Are the source and destination NACL rules correct?
6. Does the SG permit egress?
7. Is the package repository reachable and resolving correctly?
8. Is the workload accidentally using an IPv6 destination without an IPv6 egress route?

## Scenario 3: EC2 cannot connect to RDS

1. Are both resources in networks with a route between them?
2. Does the RDS SG allow the DB port from the application SG?
3. Does the application SG permit egress to the DB port?
4. Do NACLs permit both directions?
5. Is the database endpoint DNS resolving correctly?
6. Is the DB available and listening?
7. Are credentials, TLS, and database authorization correct?

## Scenario 4: Two VPCs cannot communicate

1. Is the peering connection active or is the TGW attachment available?
2. Are route tables updated on both sides?
3. For TGW, are attachment associations and route propagation/static routes correct?
4. Are CIDRs overlapping?
5. Do SGs and NACLs permit the traffic?
6. Is the return route present?
7. Is DNS resolution across VPCs configured if using hostnames?

## Scenario 5: Application cannot access S3 privately

1. Is an S3 gateway endpoint associated with the route table used by the workload?
2. Is the endpoint in the correct Region?
3. Does the endpoint policy permit the action/resource?
4. Do IAM and bucket policies permit the request?
5. Does the bucket policy restrict access to a different VPC endpoint?
6. If using an interface endpoint, are endpoint ENIs, SGs, private DNS, and routing configured?
7. If SSE-KMS is used, do the relevant KMS permissions and key policy permit access?

## Scenario 6: DNS name resolves to an unexpected address

1. Is the name in a public or private hosted zone?
2. Is the private hosted zone associated with the VPC?
3. Is a Resolver rule forwarding the domain elsewhere?
4. Is Private DNS enabled on the interface endpoint?
5. Is the application using a custom DNS server or cached result?
6. Is the query being made from the expected VPC/account/Region?

## Scenario 7: Intermittent connectivity across AZs

Investigate:

- A single-AZ NAT Gateway dependency.
- Cross-AZ routes or centralized egress dependencies.
- Load balancer target health and AZ enablement.
- Endpoint ENI placement and endpoint AZ availability.
- Route propagation and failover behavior.
- DNS answers and client connection reuse.
- Application retries, timeouts, and connection pool behavior.

---

# 24. Common Misconceptions and Corrections

| Misconception | Correct understanding |
|---|---|
| Attaching an IGW makes every subnet public | A subnet is public by routing; resources also need appropriate addressing and security rules |
| A public subnet automatically gives every EC2 a public IP | Public IPv4 assignment is separately controlled |
| A private subnet cannot communicate outside the VPC | It can use NAT, endpoints, VPN, TGW, or other configured routes |
| NAT Gateway allows unsolicited inbound connections | It allows return traffic for connections initiated from inside; it is not an inbound publishing mechanism |
| A NAT Gateway is Regional | It is deployed in one AZ |
| Security groups are subnet firewalls | They are associated with network interfaces/resources |
| NACLs are stateful | NACLs are stateless and require both directions |
| A route means traffic is allowed | Routing and authorization are separate |
| VPC peering is transitive | It is not transitive |
| Transit Gateway automatically connects all attached VPCs | Route table association, propagation/routes, and VPC routes determine reachability |
| PrivateLink connects two entire VPCs | It provides private access to a specific service |
| Gateway endpoints create ENIs | They use route-table entries |
| Interface endpoints are free | They generally have hourly and data-processing charges |
| S3 gateway endpoints are reachable from on-premises | Gateway endpoints are not directly reachable from on-premises; consider interface endpoints |
| IPv6 addresses are safe because they are hard to guess | IPv6 addresses can be globally routable; security controls remain essential |
| Direct Connect encrypts traffic by default | DX is not encrypted by default |
| Flow Logs show packet payloads | They record flow metadata, not packet contents |
| Flow Logs always identify the exact blocking rule | They show ACCEPT/REJECT metadata but do not directly name the responsible SG/NACL rule |
| Session Manager needs no network connectivity | Managed nodes need access to Systems Manager endpoints |
| An S3 endpoint grants S3 permissions | It only provides a network path; IAM and resource policies still control authorization |

---

# 25. Interview Questions and Answers

## Fundamentals

**1. What is a VPC?**  
A Regional, logically isolated virtual network in AWS where you define IP ranges, subnets, routing, and network security.

**2. What is the difference between a VPC and a subnet?**  
A VPC spans a Region; a subnet is a CIDR subdivision that belongs to one AZ.

**3. What makes a subnet public?**  
Its associated route table has a route to an Internet Gateway. A resource still needs appropriate public addressing and security configuration to be internet reachable.

**4. How many IPv4 addresses are usable in a `/24` subnet?**  
251, because AWS reserves five addresses in every IPv4 subnet.

**5. Why should VPC CIDRs not overlap?**  
Overlapping ranges complicate or prevent routing between networks and are incompatible with common peering/TGW designs.

## Routing and internet access

**6. What is the difference between an IGW and a NAT Gateway?**  
An IGW enables internet routing for eligible public resources. A NAT Gateway allows private IPv4 resources to initiate outbound connections without accepting unsolicited inbound connections.

**7. Can a private subnet access the internet?**  
Yes. A common IPv4 pattern is a default route to a public NAT Gateway in a public subnet.

**8. Does an IGW perform NAT?**  
For IPv4 public addressing, the IGW maps the instance's private IPv4 address to its associated public IPv4 address. IPv6 does not use this IPv4 mapping model.

**9. What is longest prefix match?**  
When multiple routes match a destination, the most specific matching CIDR route is selected.

**10. Why deploy one NAT Gateway per AZ?**  
To avoid a single-AZ NAT dependency and reduce cross-AZ data transfer for workloads in each AZ.

## Security

**11. Security group versus NACL?**  
SGs are stateful and attached to network interfaces/resources; NACLs are stateless and applied at subnet boundaries.

**12. Can a security group deny a specific IP?**  
No. SGs contain allow rules only. A NACL or firewall can provide explicit deny behavior.

**13. Why do NACLs need ephemeral port rules?**  
Because they are stateless; response packets use the client's ephemeral port and must be explicitly allowed.

**14. Do security groups replace application authentication?**  
No. SGs control network reachability, not user identity or application authorization.

## Connectivity

**15. VPC Peering versus Transit Gateway?**  
Peering is direct one-to-one connectivity without transitive routing. TGW is a hub for many networks and supports transitive routing when its route tables are configured accordingly.

**16. What is PrivateLink?**  
A private service connectivity mechanism that exposes a specific service through interface endpoints rather than providing full network connectivity to the provider VPC.

**17. Can two VPCs with overlapping CIDRs be peered?**  
No, overlapping IPv4 CIDRs prevent VPC peering.

**18. Does a TGW attachment automatically make two VPCs reachable?**  
No. VPC route tables, TGW route table association/propagation or static routes, return routes, and security controls must be configured.

**19. VPN versus Direct Connect?**  
VPN uses encrypted IPsec tunnels over the internet. Direct Connect provides dedicated/partner connectivity but is not encrypted by default.

## Endpoints and DNS

**20. How can a private EC2 instance access S3 without NAT?**  
Use an S3 gateway endpoint associated with the subnet's route table and configure the required authorization policies.

**21. Gateway endpoint versus interface endpoint?**  
Gateway endpoints are route-table targets for S3 and DynamoDB. Interface endpoints create ENIs with private IPs and use AWS PrivateLink for supported services.

**22. What is the VPC DNS resolver address?**  
The primary VPC IPv4 CIDR plus two, and the link-local resolver address `169.254.169.253`. It is not a separate `.2` address per subnet.

**23. What is the difference between `enableDnsSupport` and `enableDnsHostnames`?**  
The first enables Amazon-provided DNS resolution. The second enables DNS hostnames for resources where applicable.

## Operations and architecture

**24. How would you troubleshoot an EC2 connectivity issue?**  
Check DNS, IP addressing, route tables, return path, SGs, NACLs, gateway/endpoint health, host firewall, and application listener.

**25. What do VPC Flow Logs show?**  
Metadata about IP flows, including addresses, ports, protocol, and ACCEPT/REJECT action. They do not capture payloads.

**26. Flow Logs versus CloudTrail?**  
Flow Logs record IP traffic metadata. CloudTrail records AWS API activity and configuration changes.

**27. What is Reachability Analyzer?**  
A configuration analysis tool that evaluates whether a path is possible between supported network resources and identifies blocking configuration.

**28. How would you design a highly available VPC?**  
Use multiple AZs, spread load balancer targets and application capacity, deploy AZ-local NAT Gateways where needed, use Multi-AZ data services, provide resilient DNS/endpoints, and test failover.

**29. How would you design networking for a private data platform?**  
Use private application and data subnets, S3/DynamoDB gateway endpoints where applicable, interface endpoints for required service APIs, controlled NAT egress for external dependencies, least-privilege SGs, and Flow Logs.

**30. How do you reduce NAT Gateway costs?**  
Use gateway endpoints for S3/DynamoDB, consider interface endpoints for frequent AWS API traffic, avoid unnecessary cross-AZ traffic, and monitor NAT data processing.
