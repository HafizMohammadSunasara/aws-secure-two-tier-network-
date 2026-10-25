# Deep Dive — Secure Two-Tier AWS Networking, from First Principles to Engineering Judgement

**Learning handbook:** fundamentals → packet journey → operational security → verification → interview confidence  
**Companion case study:** [README.md](README.md)  
**Actual screenshots:** [Evidence index](docs/EVIDENCE-INDEX.md)

> If you remember only one idea: **Network security is a statement about which packets can cross which trust boundaries, not a collection of reassuring resource names.** “Private subnet,” “NAT,” and “security group” are not final outcomes until you can show their precise behavior.

## Learning map

1. Why a two-tier network exists
2. IP packets, ports, DNS and five-tuples
3. VPC as an isolation boundary
4. CIDR from zero, with worked calculations
5. Subnets, AZs, and the meaning of public vs private
6. AWS route tables and longest-prefix match
7. Internet gateways and public IPs
8. Network address translation (NAT/PAT) deeply explained
9. NAT instance versus NAT gateway
10. Source/destination checks and IP forwarding
11. Application and database trust boundaries
12. Security groups versus network ACLs versus host firewalls
13. Data-plane testing: allowed and denied paths
14. DNS, VPC endpoints, and minimizing egress
15. Access, administration and secrets
16. Operational reliability, AZ failure and monitoring
17. Costs and common hidden charges
18. Troubleshooting from symptoms to packet paths
19. What the three historical screenshots really prove
20. Practical exercises and checklists
21. Real-world failure scenarios
22. Interview questions and answers
23. Production-readiness review and engineering recap

---

## 1. Why separate public and private tiers?

Imagine a small online shop. Visitors need to access a web server. The web server needs to access a database. The database does **not** need direct connections from every internet user. Putting everything on one publicly reachable machine can work for a demo, but it expands the attack surface: a mistake in one public-facing rule can expose sensitive data infrastructure.

In a two-tier design, think about **who initiates each conversation**:

```text
External client  --HTTPS-->  public application
Public application  --DB protocol-->  private backend
Private backend  --optional outbound HTTPS-->  software repository
External client  --X-->  private backend directly
```

The `--X-->` is as important as the successful arrows. A secure design specifies forbidden access explicitly. It also distinguishes a frontend being in a public subnet from a frontend being safe: the frontend still needs correct patching, IAM, configuration, authentication, application security and logs.

**Analogy:** A hospital has a public reception desk and restricted records room. Being in the same building doesn't mean every visitor should enter both. The reception desk is a *controlled* gateway to permitted actions; the records room has separate access policy.

**Engineering question:** Can you write one sentence describing why an internet browser needs HTTPS to the app, and a separate sentence describing why the same browser should not talk directly to PostgreSQL port 5432?

## 2. What moves through a network: packets and connections

Before memorizing VPC menu items, learn what a network flow actually contains. A typical TCP connection is identified using five values: source IP, source port, destination IP, destination port, protocol. Example:

```text
Source:      10.40.1.22:49152
Destination: 10.40.2.40:5432
Protocol:    TCP
Purpose:     application -> PostgreSQL
```

This is an *illustrative flow*, not a value extracted from the user's AWS deployment. A security-group rule needs to match actual intended traffic. A vague claim such as “the backend is secure” is not measurable. A testable claim is: “Only application instances in security group A can establish TCP connections to the database listener on port 5432; direct internet connections are blocked.”

Know the layers of a successful request:

1. **Name lookup (DNS):** Convert hostname to an IP; this can fail independently.
2. **Routing:** Match destination address to a route table target.
3. **Filtering:** Security groups, NACLs and host firewalls may allow/deny.
4. **Transport:** TCP handshake or UDP messages reach a listening process.
5. **Application:** HTTP status or SQL query succeeds; success here proves more than merely seeing a ping reply.
6. **Return path:** Response travels back through compatible routing and firewall state.

