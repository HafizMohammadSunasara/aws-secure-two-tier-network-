# Troubleshooting — Diagnosing Two-Tier AWS VPC Networks

## Fast investigation sequence

**Symptom → instance health → addressing → routes → SG/NACL → OS firewall/service → NAT/DNS → test and record.** Avoid changing multiple controls at once.

### A. "The route table is associated, so why can't private instances reach the internet?"

A route-table association only selects which route table a subnet uses. Check the **Routes** tab for `0.0.0.0/0` target, status, NAT health and correct AZ placement. If using NAT instance, inspect source/destination check, Linux forwarding (`net.ipv4.ip_forward`), translation rules and public subnet IGW route. DNS resolution problems can masquerade as route problems.

### B. "My NAT instance is Stopped"

This is the direct state shown in historical evidence. A stopped EC2 instance is not actively translating or forwarding packets. Find out whether stop was intentional. Check route target, maintenance status and whether private outbound path has redundancy. Do not auto-start a potentially billable instance without project-owner approval.

### C. "Backend server has a private address, but is it actually private?"

Private RFC1918 addressing alone isn't an authorization control. Check subnet default route (no direct IGW path), public IP assignment, inbound SG, stateless NACL, host firewall, network endpoints and intended services. Validate one **allowed** and one **denied** path.

### D. "Public application can't reach private DB"

Validate the backend is **running** and listening on a private interface; confirm app DNS/address, DB port, app SG egress, DB SG ingress source referencing app SG, local VPC route and NACL ephemeral port return. An unreachable DB is not automatically proof that subnet isolation is correct.

### E. "Route table lists a target but traffic still fails"

Check route status (active/blackhole), actual target resource state, route specificity, subnet association, OS routing, security groups and return path. More specific routes can override an intuitive default route.

### F. "I opened too many SG ports to fix a connection"

Undo exploratory broad rules as soon as possible. During diagnosis, capture exact five-tuple: source IP/SG, source port, destination IP, destination port and protocol. Confirm the intended app path using narrow rules rather than `0.0.0.0/0` everywhere. Use Session Manager when possible instead of public SSH/RDP.

### G. "NAT is running but still not forwarding"

For self-managed NAT instances:

- AWS source/destination checks may still be enabled.
- OS forwarding might be disabled.
- `iptables`/`nftables` SNAT/MASQUERADE may target wrong interface.
- Security-group, NACL or instance firewall may block forwarding/return.
- Public route to IGW or public EIP may be missing.
- Private default route may target the wrong interface/resource or be blackholed.
- Ephemeral response ports or DNS may be filtered.

AWS's official [NAT instance guide](https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html) should be your source for current supported procedure.

## Evidence-first troubleshooting worksheet

```text
Date/time (UTC):
Region / VPC / subnet (redacted as needed):
Expected flow: source -> destination:port/protocol
Actual symptom:
Instance and route target state:
Route-table association and matched route:
SG / NACL / host firewall observations:
Packet/DNS/HTTP test result:
Single change made:
Retest result:
Rollback applied:
```

## Security of diagnostics

Logs and command outputs may contain account IDs, addresses, tokens or connection strings. Redact confidential fields. Never publish access keys, private key files or databases in the repository.
