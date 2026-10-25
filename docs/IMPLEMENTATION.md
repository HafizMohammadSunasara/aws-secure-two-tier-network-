# Implementation and Reproduction Guide

> **Purpose:** Rebuild a comparable educational network safely, while distinguishing this repeatable recipe from the original historical evidence. These steps were **not** extracted as an executed command transcript. Do not paste placeholders into AWS as though they were real resource IDs.

## 0. Boundaries, prerequisites, and cost

- Use a disposable AWS lab account and confirm the desired region (historical screenshots: Sydney `ap-southeast-2`).
- Review current pricing for EC2, public IPv4 addresses, NAT services, and outbound traffic. Stopped EC2 instances may still leave billable storage or IP resources.
- Use IAM least privilege and your organization's network policy. Avoid public RDP/SSH exposure when AWS Systems Manager Session Manager is suitable.
- Record actual VPC, subnet, route table, EC2 and security-group identifiers in a **private** worksheet. Do not publish credentials.

## 1. Plan CIDR ranges

Before creating any subnet, choose a VPC range big enough to contain all subnets. CIDRs must not overlap. E.g. **illustrative only**:

```text
VPC       10.40.0.0/16
Public    10.40.1.0/24
Private   10.40.2.0/24
```

**Historical difference:** The captured project shows `172.31.0.0/20` for the selected private subnet. Don't present the sample `10.40` network as the user's historical configuration.

## 2. Create VPC and subnets (console)

1. VPC Console → Create VPC → VPC only → chosen IPv4 CIDR.
2. Create `Public-Subnet` and `Privet-Subnet` (or correctly spelled `Private-Subnet` in a new lab) in that VPC.
3. Choose AZ deliberately; splitting tiers into subnets **does not automatically** create multi-AZ HA.
4. Verify subnet `Available` state and record subnet IDs/CIDRs.
5. Verify subnet auto-assign public IP policy and specific resources' public-address assignment separately.

## 3. Make intended route associations explicit

1. Create `Public-Subnet-RT`; associate it with the public subnet.
2. Create `Privet-Subnet-RT`; associate it with the private subnet.
3. Inspect the **Routes** tab for each table. VPC-local routes should be present.
4. Only if public internet reachability is intended: attach an IGW and create public `0.0.0.0/0 → IGW` route.
5. If private outbound IPv4 is needed: choose managed NAT gateway or correctly configured NAT instance, then create private outbound default route to the corresponding supported NAT target.
6. Verify route status as Active and check for unintentional more-specific routes.

**Evidence currently available supports Steps 1–2 association as configured, but not the exact destinations/targets in Steps 4–5.**

## 4. Configure security groups for least privilege

An example policy (adjust to your actual workload):

| Security group | Inbound | Outbound |
|---|---|---|
| Public app | TCP 443 only from authorized client range or 0.0.0.0/0 for intentionally public HTTPS | App-dependent; private DB TCP port to backend SG |
| Backend DB | Required DB port from public-app SG only (e.g. PostgreSQL 5432 or MySQL 3306, not both by default) | Only required destinations |
| NAT instance | Traffic from private subnet for forwarding and explicitly required management; check AWS NAT-instance guide | Internet-bound traffic as required |

Security groups are **stateful**. Also inspect subnet NACLs (stateless), host firewalls and service listener ports. The historical NAT screenshot shows at least two security-group entries; its complete policy cannot be reconstructed from one narrow view.

## 5. NAT-instance learning path (optional)

> AWS recommends NAT gateways for many production workloads because they offer managed availability/performance and require less maintenance. A self-managed NAT instance is still useful for studying packet forwarding and its operational liabilities.

For an EC2 NAT instance, verify ALL of the following from the AWS guide:

- Selected AMI / OS is appropriate and supported; use a modern, patched OS.
- The NAT instance is **Running** and inside a public subnet with a valid route to IGW and public IPv4/EIP as appropriate.
- Disable source/destination checks on the NAT network interface as required.
- Enable operating-system IPv4 forwarding and configure proper SNAT/MASQUERADE rules on the intended interface.
- Configure ingress from the private subnet and necessary return/outbound rules.
- Private route table default route points to correct NAT target and is active.
- Monitor health; a single NAT instance fails when stopped or unhealthy.

**Historical fact:** The supplied NAT instance screenshot shows **Stopped**. There is no verified packet-forwarding outcome.

## 6. Safely observe AWS configuration with CLI (read-only examples)

Commands below require an authorized AWS CLI session. Replace placeholders with your actual IDs or omit filters as appropriate. These are **suggested future checks**, not claims of past execution.

```bash
aws sts get-caller-identity
aws ec2 describe-vpcs --region ap-southeast-2 --output table
aws ec2 describe-subnets --region ap-southeast-2 \
  --filters Name=vpc-id,Values=vpc-REPLACE_ME \
  --query 'Subnets[].{Id:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,PublicIP:MapPublicIpOnLaunch}' --output table
aws ec2 describe-route-tables --region ap-southeast-2 \
  --filters Name=vpc-id,Values=vpc-REPLACE_ME \
  --query 'RouteTables[].{Id:RouteTableId,Routes:Routes,Associations:Associations}' --output json
aws ec2 describe-instances --region ap-southeast-2 \
  --filters Name=tag:Name,Values=Nat-Instance \
  --query 'Reservations[].Instances[].{State:State.Name,AZ:Placement.AvailabilityZone,Subnet:SubnetId,SourceDestCheck:SourceDestCheck}' --output table
```

Avoid placing AWS access keys in the repository. `aws sts get-caller-identity` returns identifying account details; share only redacted output.

## 7. Functional test plan — ONLY after environment is safe

**Positive:** From a private workload with no public IP, run a simple HTTPS/DNS test (`curl -I https://example.com`) and capture the current route/NAT state. A successful response demonstrates that endpoint can reach that destination **at that moment**, not universal network security.

**Negative:** From an unapproved external source, verify a backend service is unreachable. Do not interpret a timeout alone as formal security proof; correlate with route, SG, NACL and application status.

**Failure:** In a disposable lab, stop NAT and measure change in previously working private egress; restore it according to documented rollback steps. Never intentionally interrupt shared production infrastructure for a demonstration.

## 8. Cleanup

First capture final resource IDs and states, then remove dependent routes, detach/delete test instances and addresses, remove custom networking resources, and confirm no billable items remain. Do **not** run destructive cleanup commands blindly on resources you do not own.