`ping` (ICMP) helps test one class of reachability, but an allowed ping does not establish that a database port is reachable. Equally, blocked ping does not mean HTTPS is broken.

## 3. VPC: the network boundary, not just a checkbox

An Amazon Virtual Private Cloud provides a logically isolated network address space and networking constructs: subnets, route tables, gateways, security groups and more. It is not a replacement for operating-system or application security. It determines which network paths exist; IAM controls who can change the infrastructure. These are different control planes.

AWS resources also have **regional** context. In the provided screenshots, the VPC console and NAT instance are in Sydney (`ap-southeast-2`). One screenshot shows an EC2 availability zone `ap-southeast-2b`. That proves the EC2 resource's shown AZ, not that the entire design is highly available across AZs.

**Control plane vs data plane:** AWS console lets you configure a route and see a resource state. That is *control-plane* evidence. Real TCP/HTTP requests moving through the route are *data-plane* evidence. Good engineering verifies both.

## 4. CIDR math without fear

CIDR notation combines an IP address and a prefix length. A prefix `/20` means the first 20 IPv4 bits identify the network; the rest identify addresses within it.

The selected **private subnet** in the actual screenshot has CIDR **`172.31.0.0/20`**.

- IPv4 addresses are 32 bits total.
- `/20` leaves `32 - 20 = 12` host bits.
- The block contains `2^12 = 4096` IPv4 addresses.
- The corresponding address range is `172.31.0.0` through `172.31.15.255`.
- AWS reserves five IPv4 addresses in every subnet, leaving **4091** addresses theoretically available for resources, before other allocations or restrictions.

Do **not** treat the `/20` screenshot as proof of the VPC's full CIDR block. A subnet must be contained within a VPC CIDR, but that screenshot only gives the selected subnet's IPv4 range.

A separate *design exercise* could use:

```text
VPC              10.40.0.0/16
Public subnet    10.40.1.0/24
Private subnet   10.40.2.0/24
```

These sample ranges deliberately differ from the historical deployment; do not mix them in evidence captions. For troubleshooting, draw the exact address ranges and confirm no overlap. A missing or wrong network match can defeat a perfect security group rule.

## 5. Public and private subnets are routing concepts

A common beginner mistake is believing a subnet is private because somebody named it `Private-Subnet`. AWS does not enforce security based on the string in the **Name** tag. Route tables and network settings matter.

A subnet with a route to an **Internet Gateway** is typically considered public. A subnet without that IGW route is private. Private doesn't necessarily mean no outbound access: its route may point to a NAT device, or to a VPC endpoint for selected AWS services. Some private subnets are completely isolated.

An internet-routable EC2 endpoint additionally needs suitable IP addressing, security group/NACL permissions and service listeners. A public subnet does not automatically expose every EC2 instance. A private subnet does not automatically block every malicious communication from inside the VPC; lateral movement remains a concern.

**In the user's actual screenshots:** both a `Public-Subnet` and `Privet-Subnet` are listed and `Available`. That substantiates resources and labels, not route semantics. The route-list screenshots show **table associations**, not route entries. Therefore we cannot honestly declare the public subnet has an IGW route or the private subnet has a working NAT route based on these three images alone.

## 6. How AWS route tables decide where to send packets

Think of a route table as a set of directional instructions keyed by destination IP. It does not authorize the packet—that is a separate security task.

```text
Destination          Target
10.40.0.0/16         local
0.0.0.0/0            igw-example
```

The route to `10.40.0.0/16` means traffic for addresses inside the VPC remains locally routed. The default route `0.0.0.0/0` handles addresses without a more specific matching route. AWS uses **longest-prefix match**; `10.40.2.0/24` is more specific than `10.40.0.0/16`.

