# AWS Networking: VPC, Routing, Security, and Traffic Engineering

## 30-Second Intuition

A VPC is your own private slice of AWS's network — a virtual router (the implicit VPC router), a set of subnets (IP address ranges pinned to one Availability Zone each), and a handful of attachable gateways (Internet Gateway, NAT Gateway, Transit Gateway, VPC endpoints) that decide what a packet is allowed to reach once it leaves an instance's network interface. Everything in this doc is really one question asked at four different layers: **given a packet, who decides where it goes, and who decides whether it's allowed to go there?** Routing (route tables) answers "where." Security Groups and NACLs answer "allowed?" — but they answer it with a fundamentally different memory model, and that's the one fact that causes more AWS production outages than anything else in this doc: **Security Groups are stateful (return traffic is auto-allowed), NACLs are stateless (you must explicitly allow the return traffic, including ephemeral ports, or the connection hangs instead of failing cleanly)**. This doc assumes you already know TCP/IP mechanics (three-way handshake, ports, TLS) — see [networking-layers.md](/systems-engineering/networking/networking-layers.md) for that; this doc is about what AWS bolts on top: which subnet a packet's routed through, which of two independent firewall layers inspects it, and how it gets out to the internet or to another VPC/account.

---

## Resource-Layer Map

This KB's Systems-mode template maps a technology onto CPU/memory/disk/network/GPU. That table doesn't fit here — AWS networking has no CPU scheduler, no disk, no GPU of its own; it *is* the network layer, sitting entirely inside what the generic template would call one row. Collapsing four AWS-specific concerns into "network: uses network" would say nothing. Instead, this doc maps the **network layer's own internal sublayers** — the four places a packet's fate actually gets decided inside a VPC:

| Layer | Role | What it's optimizing for |
|---|---|---|
| **Routing** (route tables) | Per-subnet table deciding the next hop for a packet based on destination CIDR | Longest-prefix-match correctness; local routes always win by construction |
| **Security enforcement** (Security Groups + NACLs) | Two independent, differently-scoped firewall layers — SG at the ENI, NACL at the subnet boundary | SG: stateful, allow-only, cheap per-connection tracking. NACL: stateless, allow+deny, coarse defense-in-depth |
| **NAT / egress** (NAT Gateway, NAT instance, Egress-Only IGW) | Where outbound-only internet access for private-subnet resources is enforced and where the public IP substitution happens | Managed HA + throughput (NAT Gateway) vs. cost control at low scale (NAT instance) |
| **Inter-VPC / inter-account connectivity** (Peering, Transit Gateway, PrivateLink) | How traffic crosses a VPC boundary at all — mesh links, a routing hub, or service-level ENI exposure | Peering: cheap, 1:1, non-transitive. Transit Gateway: hub-and-spoke, transitive, scales past O(n²). PrivateLink: no route table entry at all — service exposure via ENI |

Every mechanism below is one of these four rows. If you're ever unsure which section of this doc a given AWS networking question belongs in, ask "is this about where a packet goes, whether it's allowed, how it gets to the internet, or how it crosses a VPC boundary" — those are the four rows, and they compose in that order on every packet's path.

---

## The Signature Mechanism: Stateful SGs vs. Stateless NACLs

Every other AWS networking concept in this doc (routing, NAT, peering) has a close non-AWS analog. This one doesn't get the respect it deserves because both mechanisms look like "just a firewall" from a distance — they are not implemented the same way, and conflating them is the single most consequential AWS networking mistake in production.

- **Security Groups are stateful**: they track connections. If you allow *outbound* traffic on a rule, the corresponding *inbound* return traffic for that same connection is automatically permitted — AWS's hypervisor-level connection tracker remembers "this instance opened a connection out on 443, so let the reply back in," without you writing a second rule.
- **Network ACLs are stateless**: no connection tracking. An allowed outbound request does **not** imply the reply is allowed back in. You must write two rules — one for the request direction, one for the reply direction — and because a TCP client's source port is an ephemeral (high, randomly assigned) port, the NACL's inbound-reply rule has to open the *entire ephemeral range*, not the specific port.

### Worked example: an EC2 instance making one outbound HTTPS call

The instance calls `https://api.example.com:443`. TCP negotiates ephemeral source port `54321` on the instance side; the server replies from `443` to `54321`.

**Security Group — 1 rule needed:**

```
Outbound rule:  Protocol=TCP  Port=443           Destination=0.0.0.0/0
# Inbound reply on port 54321 is allowed automatically — connection tracking
# recognizes the reply belongs to a connection this instance's SG already permitted outbound.
```

