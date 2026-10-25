# Validation — Evidence-Based Engineering Assessment

## Scope

This document classifies **what the three retained AWS screenshots show** and what remains unverified. The distinction is essential: resource provisioned ≠ configured correctly ≠ service reachable ≠ resilience proven.

## Observed results

| Check | Status | Evidence | Explanation |
|---|---|---|---|
| Public subnet resource visible | **Observed** | [Subnets](../evidence/02-public-private-subnets.png) | `Public-Subnet` listed Available |
| Private subnet resource visible | **Observed** | [Subnets](../evidence/02-public-private-subnets.png) | `Privet-Subnet` listed Available, selected private CIDR `172.31.0.0/20` |
| Separate route tables shown | **Observed** | [Route tables](../evidence/03-public-private-route-tables.png) | Two named tables, each explicit subnet association |
| NAT EC2 instance exists | **Observed** | [NAT instance](../evidence/01-nat-instance-stopped.png) | Resource named `Nat-Instance`, `t3.micro`, Sydney/AZ `ap-southeast-2b` |
| NAT EC2 running/forwarding | **Not demonstrated** | [NAT instance](../evidence/01-nat-instance-stopped.png) | Screenshot **explicitly shows Stopped** |
| Public table's IGW default route | **Unknown** | Route table names only | Routes tab not archived |
| Private table's NAT default route | **Unknown** | Route table names only | Routes tab not archived |
| App/database server deployments | **Unknown** | No corresponding screenshot | Tier roles are conceptual / LinkedIn description, not independently confirmed here |
| Private workload outbound test | **Not documented** | No terminal evidence | No captured DNS, TCP, HTTPS or NAT translation check |
| Denied inbound test | **Not documented** | No terminal evidence | No failed connection proof or Flow Log observation |
| Source/destination check disabled | **Unknown** | Not visible | Important for EC2 NAT-instance routing |
| Multi-AZ HA, zero downtime, hardened production | **Not demonstrated** | No tests | Cannot be inferred from these images |

## Proposed acceptance criteria for a future rerun

1. Record actual VPC/subnet CIDRs, route destinations/targets, and NAT interface placement.
2. Public route should lead to attached IGW if public app is intentionally exposed.
3. Private route should direct allowed egress via correct healthy NAT; verify source/destination check and system SNAT.
4. Backend has no direct public listener and app-to-backend is limited to necessary port/source.
5. From a private resource, verify DNS and HTTPS succeed while direct external-initiated inbound to private resource is denied.
6. Record flow logs / VPC Reachability Analyzer path information where available (these have limitations; correlate with live packet tests).
7. For resilience: introduce controlled NAT failure **only in disposable lab**, record time to detection and restoration. Document blast radius and prevention.
8. Mask account numbers, public IPs and secrets before committing any evidence.

## Honest project outcome

The screenshots show **configuration of subnet and routing separation** and presence of an **inactive EC2 NAT resource**. They are meaningful partial architectural evidence. They **do not prove** a secure, reachable or production-ready two-tier application. The implementation and deep dive explain how to close that gap through verifiable tests.