Every subnet must be associated with one effective route table. It may be the VPC's main table implicitly or another table explicitly. An explicit association provides clarity when reviewing architecture. In the provided screenshot, `Public-Subnet-RT` is explicitly associated with `Public-Subnet`; `Privet-Subnet-RT` is explicitly associated with `Privet-Subnet`. That is legitimate visible proof of *association*. However, the target of a default route remains unknown until you open the **Routes** tab.

Debugging technique: for a packet leaving the private host, write its destination IP; then identify the source instance's subnet; then the subnet's associated route table; then the **single most specific** route that matches; then that route target's current state. This removes guesswork.

## 7. Internet Gateways: connectivity is a chain

An Internet Gateway is an AWS networking component attached to a VPC. A typical public IPv4 flow requires:

1. An IGW attached to the VPC.
2. An outbound route from the public subnet to the IGW.
3. Public IPv4/EIP mapping for the public-facing instance, if direct EC2 internet access is intended.
4. Security rules that permit traffic.
5. A running instance and listening application.

If any link is absent, “the subnet is public” doesn't equal “the app is reachable.” For IPv6, routing and public address behavior differ; treat IPv6 as a separate design/security review and do not blindly copy IPv4-only NAT ideas.

## 8. NAT: outbound egress without unsolicited inbound initiation

**Network Address Translation** (commonly NAT; often PAT as well) lets many internal endpoints share an external IPv4 address for their outbound connections. An internal host initiates a flow, the NAT device rewrites source address/port, tracks the translation and maps returning packets back to the correct origin.

Example conceptually:

```text
Private server 10.40.2.40:54321
    -> NAT private-side address:60001
    -> internet destination 93.184.215.14:443

Reply: internet destination -> NAT mapping -> 10.40.2.40:54321
```

These are teaching addresses/ports; the actual public IP and ephemeral port will vary. The key property is **connection initiation**. NAT helps a private machine fetch updates; it does not create a path for arbitrary external clients to initiate database connections. NAT is not a security policy replacement. Security group rules and routing must still reflect intended traffic.

Private subnet outbound request can fail for many reasons: default route missing, NAT unhealthy, NAT itself lacking a public path, source/destination check, OS translation disabled, SG/NACL, DNS failure, or destination service refusing traffic. Distinguish these with small, reproducible tests.

## 9. NAT instance versus NAT gateway: a practical trade-off

A **NAT instance** is an EC2 server you configure and operate to forward traffic. A **managed NAT gateway** is provided by AWS and removes much of the VM maintenance burden. AWS documentation recommends NAT gateways for many production designs because of availability, scalability and operational ease.

| Question | EC2 NAT instance | Managed NAT gateway |
|---|---|---|
| Who patches the OS? | You | AWS manages underlying platform |
| How is forwarding configured? | Linux forwarding + NAT rules + AWS routing | Route to managed gateway |
| What if one resource fails? | You own failover, monitoring, replacement and route update | AWS offers managed redundancy within AZ, design per AZ for resilience |
| Performance | Instance-type limits and software config | Managed service characteristics |
| Why use it in a lab? | Learn forwarding, instance state and security boundaries | Learn managed egress and reliability practices |
| Cost considerations | Instance hours, EBS, IP, transfer, ops work | Gateway hours, data processed, transfer/cross-AZ as applicable |

**Historical screenshot fact:** The resource named `Nat-Instance` is a `t3.micro` in `ap-southeast-2b` and shown as **Stopped**. That image supports neither operational NAT nor proof of dedicated subnet placement. Never describe a stopped machine as actively providing egress.

## 10. Why source/destination checks matter for NAT instances

AWS EC2 normally expects traffic sent or received by an instance's interface to have that instance as the intended source/destination. A router/NAT is different: it forwards packets whose final destination is another host. AWS source/destination checking must be configured appropriately for that routing role, together with a correct route target.

