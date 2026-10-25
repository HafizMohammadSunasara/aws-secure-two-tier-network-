# Architecture — Secure Two-Tier AWS VPC

![Intended topology and known unknowns](../diagrams/architecture.png)

## 1. Intent and observation are separate categories

**Observed:** Sydney region AWS console; `Public-Subnet`; `Privet-Subnet`; two route tables associated with respective subnets; EC2 resource named `Nat-Instance`, **Stopped**. Selected private subnet CIDR is `172.31.0.0/20`.

**Designed/illustrative:** internet gateway and inbound web path; app instance in public tier, database/private server in backend tier; routing from private tier to a NAT resource. None of those are fully demonstrated by current screenshots. This is why all intended paths in the architecture diagram are dashed.

## 2. Trust boundaries

| Boundary | Desired permission model | Risk if misconfigured |
|---|---|---|
| Internet → app tier | Only explicitly needed public listener ports; TLS where appropriate | Unnecessary remote administration/ports visible |
| App → backend | App security-group identity / exact backend port | Any host in broad CIDR can reach sensitive service |
| Backend → internet | Outbound initiated through NAT only when needed | Public exposure, dependency on single NAT instance |
| Administrator → cloud resources | IAM + secure remote-access approach, ideally SSM | Public management port attack surface |

## 3. Route table model

A route table contains `(destination CIDR, target)` entries. AWS uses longest-prefix match. A subnet is public **because a route to an internet gateway exists**, not because it is named Public-Subnet. A private subnet typically has no direct IGW default route. It might have a private egress route via NAT, or be completely isolated.

**Illustrative, NOT extracted from retained screenshots:**

```text
Public subnet route table
  VPC-CIDR       local
  0.0.0.0/0      igw-<real-id>

Private subnet route table
  VPC-CIDR       local
  0.0.0.0/0      nat-<real-id>  # managed NAT gateway example
                OR eni-<real-id> / instance routing target where supported
```

Always inspect actual AWS route targets, status and resource state instead of copying a placeholder. If the private tier needs no internet, omit the NAT default route entirely.

## 4. Packet walkthrough A: external client to public app (target design)

1. DNS resolves public application name to a public frontend endpoint.
2. Internet traffic reaches an Internet Gateway associated with the VPC.
3. Public route table and public addressing let an allowed frontend resource receive traffic.
4. Frontend SG admits only its configured listener port; host firewall/application must also allow it.
5. Application responds; established return traffic follows appropriate route and stateful SG behavior.

**Not verified by supplied screenshots:** DNS, IGW attachment/routes, server instance, listener, public IP, application response and TLS.

## 5. Packet walkthrough B: app to private database (target design)

1. App resolves backend private address (or private DNS).
2. Local VPC routing delivers the packet inside the VPC CIDR.
3. Database SG allows a specific port from an application SG, not from the whole internet.
4. Database process listens only on appropriate internal interface/address.
5. Return is allowed for an established connection by stateful security groups, subject to OS and NACL behavior.

**Not verified:** database EC2/RDS instance, SG-to-SG reference, listener process or real application query.

## 6. Packet walkthrough C: private subnet egress through NAT (target design)

1. Private host initiates HTTPS to an external repository.
2. Its subnet route table points outbound IPv4 traffic at the NAT resource.
3. The NAT resource must be running/healthy, have correct routing and translation, and (if an EC2 NAT instance) satisfy source/destination-check, IP-forwarding and firewall requirements.
4. NAT replaces source address/port, forwards through a public route via IGW and receives the response.
5. Translation state returns the reply to the initiating private host.

A NAT is **not** a way for arbitrary internet clients to initiate new connections to private instances. A stopped NAT instance cannot forward packets.

## 7. Availability limitations

A single NAT instance is a single failure domain unless failover and route reconfiguration are deliberately engineered. A single subnet in one AZ is not a multi-AZ resilient architecture. For production egress, consider AWS managed NAT gateways **per AZ** or appropriate VPC endpoints, and assess costs and failure modes; don't assume a NAT gateway in one AZ protects others from AZ failure.

## 8. Regional facts

All three historical AWS console screenshots show the **Sydney** (`ap-southeast-2`) region. The NAT EC2 screenshot lists `ap-southeast-2b`. The private subnet is shown as `172.31.0.0/20`; **that is a subnet CIDR, not documented evidence of the whole VPC CIDR**. Public-subnet CIDR and complete route entries are not visible.