```bash
aws ec2 authorize-security-group-egress \
  --group-id sg-0abc123456789 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
# That's it. No inbound rule required for the reply.
```

**Network ACL — 2 rules needed, and one of them has to be a range:**

```
Outbound rule 100:  Protocol=TCP  Port=443            Destination=0.0.0.0/0   ALLOW
Inbound  rule 100:  Protocol=TCP  Port=1024-65535      Source=0.0.0.0/0        ALLOW
# The inbound rule can't say "port 54321" — the client picks a random ephemeral
# port per connection, so the NACL must open the *entire* ephemeral range for
# ANY outbound-initiated TCP reply to work at all.
```

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0123456789abcdef0 \
  --rule-number 100 --protocol tcp --port-range From=443,To=443 \
  --egress --cidr-block 0.0.0.0/0 --rule-action allow

aws ec2 create-network-acl-entry \
  --network-acl-id acl-0123456789abcdef0 \
  --rule-number 100 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow
# (no --egress flag = inbound rule)
```

Forget the second NACL rule and the outbound SYN leaves cleanly, the server replies, and the reply is silently dropped at the subnet boundary on the way back in. There is no RST, no ICMP error the application can see — from the instance's point of view the connection just **hangs until the client's own TCP timeout fires**, which is why this specific mistake is so much harder to debug than a Security-Group misconfiguration: an SG mistake fails fast and loud (connection refused / timed out immediately at connect()); a missing-ephemeral-range NACL mistake fails slow and silent (connect() succeeds, then nothing, for 30-120+ seconds depending on client timeout settings).

This maps directly onto the resource-layer table's "security enforcement" row: SG's per-connection state tracking is *why* it only needs allow-only rules in one direction; NACL's statelessness is *why* it needs allow+deny in both directions including a port range most engineers forget exists.

---

## High-to-Low Walkthrough: Building a 2-Tier VPC and Tracing a Packet

### Step 1 — Create the VPC

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=demo-vpc}]'
```

```json
{
  "Vpc": {
    "CidrBlock": "10.0.0.0/16",
    "VpcId": "vpc-0a1b2c3d4e5f67890",
    "State": "pending",
    "InstanceTenancy": "default"
  }
}
```

`10.0.0.0/16` gives 65,536 addresses; AWS reserves the first 4 and the last 1 of every subnet carved from it (network address, VPC router, DNS, future-use, broadcast — IPv4 has no broadcast concept in VPC but AWS still reserves it), so a `/24` subnet's usable count is 256 − 5 = 251, not 256.

### Step 2 — Carve public and private subnets in the same AZ

```bash
aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f67890 \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

aws ec2 create-subnet --vpc-id vpc-0a1b2c3d4e5f67890 \
  --cidr-block 10.0.2.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'
```

A subnet is pinned to exactly one AZ at creation and can never move — this is why every "one resource per AZ" gotcha in this doc (NAT Gateway included) traces back to this single fact.

### Step 3 — Internet Gateway, attached to the VPC (not a subnet)

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=demo-igw}]'
# returns igw-0123456789abcdef0

aws ec2 attach-internet-gateway \
  --vpc-id vpc-0a1b2c3d4e5f67890 --internet-gateway-id igw-0123456789abcdef0
```

### Step 4 — Public route table: 0.0.0.0/0 → IGW

```bash
aws ec2 create-route-table --vpc-id vpc-0a1b2c3d4e5f67890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'
# returns rtb-0111111111111aaaa

aws ec2 create-route --route-table-id rtb-0111111111111aaaa \
  --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0123456789abcdef0

aws ec2 associate-route-table \
  --route-table-id rtb-0111111111111aaaa --subnet-id subnet-0aaaa1111bbbb2222   # public-1a
```

A subnet is "public" purely because its associated route table sends `0.0.0.0/0` to an IGW — there is no separate "make this subnet public" flag.

### Step 5 — NAT Gateway for the private subnet (needs an Elastic IP, lives in the public subnet)

```bash
aws ec2 allocate-address --domain vpc
# returns AllocationId eipalloc-0123456789abcdef0

aws ec2 create-nat-gateway \
  --subnet-id subnet-0aaaa1111bbbb2222 \
  --allocation-id eipalloc-0123456789abcdef0 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=demo-nat}]'