Inside the Linux NAT instance, the kernel must be willing to forward IPv4 packets; a common check is `sysctl net.ipv4.ip_forward`. The OS must also have appropriate translation/firewall rules, commonly implemented with `iptables` or `nftables`. Interface names can vary (`ens5`, etc.), so hard-coding someone else's interface command without inspection is risky.

Use the current AWS NAT-instance guide for exact supported configuration. The three retained screenshots do **not** show these settings. Their absence from evidence is not proof they were wrong or right: it means **unknown**. If you re-run the lab, record them properly.

## 11. The app/database trust boundary

A secure two-tier system should define **specific** allowed app-to-backend connections. Instead of a database security group that allows everyone in the internet (`0.0.0.0/0`) or perhaps even everyone in the whole VPC, prefer a rule with a precise application security-group source when topology permits.

Example intended policy:

```text
Frontend SG:
    inbound TCP 443 from legitimate clients
    outbound TCP 5432 to database SG

Database SG:
    inbound TCP 5432 from frontend SG only
    outbound only what is required
```

This is a **design pattern**, not evidence that this exact policy was deployed. Also note that in a full architecture the frontend is often behind a public load balancer while the app EC2 tier itself is private. Calling every public subnet application “production-ready” ignores attack surface and reliability considerations.

To verify a network security claim, first prove the application can talk to the database as expected, then prove a disallowed source cannot. Otherwise a closed port might simply mean the database process is stopped.

## 12. Security groups, NACLs and host firewalls are different controls

**Security Groups (SGs)** are stateful allowlists attached to ENIs/resources. If an established request is allowed, the response is normally allowed automatically by connection tracking. You cannot add an explicit SG deny rule.

**Network ACLs (NACLs)** operate at subnet boundaries with stateless allow/deny rules evaluated in order. Return traffic needs compatible inbound/outbound rules, including ephemeral ports where relevant. A NACL denying a return port can make a flow fail even while an SG appears permissive.

**Host firewall** (`nftables`, Windows Firewall etc.) is yet another filter inside the OS. The application must be running and listening. A correct cloud SG cannot force a stopped database to respond.

This layered mental model is crucial:

```text
Correct route? -> SG permits? -> NACL permits? -> host firewall permits?
    -> process listening? -> authentication/application responds?
```

Troubleshooting only the SG while ignoring the route can waste hours. Conversely, opening a broad SG port to “fix the route” creates risk without addressing the cause.

## 13. Data-plane verification: prove both positive and negative outcomes

For every security requirement, define a measurable check.

**Positive test:** “From app SG A, can I connect to database private IP on TCP 5432 and complete an authenticated query?” Test network and application layers separately. A successful `nc -vz` indicates one TCP property; a successful SQL query establishes more.

**Negative test:** “From an unrelated external host or unauthorized subnet, is the DB listener unreachable?” If the probe fails, inspect whether the DB was healthy, that the path was the intended test and whether logs support the denial. A network timeout is not by itself a comprehensive security audit.

**NAT test:** Private EC2 instance has no public IP; `curl -I https://example.com` returns a response through the intended outbound path. Document route target, NAT state and test moment. Validate DNS separately with `getent hosts`, `dig`, or `nslookup` if installed.

**Failure test:** In a disposable test environment, stop NAT intentionally and document loss of outbound connectivity, then restore. This is the difference between designing a dependency and understanding its failure mode.

## 14. DNS, endpoints and egress reduction

Not every private application needs broad internet access. For AWS APIs (e.g., S3, Systems Manager) and supported services, VPC endpoints can keep selected traffic on AWS networking without passing through a general-purpose NAT path, subject to endpoint type and service support.

DNS failures can look exactly like egress failures to a beginner. For example, `curl https://some-host` fails with a name-resolution error, while `curl` to a known IP may reach the destination. Investigate resolver configuration, VPC DNS support/hostnames, DNS security groups and endpoint dependencies before rewriting routes.

An egress policy should answer **where** the workload genuinely needs to connect, not just “allow all.” For mature systems, add network logs and workload-level allowlists where feasible.

