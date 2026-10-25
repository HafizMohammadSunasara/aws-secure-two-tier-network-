# Evidence Index — What the AWS Screenshots Prove

**Principle:** Evidence should support each claim at the level it actually captures. Console configuration ≠ connectivity validation ≠ operational reliability.

| File | Visible facts | Limitations |
|---|---|---|
| [01 NAT instance](../evidence/01-nat-instance-stopped.png) | Region Sydney; `Nat-Instance`; `t3.micro`; availability zone `ap-southeast-2b`; **Stopped**; security-group rule table displayed | No NAT traffic test; no proof of subnet location, route target, source/destination-check status, forwarding, or working application |
| [02 subnets](../evidence/02-public-private-subnets.png) | `Public-Subnet` and `Privet-Subnet` listed `Available`; private subnet IPv4 CIDR `172.31.0.0/20`; shared VPC listing | Names don't prove effective public/private reachability; no subnet route list, public-address policy verification or compute instances in subnet |
| [03 route tables](../evidence/03-public-private-route-tables.png) | `Public-Subnet-RT` explicitly associated with `Public-Subnet`; `Privet-Subnet-RT` explicitly associated with `Privet-Subnet`; selected table is not main | The `Routes` tab contents are not visible; IGW/NAT default routes not proven |

## Redaction and enhancement

Screenshots are resolution-upscaled, mildly sharpened and redacted **only** in the top-right AWS account display; the NAT screenshot also masks a visible public client address. They are the same real screenshots, **not** reconstructed console mockups. Original files should be retained privately by the project owner. The repository includes **one `evidence/` folder**, not duplicative raw/enhanced trees.

## Evidence upgrade plan

For a second validation pass, capture these focused screens/outputs (only if actually executed):

- Public and private route tables on the **Routes** tab with destination, target and state.
- NAT instance networking details, current state, subnet and source/destination check.
- Network security rules for frontend/backend with allowed and denied paths.
- Private-server network proof (`curl`/DNS/trace) and external reachability **negative test**.
- NAT logs/metrics or controlled failure test to show what happens when egress fails.

Do not add fabricated output or screenshots. New experiments belong in an explicitly dated evidence addition.