# returns nat-0fedcba9876543210 — this NAT Gateway is scoped to AZ us-east-1a
```

### Step 6 — Private route table: 0.0.0.0/0 → NAT Gateway

```bash
aws ec2 create-route-table --vpc-id vpc-0a1b2c3d4e5f67890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]'
# returns rtb-0222222222222bbbb

aws ec2 create-route --route-table-id rtb-0222222222222bbbb \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0fedcba9876543210

aws ec2 associate-route-table \
  --route-table-id rtb-0222222222222bbbb --subnet-id subnet-0bbbb2222cccc3333   # private-1a
```

### Step 7 — Security group for the public-subnet web tier

```bash
aws ec2 create-security-group --vpc-id vpc-0a1b2c3d4e5f67890 \
  --group-name web-sg --description "public web tier"
# returns sg-0aaa111122223333

aws ec2 authorize-security-group-ingress \
  --group-id sg-0aaa111122223333 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
# Outbound is open by default on a new SG (implicit allow-all egress) —
# no equivalent implicit-allow exists for a new custom NACL, see below.
```

### Step 8 — Verify with a route table lookup

```bash
aws ec2 describe-route-tables --route-table-ids rtb-0111111111111aaaa \
  --query 'RouteTables[0].Routes'
```

```json
[
  { "DestinationCidrBlock": "10.0.0.0/16", "GatewayId": "local", "Origin": "CreateRouteTable" },
  { "DestinationCidrBlock": "0.0.0.0/0",   "GatewayId": "igw-0123456789abcdef0", "Origin": "CreateRoute" }
]
```

The `local` route is implicit, always present, and always wins for any address inside the VPC's CIDR — you cannot remove or override it, which is exactly why VPC Peering's non-transitivity (below) can't be worked around by "just adding a route."

### Packet trace 1 — internet client to the public-subnet instance

```
Internet client
   │  TCP SYN to <public IP>:443
   ▼
Internet Gateway (1:1 NAT: public IP ⇄ instance's private IP, stateless mapping)
   ▼
VPC router: destination = instance's private IP (10.0.1.50) → matches `local` route
   ▼
Subnet route table: no lookup needed further, target is inside the VPC
   ▼
Network ACL on public-1a subnet (stateless — checks inbound rule for 443, must also
   have outbound rule allowing ephemeral range for the SYN-ACK to leave)
   ▼
Security Group on the instance's ENI (stateful — inbound 443 checked once;
   return traffic auto-allowed for the life of the connection)
   ▼
EC2 instance, port 443
```

### Packet trace 2 — private-subnet instance calling out through NAT Gateway

```
Private-subnet instance (10.0.2.75) calls api.example.com:443
   │
   ▼
Security Group (stateful): outbound 443 allowed → connection tracked
   ▼
NACL on private-1a subnet (stateless): outbound 443 allowed,
   inbound ephemeral range must be allowed for the reply
   ▼
Private route table: 0.0.0.0/0 → nat-0fedcba9876543210 (NAT Gateway, in public-1a)
   ▼