## 15. Secure administration and secret handling

An administrator needs to manage both web and database instances. That does not imply opening SSH or RDP to the entire internet. Prefer controlled paths such as AWS Systems Manager Session Manager for eligible instances, least-privilege IAM, strong MFA and auditable commands.

Avoid attaching long-lived AWS access keys to public repositories, screenshots, AMIs or userdata. Also inspect logs, cloud console exports and copy/pasted command examples before publishing. Account numbers and public IPs are not authentication secrets by themselves, but unnecessary identifiers may be redacted when not needed for evidence.

In this repository's screenshot set, the AWS account display and one visible public client address have been masked. Crucially, **NAT state, subnet names, CIDR and route association values are left intact**.

## 16. Reliability: a secure network must survive failures safely

Reliability and security can conflict if you design only for the happy path. Consider a single NAT instance serving the private tier. If it stops, outbound package updates or API calls may fail. A fail-open workaround (e.g., giving every private server a public IP) can undermine segmentation.

A resilient design needs explicitly scoped availability goals. A single subnet in one AZ does not prove multi-AZ high availability. If a managed NAT gateway is used, placing gateways per AZ can reduce dependency on cross-AZ egress and AZ failure; make your routing choices intentional and assess fees.

**Metrics to consider:** EC2/NAT health, route target status, VPC Flow Logs, DNS errors, application failure rates and time to recovery. A security architecture that nobody can diagnose is not operationally mature.

## 17. Cost engineering: don't let a lab silently bill you

Cloud networking resources have different charging models. The internet gateway itself may not carry an hourly gateway fee, but public IPv4 allocations, EC2 instance hours, EBS volumes, NAT gateway hours, per-GB processing and traffic transfer may incur costs. Regional prices change; use the official AWS pricing calculator before assuming costs.

When a NAT instance is stopped, EC2 compute charging normally stops, but storage and certain attached/allotted resources can still bill. Managed NAT gateways may charge while provisioned even without meaningful traffic. The correct lab cleanup is to identify resources, remove intentional test infrastructure and verify no unexpected active charges remain.

Never publish exact account bills or cost claims unless you actually measured them.

## 18. Troubleshooting with a decision tree

When connectivity fails, gather **one clear symptom** and trace the path backwards:

1. Is the source workload running? Does the destination service listen?
2. Which private IP, subnet ID, VPC and AZ does the source have?
3. Which table is associated with that subnet? Which route matches destination IP?
4. Is the route target active? If NAT instance, is the EC2 running and properly routing?
5. Do source outbound SG and destination inbound SG permit the actual protocol/port?
6. Do NACLs allow both directions/ephemeral return ports?
7. Is host firewall or DNS responsible?
8. Can the same test succeed when one variable is changed?
9. What logs confirm the root cause? How do you roll back safely?

Avoid changing SG, NACL and routes simultaneously. Controlled diagnosis isolates one variable and records before/after evidence.

## 19. Historical evidence: read it like an engineer

### Screenshot 1 — the NAT instance

`Nat-Instance` appears as a stopped `t3.micro` EC2 resource in Sydney, AZ `ap-southeast-2b`. This proves the resource existed in the captured console state. It also proves the instance was *not actively forwarding at that moment*. The lower table shows security-group rule details, but not a comprehensive network audit.

### Screenshot 2 — two subnets

Both `Public-Subnet` and `Privet-Subnet` appear available. The selected private subnet shows `172.31.0.0/20`. Note the original misspelling (`Privet`); preserving the exact resource label makes documentation auditable. The screenshot does not reveal the public subnet's full route entries or a deployed database server.

### Screenshot 3 — route table associations

`Public-Subnet-RT` and `Privet-Subnet-RT` are visible and explicitly linked to their corresponding subnets. This is evidence of **network organization and association**, not proof of default route targets or live egress.

**Good technical storytelling includes uncertainty.** It is better to state “I can prove route-table associations; NAT packet forwarding still requires validation” than to pretend three screenshots establish 100% network security.

## 20. Practical exercises: learn by measurement

Try these in a disposable lab, with budget guardrails:

**Exercise A — CIDR drawing:** Sketch a `/16` VPC and two `/24` subnets. Write which addresses fall inside each. Then compute available IPv4 address count after AWS subnet reservations.

**Exercise B — routing matrix:** Write sample route tables for public and private subnets. Circle the longest-prefix match for an internal destination and an arbitrary external destination.

**Exercise C — privilege matrix:** Create a table: external user → HTTPS app (allow); app SG → DB TCP 5432 (allow); external user → DB (deny); private host → software update endpoint (allow if required). Translate into SG rules and route requirements.

**Exercise D — DNS versus transport:** From a test instance, run name resolution and an HTTPS request separately. Record the error category and troubleshoot the appropriate layer.

**Exercise E — NAT state:** Observe private egress with the NAT healthy, then in the disposable environment stop the NAT and observe the dependency. Restore and validate recovery. Capture actual state changes; do not invent them.

**Exercise F — security audit:** Review all ingress rules for `0.0.0.0/0` or unexpectedly broad ranges. Explain each necessary exception with its owning application and threat model.

## 21. Real-world failure scenarios

### Scenario 1: The private server cannot install updates

Don't immediately give it a public IP. Diagnose resolver, route table, NAT health, instance source/destination check, forwarding rules, outbound SG and return path. A private machine can remain private while obtaining controlled egress.

### Scenario 2: App has healthy web responses but DB access fails

A load balancer health check might hit a shallow `/health` endpoint while real transactions require the database. Test application-to-database path, credentials and DB service separately. The problem might be network filtering, DNS, DB outage or exhausted connections; use evidence to distinguish.

### Scenario 3: A security group was opened broadly during debugging

Restore least-privilege rules, capture the legitimate traffic's five-tuple, and verify both allowed and denied flows. Prevent recurrence with peer review, IaC, automated policy checks and account monitoring.

### Scenario 4: A single NAT instance was stopped

The diagram's egress dependency is now a real incident. Record affected workloads, route target health, time to detection, and recovery. Consider managed NAT, VPC endpoints or appropriate HA only after understanding workload and cost requirements.

### Scenario 5: Route-table names look right, packet path is wrong

Open the actual route entries and subnet association details. Tags do not route traffic. An unexpected more-specific route or stale target can explain behavior that appears impossible from just resource names.

## 22. Interview questions: explain, don't recite

**Q1. What is a VPC?**  
A logically isolated AWS virtual network where you define address space, subnets, routing and network filters. It is not equivalent to application security.

**Q2. What makes a subnet public?**  
Its route-table path to an Internet Gateway, not its Name tag. Actual internet reachability also depends on public addressing, SG/NACL and workload state.

**Q3. What does `172.31.0.0/20` mean?**  
20 fixed network bits, 12 host bits: 4096 addresses in the range `172.31.0.0` through `172.31.15.255`. In AWS, five IPv4 addresses per subnet are reserved.

**Q4. Why use two route tables?**  
To let the public and private subnet have different intended next hops and make their behavior explicit. Tables determine routes; they don't authorize traffic.

**Q5. What is the local route?**  
The VPC route that keeps traffic for the VPC address block inside VPC networking.

**Q6. What is longest-prefix match?**  
Among matching route destinations, the most specific prefix determines the forwarding target.

**Q7. Why can't a private instance simply use an IGW through a private subnet?**  
A private subnet typically has no direct IGW default route, and public IPv4 addressing matters for direct internet egress. Instead use a suitable NAT or service endpoint when outbound access is needed.

**Q8. What does NAT protect against?**  
It enables outbound-initiated translated connections without inherently allowing unsolicited inbound initiation. It is not a substitute for firewall policy or IAM.