NAT Gateway: rewrites source (10.0.2.75:54321) → (NAT's Elastic IP:some port),
   keeps a translation table entry to reverse this on the reply
   ▼
Public route table (public-1a, where the NAT Gateway's ENI lives): 0.0.0.0/0 → IGW
   ▼
Internet Gateway → internet → api.example.com
```

Note the NAT Gateway's ENI lives *in* the public subnet and uses *that* subnet's route table to reach the IGW — the private instance never talks to the IGW directly, and the IGW has no idea the private instance exists.

---

## Deep Internals

### Route table evaluation: most-specific-match wins, not first-match

A route table is not evaluated top-to-bottom like a firewall rule list — every route is checked and the **longest matching prefix (most specific CIDR) wins**, regardless of insertion order.

```bash
aws ec2 create-route --route-table-id rtb-0222222222222bbbb \
  --destination-cidr-block 10.0.0.0/16 --gateway-id local   # implicit, always present
aws ec2 create-route --route-table-id rtb-0222222222222bbbb \
  --destination-cidr-block 10.99.0.0/24 --vpc-peering-connection-id pcx-0abc123456789
aws ec2 create-route --route-table-id rtb-0222222222222bbbb \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0fedcba9876543210
```

A packet to `10.99.0.5` matches both `0.0.0.0/0` (32 bits less specific) and `10.99.0.0/24`; the `/24` wins because it's a longer, more specific prefix — the packet goes over the peering connection, not to the NAT Gateway, even though the NAT route was added last. This is the same longest-prefix-match rule as any IP router; AWS route tables are not a special case.

### Why VPC Peering does not transitively route

VPC A (`10.0.0.0/16`) is peered with VPC B (`10.1.0.0/16`). VPC B is separately peered with VPC C (`10.2.0.0/16`). A cannot reach C through B, ever, without a third direct A↔C peering connection.

The reason is mechanical, not a policy restriction: peering only installs a route in the *two peered VPCs'* route tables, pointing at the peering connection ID as if it were a gateway. It never propagates a route into a third VPC.

```bash
# On VPC A's route table — this route only exists because A is directly peered with B:
aws ec2 create-route --route-table-id rtb-A \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-AB

# VPC A's route table has NO entry for 10.2.0.0/16 (VPC C's CIDR) — that route was
# never created, because A and C never established a peering connection with each
# other. B's route table has a route to C via pcx-BC, but that's B's route table,
# not A's — traffic from A arriving at B over pcx-AB is not re-routed onward to
# pcx-BC. A peering connection is a point-to-point link, not a router hop.
aws ec2 describe-route-tables --route-table-ids rtb-A --query 'RouteTables[0].Routes'
```

```json
[
  { "DestinationCidrBlock": "10.0.0.0/16", "GatewayId": "local" },
  { "DestinationCidrBlock": "10.1.0.0/16", "VpcPeeringConnectionId": "pcx-AB" }
]
```

No route to `10.2.0.0/16` exists in A's table and AWS will not synthesize one — this is the single most common multi-VPC incident root cause in production AWS estates: someone assumes "A talks to B, B talks to C" implies "A talks to C," adds no explicit route, and gets silent unreachability (not an error — the packet just has nowhere to go and the connection times out, same silent-failure shape as the missing-NACL-ephemeral-rule gotcha above, different cause).

### Transit Gateway: the hub-and-spoke fix

Transit Gateway is a regional router that VPCs attach to as spokes; it maintains its *own* route table(s) and — critically — **does** propagate reachability transitively among everything attached to it, because every attached VPC's routes go through the *same* central router rather than through separate point-to-point links.

```bash
aws ec2 create-transit-gateway \
  --description "hub for A, B, C" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable
# returns tgw-0123456789abcdef0

aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0123456789abcdef0 --vpc-id vpc-A --subnet-ids subnet-A1
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0123456789abcdef0 --vpc-id vpc-B --subnet-ids subnet-B1
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0123456789abcdef0 --vpc-id vpc-C --subnet-ids subnet-C1

# Each VPC's own route table still needs one route pointing AT the TGW —
# TGW attachment doesn't remove the need for a route, it just means you only
# need ONE route per VPC (to the TGW) instead of one route per peer VPC:
aws ec2 create-route --route-table-id rtb-A \
  --destination-cidr-block 10.0.0.0/8 --transit-gateway-id tgw-0123456789abcdef0
```

With default route table association/propagation enabled, A, B, and C can now all reach each other with **3 attachments** instead of the **3 pairwise peering connections** a full mesh of 3 VPCs would need — and that gap grows fast: 10 VPCs need 45 peering connections in a full mesh (`n(n-1)/2`) but only 10 Transit Gateway attachments. Transit Gateway currently supports up to 5,000 attachments per Transit Gateway (regional quota, adjustable) with up to 100 Gbps per Availability Zone per VPC attachment; VPC Peering has no separate published bandwidth cap of its own (bound by underlying instance/ENI limits) but is capped at 125 active peering connections per VPC.

### VPC Endpoints: Gateway (free) vs. Interface/PrivateLink (metered)

Two unrelated mechanisms share the name "VPC endpoint":

**Gateway endpoints** — S3 and DynamoDB only, free, work by adding a target to the route table (no ENI, no IP address consumed):

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0a1b2c3d4e5f67890 --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway --route-table-ids rtb-0222222222222bbbb
```

```bash
aws ec2 describe-route-tables --route-table-ids rtb-0222222222222bbbb \
  --query 'RouteTables[0].Routes'
```

```json
[
  { "DestinationCidrBlock": "10.0.0.0/16", "GatewayId": "local" },
  { "DestinationCidrBlock": "0.0.0.0/0", "NatGatewayId": "nat-0fedcba9876543210" },
  { "DestinationPrefixListId": "pl-63a5400a", "GatewayId": "vpce-0123456789abcdef0" }
]
```

Traffic to S3's published IP-prefix list now routes straight to the endpoint instead of out through the NAT Gateway — this is also a cost optimization: S3 traffic that used to be metered NAT Gateway data-processing charges (`$0.045/GB`, us-east-1, 2026 pricing) now costs nothing.

**Interface endpoints (PrivateLink)** — almost every other AWS service (Secrets Manager, SQS, ECR, CloudWatch Logs, KMS, etc.), backed by an ENI in your subnet with a private IP, billed per-AZ-hour plus per-GB:

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0a1b2c3d4e5f67890 --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface --subnet-ids subnet-0bbbb2222cccc3333 \
  --security-group-ids sg-0aaa111122223333
```

Interface endpoints are ENI-based and stateful like any other ENI — they sit behind a Security Group you control, unlike gateway endpoints which have no SG of their own (they're a route table entry, not a network interface). Pricing (us-east-1, 2026): roughly `$0.01/hour` per AZ the endpoint is provisioned in, plus `$0.01/GB` processed — a 3-AZ interface endpoint runs a fixed ~`$21.90/month` baseline *before* any data, and a workload calling many different AWS services from a private subnet (Secrets Manager, KMS, SQS, ECR, CloudWatch Logs, STS...) pays that baseline **per service**, which is why teams that reflexively "just add an interface endpoint for everything" for defense-in-depth see this line item grow unexpectedly — check whether NAT Gateway's per-GB processing charge would actually be cheaper than N interface endpoints' combined hourly baseline for low-traffic services, and reserve interface endpoints for services with either compliance requirements (no public internet path) or genuinely high call volume where NAT's per-GB cost would exceed the endpoint's flat hourly rate.

### NAT Gateway vs. NAT instance

| | NAT Gateway | NAT instance (self-managed EC2 + IP forwarding) |
|---|---|---|
| Management | Fully managed by AWS | You patch, scale, and monitor the EC2 instance yourself |
| HA within an AZ | Built-in (AWS-managed redundancy inside the AZ) | None — a single EC2 instance is a single point of failure unless you build your own failover (scripted EIP re-association, etc.) |
| Multi-AZ HA | Requires one NAT Gateway **per AZ** you want covered (traditional "zonal" mode) — see gotcha below | Same requirement, plus you're building the failover logic yourself |
| Throughput | Scales automatically up to 100 Gbps | Capped by the instance type you chose (and its network performance tier) |
| Cost model | `$0.045/hour` + `$0.045/GB` processed (us-east-1, 2026) — bills even when idle | EC2 instance-hour cost only (e.g. a `t4g.nano` at a few dollars/month) — no per-GB processing fee, cheaper at low, steady, low-bandwidth traffic |
| Security group | Cannot attach one (managed resource) | Yes — you control its SG like any instance |

```bash
# Regional NAT Gateway (announced Nov 2025) — one gateway that automatically
# expands across AZs as workloads appear, instead of one-per-AZ:
aws ec2 create-nat-gateway \
  --subnet-id subnet-0aaaa1111bbbb2222 \
  --allocation-id eipalloc-0123456789abcdef0 \
  --availability-mode regional
```

The regional mode (see gotcha below) changes the traditional "you must provision one NAT Gateway per AZ" guidance for newly-built VPCs, but it is optional and additive — the default `create-nat-gateway` call without `--availability-mode regional` still creates the traditional AZ-scoped resource, and most existing production VPCs (built before Nov 2025) are still running the zonal version.

---

## Comparative: Security Groups vs. NACLs, and Peering vs. Transit Gateway vs. PrivateLink

| | Security Group | Network ACL |
|---|---|---|
| Scope | Per ENI / instance | Per subnet (applies to everything in it) |
| State | Stateful — return traffic auto-allowed | Stateless — both directions must be explicit |
| Rule types | Allow only | Allow **and** explicit deny |
| Evaluation | All matching rules evaluated (no ordering) | Rule numbers evaluated in order, first match wins |
| Default (new custom resource) | Deny all inbound, allow all outbound | Deny all inbound and outbound (custom NACL) / default NACL allows all |
| Typical use | Primary, fine-grained access control | Coarse defense-in-depth; the only layer that can explicitly **deny** |

**When you'd actually reach for a NACL deny rule** (SGs structurally cannot deny anything — allow-only):

```bash
# Block a specific malicious /24 at the subnet boundary, defense-in-depth,
# independent of whatever any instance's Security Group says:
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0123456789abcdef0 \
  --rule-number 50 --protocol -1 --cidr-block 198.51.100.0/24 \
  --rule-action deny
# Rule number 50 evaluates before rule 100 (the broader allow) — first
# numeric match wins, so this DENY is checked and applied before the
# ALLOW 0.0.0.0/0 rule ever gets a chance to match this specific /24.
```

Reach for this when: you have an entire subnet's worth of instances (so a per-instance SG change would mean touching every SG) that all need to reject one known-bad range immediately, without waiting to identify and edit N different security groups.

| | VPC Peering | Transit Gateway | PrivateLink (Interface Endpoint) |
|---|---|---|---|
| Topology | Point-to-point, one connection per VPC pair | Hub-and-spoke, one attachment per VPC | Not a VPC-to-VPC link at all — service exposure via ENI |
| Transitive routing | No — never | Yes — by design | N/A (no routing between VPCs happens) |
| Scaling to N VPCs | O(n²) connections | O(n) attachments | N/A — scales per consumer, not per VPC pair |
| Max per resource (2026 quotas) | 125 peering connections/VPC | 5,000 attachments/TGW | Limited by ENIs/subnets, not a peer count |
| Cost | Free (data transfer charged at normal cross-AZ/region rates) | Per-attachment-hour + per-GB processed | Per-AZ-hour + per-GB processed |
| Use when | A handful of VPCs, simple topology, cost-sensitive | Many VPCs/accounts, hybrid on-prem via Direct Connect/VPN, need transitivity | Consuming a *specific service* privately (yours or a SaaS vendor's) without routing entire VPC CIDRs together |

---

## Key Gotchas

- **Peering is not transitive — the single most common multi-VPC incident root cause**: A↔B and B↔C peered does not give A↔C reachability. No route to C ever gets created in A's route table, so the failure is silent unreachability (connection timeout), not an error. Fix: either a direct A↔C peering connection, or replace the peering mesh with Transit Gateway once you have more than a few VPCs that all need to talk to each other.
- **Forgotten NACL ephemeral-port rule → silent hangs, not clean rejections**: a NACL that allows outbound 443 but not inbound `1024-65535` doesn't reject the connection — the SYN and even the server's reply leave/arrive fine at the IP layer, but the reply is dropped at the subnet boundary on the way back to the client, so the connection just hangs until the client's own TCP timeout. This is much harder to debug than an SG mistake (which fails fast) — if a connection times out slowly rather than failing immediately, suspect a NACL before anything else.
- **Security group rule limits eventually force a redesign, not just "add more rules"**: default quota is 60 inbound + 60 outbound rules per security group (adjustable, but capped so that `rules-per-SG × SGs-per-ENI ≤ 1,000`), and 5 SGs per ENI by default (adjustable up to 16). Wide-open "allow every service that talks to this instance" security groups hit this ceiling in practice long before 1,000 total rules feels close — the fix is usually referencing a customer-managed prefix list (one prefix list reference counts as one rule regardless of how many CIDRs it holds) instead of one rule per CIDR.
- **NAT Gateway is (traditionally) a per-AZ resource, not a VPC-wide one**: a NAT Gateway created in `us-east-1a` provides no failover for instances in `us-east-1b` — if `us-east-1a` has an issue, `us-east-1b`'s private instances lose internet access too unless a *second* NAT Gateway was provisioned in `us-east-1b` with its own route in that AZ's private route table. This is why the standard HA pattern is one NAT Gateway per AZ, each referenced only by that AZ's private subnets' route tables — not one NAT Gateway shared across AZs pointed at by every private route table. **Open/recent item**: AWS announced a "regional" NAT Gateway availability mode in November 2025 (`--availability-mode regional`) that automatically expands across AZs from a single resource — this changes the guidance for newly-built VPCs, but most existing production VPCs still run the traditional zonal NAT Gateway and the one-per-AZ rule still applies to them. Confirm which mode a given VPC uses before assuming either behavior.
- **Interface endpoint costs add up fast for chatty, multi-service workloads**: each interface endpoint bills roughly `$0.01/hour` per AZ plus `$0.01/GB` processed (2026, us-east-1) — a private-subnet workload that calls Secrets Manager, KMS, SQS, STS, and CloudWatch Logs across 3 AZs is paying that hourly baseline **five times over**, before any data-processing charges, which can exceed what routing the same calls out through a NAT Gateway would have cost for anything but high-volume or compliance-mandated traffic. Gateway endpoints (S3, DynamoDB) don't have this problem — they're free — so this gotcha is specific to Interface/PrivateLink endpoints for the ~100+ other services that use them.