**Q9. What is the difference between NAT instance and NAT gateway?**  
One is self-managed EC2 requiring forwarding, patching, routing and availability planning; the other is an AWS-managed NAT service with distinct availability, throughput and cost characteristics.

**Q10. What happens when your NAT instance stops?**  
Private workloads relying on it for external IPv4 egress lose that path; the blast radius depends on routes and alternative paths.

**Q11. Why disable EC2 source/destination checks for a NAT instance?**  
A forwarding host carries packets whose source/destination is another machine, unlike ordinary EC2 end-host traffic.

**Q12. How do security groups differ from NACLs?**  
SGs are stateful resource-level allowlists. NACLs are stateless subnet-level allow/deny lists; return traffic must be permitted separately.

**Q13. Is a private IP enough to secure a database?**  
No. Check paths, SGs, NACLs, listener, authentication, encryption, operating system and IAM. Scope permitted application sources precisely.

**Q14. What's stronger evidence: an Active route or a successful request?**  
They prove different things. A route state is control-plane evidence; a successful request proves a specific data-plane path at a specific time. Need both plus denied-path checks.

**Q15. How would you prove the database is unreachable from the internet?**  
Verify no direct public routing/addressing, tight DB SG rules, and actual negative network tests while checking that the DB is healthy for approved traffic.

**Q16. Can you conclude this historical project had working NAT?**  
No. The retained NAT EC2 screenshot shows **Stopped**, and route entries/forwarding/egress tests are not shown.

**Q17. Does one subnet in one AZ establish high availability?**  
No. AZ failure resilience needs multi-AZ design for relevant workloads and egress paths, not just redundant-looking names.

**Q18. What is the role of VPC endpoints?**  
Allow private connectivity to supported AWS services without general-purpose internet egress through NAT for those flows.

**Q19. What should go into a troubleshooting record?**  
Symptom, source/destination, matching route, SG/NACL, host state, test output, single change, result and rollback.

**Q20. How would you improve this repository next?**  
Capture actual route targets, NAT configuration and state, application-to-backend positive test, external-to-backend negative test, VPC Flow Logs, cleanup and cost evidence. Treat new tests as dated additions.

## 23. Production-readiness checklist and final mental model

```text
[ ] VPC/subnet CIDRs correct and documented
[ ] Each subnet's effective route table verified
[ ] IGW/NAT route targets intentional and healthy
[ ] No unnecessary public IPs or management ports
[ ] Workload SGs define precise sources/protocols/ports
[ ] NACL, host firewall, TLS, IAM and secrets reviewed
[ ] Private-to-internet egress tested only if required
[ ] Approved app-to-backend traffic succeeds
[ ] Unapproved direct backend access is denied
[ ] Monitoring and flow diagnostics available
[ ] NAT/AZ failures understood and mitigated where needed
[ ] Egress/cost choices evaluated
[ ] Recovery and cleanup documented
[ ] Screenshot claims match what screenshots actually show
```

**Remember the five-question engineering loop:** What traffic should flow? Which resource chooses the route? Which control allows it? What happens when the dependency fails? What observation proves your answer?

---

### Continue exploring

- [Architecture and packet journeys](docs/ARCHITECTURE.md)
- [Reproduction steps](docs/IMPLEMENTATION.md)
- [Observed vs unverified validation](docs/VALIDATION.md)
- [Troubleshooting handbook](docs/TROUBLESHOOTING.md)
- [Evidence index](docs/EVIDENCE-INDEX.md)
- [AWS official route tables](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html)
- [AWS official NAT guidance](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html)
- [AWS NAT instance operations](https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html)
- [AWS security-group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [AWS NAT comparison](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)

**Authorship:** Hafiz Mohammad Sunasara · Cloud & DevOps learning portfolio. This is educational work based on real console evidence; it does not imply an audited production security posture.
