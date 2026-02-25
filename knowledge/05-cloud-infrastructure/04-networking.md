# Networking Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Networking

Networking in AWS is how you isolate resources, control traffic flow, and expose services safely. Every decision here has a security implication — this note covers both the technical mechanics and the functional reasoning behind each layer.

## Prerequisites

- [Compute Services](./01-compute.md) — EC2 instances and EKS clusters live inside VPCs
- [Security and IAM](./05-security-and-iam.md) — IAM controls who can call AWS APIs; networking controls what traffic can reach your resources

---

## Amazon VPC (Virtual Private Cloud)

A VPC is a logically isolated network you define inside AWS. Nothing gets in or out unless you explicitly allow it. Think of it as your private data center network, but software-defined.

### CIDR and Private IP Ranges

When you create a VPC you give it a CIDR block like `10.0.0.0/16`. This defines the entire pool of IP addresses available inside your VPC.

**What the `/16` means:**

An IP address is 32 bits total. The number after the `/` tells you how many of those bits are locked (fixed network part) vs free (yours to assign). Locked bits are marked `x`, free bits are marked `_`.

`10.0.0.0/16` — first 16 bits locked:

```
Octet:    10              . 0               . 0               . 0   
Binary:   0 0 0 0 1 0 1 0 . 0 0 0 0 0 0 0 0 . 0 0 0 0 0 0 0 0 . 0 0 0 0 0 0 0 0
Locked:   x x x x x x x x   x x x x x x x x   _ _ _ _ _ _ _ _   _ _ _ _ _ _ _ _
          └───────────────────────────────┘   └──────────────────────────────────┘
                   fixed (must stay 10.0)               free (0–255 . 0–255)
```

The 16 free bits can be anything → `10.0.0.0` to `10.0.255.255` — **65,536 addresses**.

---

Now carve a `/24` subnet out of that VPC — `10.0.1.0/24` — first 24 bits locked:

```
Octet:    10              . 0               . 1               . 0   
Binary:   0 0 0 0 1 0 1 0 . 0 0 0 0 0 0 0 0 . 0 0 0 0 0 0 0 1 . 0 0 0 0 0 0 0 0
Locked:   x x x x x x x x   x x x x x x x x   x x x x x x x x   _ _ _ _ _ _ _ _
          └──────────────────────────────────────────────────┘  └──────────────┘
                        fixed (must stay 10.0.1)                   free (0–255)
```

Only 8 bits free → `10.0.1.0` to `10.0.1.255` — **256 addresses**.

---

Side by side — you can see exactly how subnets are slices of the VPC:

```
/16  VPC:
  x x x x x x x x . x x x x x x x x . _ _ _ _ _ _ _ _ . _ _ _ _ _ _ _ _
  10                  0                  (0–255)            (0–255)
  → covers 10.0.0.0 – 10.0.255.255   (65,536 addresses)

/24  subnet 1  (public):
  x x x x x x x x . x x x x x x x x . x x x x x x x x . _ _ _ _ _ _ _ _
  10                  0                  0                  (0–255)
  → covers 10.0.0.0 – 10.0.0.255     (256 addresses)

/24  subnet 2  (public):
  x x x x x x x x . x x x x x x x x . x x x x x x x x . _ _ _ _ _ _ _ _
  10                  0                  1                  (0–255)
  → covers 10.0.1.0 – 10.0.1.255     (256 addresses)

/24  subnet 3  (private):
  x x x x x x x x . x x x x x x x x . x x x x x x x x . _ _ _ _ _ _ _ _
  10                  0                  10                 (0–255)
  → covers 10.0.10.0 – 10.0.10.255   (256 addresses)
```

The pattern: the more bits you lock (`x`), the smaller the block. `/32` locks all 32 bits — exactly one address.

**Why `10.x.x.x` specifically?**

`10.x.x.x` is a private IP range defined by RFC 1918. These addresses are not routable on the public internet — no router on the internet will forward a packet destined for `10.0.0.5`. That's exactly why AWS uses them inside VPCs. The three private ranges are:

```
10.0.0.0/8        →  10.0.0.0    – 10.255.255.255   (16 million addresses)
172.16.0.0/12     →  172.16.0.0  – 172.31.255.255
192.168.0.0/16    →  192.168.0.0 – 192.168.255.255  (your home router uses this)
```

You can use any of these three for your VPC. `10.0.0.0/16` is the most common choice because it gives you the most room to grow — you can create many subnets without running out of addresses.

**Why not `/8`? That's even bigger.**

You're right that `/8` gives more addresses — 16 million vs 65,536. But AWS has hard limits on how large a VPC CIDR can be:

```
AWS allowed VPC CIDR range:   /16  (65,536 addresses)   ← largest allowed
                         to   /28  (16 addresses)       ← smallest allowed

/8  is NOT allowed — AWS rejects it
```

So `/16` is simply the biggest you're allowed to go for a single VPC. That's why it's the default recommendation — you're taking the maximum AWS gives you.

Beyond that, even if AWS allowed `/8`, there are practical reasons not to want one giant block:

- **Peering conflicts** — if you ever connect two VPCs or connect to an on-premises network, overlapping CIDRs break routing. A `/8` block is so large it's almost guaranteed to clash with something else.
- **Blast radius** — a misconfigured route in a `/8` network can affect 16 million addresses. Smaller, well-defined blocks are easier to reason about and contain mistakes.
- **Multiple VPCs is the right pattern** — instead of one massive `/8` VPC, the AWS best practice is multiple VPCs per environment (dev, staging, prod) each with their own `/16`, connected via VPC peering or Transit Gateway. This gives you isolation between environments and avoids the peering conflict problem.

```
Good pattern:
  dev VPC:     10.0.0.0/16   (10.0.x.x)
  staging VPC: 10.1.0.0/16   (10.1.x.x)
  prod VPC:    10.2.0.0/16   (10.2.x.x)

  No overlap → can peer all three safely

Bad pattern:
  one giant VPC: 10.0.0.0/8  (not allowed anyway, and would be a mess)
```

> Avoid picking a CIDR that overlaps with other networks you might need to connect to (e.g. on-premises, VPN, VPC peering). If your office network is `10.0.0.0/16` and your VPC is also `10.0.0.0/16`, you can't peer them — the addresses clash.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "main-vpc" }
}
```

---

### Subnets

Subnets divide your VPC's IP range into smaller segments. The CIDR block (e.g. `10.0.1.0/24`) just defines the IP address range of that subnet — it has nothing to do with whether the subnet is public or private.

**What actually makes a subnet public or private is its route table.**

- **Public Subnet** — its route table has a route `0.0.0.0/0 → Internet Gateway`. Any traffic destined for the internet gets sent to the IGW.
- **Private Subnet** — its route table has a route `0.0.0.0/0 → NAT Gateway` (or no default route at all). Traffic can go out through NAT, but nothing from the internet can initiate a connection in.

```
Route table for PUBLIC subnet:
  10.0.0.0/16  →  local          (stay inside VPC)
  0.0.0.0/0    →  igw-xxxxxxxx   (everything else → internet)

Route table for PRIVATE subnet:
  10.0.0.0/16  →  local          (stay inside VPC)
  0.0.0.0/0    →  nat-xxxxxxxx   (everything else → NAT Gateway)
```

The subnet resource itself is neutral — you associate a route table to it, and that association is what determines its behaviour. You can have two subnets with identical CIDR sizes sitting next to each other; one is public and one is private purely because of which route table is attached.

> Security principle: put everything in private subnets by default. Only move something to a public subnet if it explicitly needs to be internet-facing. Your EC2 instances, RDS, and EKS nodes should never be in a public subnet.

**Breaking down the Terraform subnet code:**

```hcl
resource "aws_subnet" "public" {
  count      = 2
  cidr_block = cidrsubnet("10.0.0.0/16", 8, count.index)
}
```

Two things to unpack here:

**1. Why pass `"10.0.0.0/16"` into `cidrsubnet` if the VPC already has it?**

The VPC resource and the subnet resource are separate Terraform blocks. The subnet doesn't automatically know the VPC's CIDR — you have to tell `cidrsubnet()` what the parent range is so it can do the math. In practice you'd reference the VPC: `cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)` — that way if you ever change the VPC CIDR, the subnets update automatically.

**2. What does `cidrsubnet("10.0.0.0/16", 8, count.index)` actually do?**

`cidrsubnet(parent_cidr, newbits, index)` carves a smaller subnet out of the parent.

- `parent_cidr` = `"10.0.0.0/16"` — the pool to carve from
- `newbits` = `8` — how many extra bits to lock down (16 + 8 = `/24` subnet)
- `index` = `count.index` — which slice to take (0, 1, 2…)

```
cidrsubnet("10.0.0.0/16", 8, 0)  →  10.0.0.0/24   (index 0 = first slice)
cidrsubnet("10.0.0.0/16", 8, 1)  →  10.0.1.0/24   (index 1 = second slice)
cidrsubnet("10.0.0.0/16", 8, 2)  →  10.0.2.0/24   (index 2 = third slice)
```

**Best practice: split the VPC into named ranges upfront using explicit CIDR maps**

Instead of relying on offsets, pre-allocate a dedicated range per tier. The VPC is `/16` (256 possible `/24` subnets). Divide it into clean sections:

```
VPC: 10.0.0.0/16  (indices 0–255 available)

  0–63    →  public tier    (64 subnets worth of room)
  64–127  →  private tier   (64 subnets worth of room)
  128–191 →  database tier  (64 subnets worth of room)
  192–255 →  reserved / future use
```

In Terraform, use a `locals` map with explicit CIDRs per tier — no magic offsets, no guessing:

```hcl
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)

  public_cidrs   = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i)]
  private_cidrs  = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i + 64)]
  database_cidrs = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i + 128)]
}

resource "aws_subnet" "public" {
  count                   = length(local.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${local.azs[count.index]}" }
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = { Name = "private-${local.azs[count.index]}" }
}

resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = { Name = "database-${local.azs[count.index]}" }
}
```

What this looks like in the VPC:

```
VPC: 10.0.0.0/16
  ├── public tier    (indices 0–63,   room for 64 subnets)
  │     ├── 10.0.0.0/24   AZ-a
  │     ├── 10.0.1.0/24   AZ-b
  │     └── 10.0.2.0/24   AZ-c
  │
  ├── private tier   (indices 64–127, room for 64 subnets)
  │     ├── 10.0.64.0/24  AZ-a
  │     ├── 10.0.65.0/24  AZ-b
  │     └── 10.0.66.0/24  AZ-c
  │
  ├── database tier  (indices 128–191, room for 64 subnets)
  │     ├── 10.0.128.0/24 AZ-a
  │     ├── 10.0.129.0/24 AZ-b
  │     └── 10.0.130.0/24 AZ-c
  │
  └── reserved       (indices 192–255, free for future tiers)
```

No overlap possible between tiers regardless of how many AZs or subnets you add, as long as you stay within your 64-index allocation per tier. The names in `locals` make the intent obvious — no mental math required.

---

**Separating environments: staging vs prod**

Each environment gets its own VPC with a non-overlapping CIDR. Only `var.vpc_cidr` changes per environment — the subnet logic is identical across all of them.

```hcl
variable "vpc_cidr" {
  description = "VPC CIDR block — must be unique per environment"
  type        = string
}

variable "environment" {
  type = string
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = var.environment, Environment = var.environment }
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)

  public_cidrs   = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i)]
  private_cidrs  = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i + 64)]
  database_cidrs = [for i, az in local.azs : cidrsubnet(aws_vpc.main.cidr_block, 8, i + 128)]
}

resource "aws_subnet" "public" {
  count                   = length(local.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${local.azs[count.index]}", Environment = var.environment }
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = { Name = "private-${local.azs[count.index]}", Environment = var.environment }
}

resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = { Name = "database-${local.azs[count.index]}", Environment = var.environment }
}
```

Deploy each environment by passing a different `vpc_cidr` — the tier layout is always the same, just shifted. You define the value in a per-environment `tfvars` file:

```
infrastructure/
  envs/
    dev/
      terraform.tfvars     ← vpc_cidr = "10.0.0.0/16", environment = "dev"
    staging/
      terraform.tfvars     ← vpc_cidr = "10.1.0.0/16", environment = "staging"
    prod/
      terraform.tfvars     ← vpc_cidr = "10.2.0.0/16", environment = "prod"
```

```hcl
# envs/dev/terraform.tfvars
vpc_cidr    = "10.0.0.0/16"
environment = "dev"

# envs/staging/terraform.tfvars
vpc_cidr    = "10.1.0.0/16"
environment = "staging"

# envs/prod/terraform.tfvars
vpc_cidr    = "10.2.0.0/16"
environment = "prod"
```

Terraform reads the `vpc_cidr` from the tfvars file, passes it into `cidrsubnet`, and the locals compute all the subnet CIDRs automatically. The output for each environment:

```
dev     vpc_cidr = "10.0.0.0/16":
  public:   10.0.0.0/24,   10.0.1.0/24,   10.0.2.0/24
  private:  10.0.64.0/24,  10.0.65.0/24,  10.0.66.0/24
  database: 10.0.128.0/24, 10.0.129.0/24, 10.0.130.0/24

staging vpc_cidr = "10.1.0.0/16":
  public:   10.1.0.0/24,   10.1.1.0/24,   10.1.2.0/24
  private:  10.1.64.0/24,  10.1.65.0/24,  10.1.66.0/24
  database: 10.1.128.0/24, 10.1.129.0/24, 10.1.130.0/24

prod    vpc_cidr = "10.2.0.0/16":
  public:   10.2.0.0/24,   10.2.1.0/24,   10.2.2.0/24
  private:  10.2.64.0/24,  10.2.65.0/24,  10.2.66.0/24
  database: 10.2.128.0/24, 10.2.129.0/24, 10.2.130.0/24
```

No overlaps anywhere → safe to peer any two environments or connect through a Transit Gateway.

---

### Internet Gateway (IGW)

A VPC on its own is completely isolated — resources inside it can talk to each other, but nothing can reach the internet and the internet can't reach them. The IGW is what connects the VPC to the public internet.

Think of it this way:

```
Without IGW:                      With IGW:

  ┌─── VPC ───┐                     ┌─── VPC ───┐
  │           │                     │           │
  │   EC2 ←──────── ✗ internet      │   EC2 ←──────── IGW ←──── internet
  │           │                     │           │
  └───────────┘                     └───────────┘

  Resources can only talk           Resources in public subnets
  to each other inside the VPC.     can now send/receive internet traffic.
```

**When do you actually need one?**

- Your ALB needs to receive traffic from users on the internet → needs IGW
- Your NAT Gateway needs to forward outbound requests from private subnets → NAT itself sits in a public subnet, which needs IGW
- A bastion host needs to be SSH'd into from outside → needs IGW

**What is a bastion host?**

A bastion host (also called a jump box) is a small, hardened EC2 instance that sits in a public subnet. Its only job is to be the single entry point for SSH access into your private network. You never SSH directly into your app servers or EKS nodes — you SSH into the bastion first, then hop from there to whatever private resource you need.

```
Your laptop
    │
    │  SSH (port 22)
    ▼
┌─────────────────────────────────────────────────┐
│  public subnet                                  │
│                                                 │
│   Bastion Host (EC2, tiny instance)             │
│   SG: allows port 22 from your IP only          │
└──────────────────────┬──────────────────────────┘
                       │
                       │  SSH hop
                       ▼
┌─────────────────────────────────────────────────┐
│  private subnet                                 │
│                                                 │
│   App server / EKS node / RDS                   │
│   SG: allows port 22 from bastion SG only       │
└─────────────────────────────────────────────────┘
```

The security model: your private instances have a security group rule that only allows SSH from the bastion's security group — not from the internet, not from your laptop directly. If someone compromises your laptop, they still can't reach your private servers without also compromising the bastion.

```hcl
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[0].id

  vpc_security_group_ids = [aws_security_group.bastion.id]
  key_name               = aws_key_pair.bastion.key_name

  metadata_options {
    http_tokens = "required"
  }

  tags = { Name = "bastion" }
}

resource "aws_security_group" "bastion" {
  name        = "bastion-sg"
  description = "Bastion host — SSH from known IPs only"
  vpc_id      = aws_vpc.main.id

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "bastion_ssh_in" {
  type              = "ingress"
  security_group_id = aws_security_group.bastion.id
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["YOUR_OFFICE_IP/32"] # never 0.0.0.0/0
}

resource "aws_security_group_rule" "bastion_egress" {
  type              = "egress"
  security_group_id = aws_security_group.bastion.id
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Private app servers only accept SSH from the bastion
resource "aws_security_group_rule" "app_ssh_from_bastion" {
  type                     = "ingress"
  security_group_id        = aws_security_group.app.id
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
}
```

> In modern setups, AWS Systems Manager Session Manager is often preferred over a bastion — it lets you shell into private instances without opening port 22 at all, using IAM for auth instead of SSH keys. But understanding the bastion pattern is important because it's still widely used and it illustrates the principle of a single controlled entry point.

**Bastion vs SSM Session Manager — which to use?**

SSM Session Manager replaces the bastion for most things. You shell into any instance using the AWS CLI with no port 22, no SSH keys, no public IP needed — just IAM permissions.

```bash
# No bastion, no SSH key, no open ports needed
aws ssm start-session --target i-0abc123def456
```

What SSM covers:
- Interactive shell access to EC2 instances
- Port forwarding to private resources (e.g. tunnel to RDS on port 5432)
- Full audit trail in CloudTrail — every session logged automatically
- Works on instances in fully private subnets with no IGW

```bash
# Port forward to a private RDS instance through SSM — no bastion needed
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.xxxx.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Where a bastion still makes sense:
- **Non-AWS resources** — connecting to on-premises servers, third-party systems, or anything SSM can't reach
- **Legacy instances** — old AMIs or OSes where the SSM agent isn't installed or supported
- **Compliance requirements** — some auditors specifically require a bastion as a documented control
- **SCP restrictions** — if your org's Service Control Policies block SSM in certain accounts

For a greenfield AWS setup in 2024+, skip the bastion and use SSM. You get better security (no open ports, IAM-based auth), better auditability, and less infrastructure to maintain.

**When don't you need one?**

- A fully internal setup — e.g. a private API only called by other services inside the VPC or via VPN
- Your EKS nodes, RDS, ElastiCache — these live in private subnets and should never have a route to the IGW directly

One IGW per VPC is all you ever need — it's not a bottleneck, it scales automatically and has no bandwidth limit.

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```

Attaching the IGW to the VPC isn't enough on its own — you also need a route table entry pointing `0.0.0.0/0` at it, and that route table must be associated with the subnet. That's what makes a subnet "public". Without the route table association, the IGW exists but traffic never gets directed to it.

---

### NAT Gateway

Sits in a public subnet and lets instances in private subnets initiate outbound connections (e.g. pulling OS updates, calling external APIs) without being reachable from the internet. Traffic flows out through the NAT, but nothing can initiate a connection inward.

> This is the correct pattern for EKS nodes — they need to pull container images and reach AWS APIs, but they should never accept inbound connections from the internet.

**Where should the NAT Gateway's `subnet_id` come from?**

You're right that writing `aws_subnet.public[0].id` directly is fragile — if the subnet is defined in a different file or module, Terraform can't resolve that reference and will error. The correct pattern is to pass subnet IDs through `outputs.tf` and `variables.tf` so each module only knows what it's explicitly given.

```
modules/
  networking/
    main.tf        ← defines VPC, subnets, IGW, NAT, route tables
    outputs.tf     ← exposes subnet IDs for other modules to consume
    variables.tf   ← accepts vpc_cidr, environment, etc.
  eks/
    main.tf        ← consumes subnet IDs via var.private_subnet_ids
    variables.tf   ← declares the variables it expects
```

`modules/networking/outputs.tf` — expose what other modules need:

```hcl
output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  value = aws_subnet.database[*].id
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

`envs/dev/main.tf` — wire modules together using outputs as inputs:

```hcl
module "networking" {
  source      = "../../modules/networking"
  vpc_cidr    = var.vpc_cidr
  environment = var.environment
}

module "eks" {
  source             = "../../modules/eks"
  vpc_id             = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
}

module "rds" {
  source              = "../../modules/rds"
  vpc_id              = module.networking.vpc_id
  database_subnet_ids = module.networking.database_subnet_ids
}
```

Now the NAT Gateway inside the networking module safely references its own subnet — no cross-module hardcoding:

```hcl
resource "aws_eip" "nat" {
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id  # safe — defined in the same module
  depends_on    = [aws_internet_gateway.main]
}
```

And the EKS module receives the subnet IDs as a variable — it never reaches into the networking module directly:

```hcl
# modules/eks/variables.tf
variable "private_subnet_ids" {
  description = "Private subnet IDs for EKS node groups"
  type        = list(string)
}

# modules/eks/main.tf
resource "aws_eks_node_group" "workers" {
  subnet_ids = var.private_subnet_ids  # passed in, not hardcoded
  ...
}
```

The rule: a module should only reference resources it owns. Anything from another module comes in through `variables.tf` and is wired up at the `envs/` level via outputs.

---

### Route Tables

Route tables are what actually wire subnets to the internet or NAT. Create one per tier and associate it with the matching subnets.

```hcl
# Public route table — sends internet-bound traffic to the IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route table — sends internet-bound traffic to the NAT Gateway
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = { Name = "private-rt" }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

---

## Security Groups vs NACLs — The Real Difference

This is one of the most misunderstood areas in AWS networking. The video framing of "NACLs = what can't get in, Security Groups = what can get in" is a rough shortcut that misses the functional reality.

---

### Security Groups (Instance-level firewall)

Security Groups are **stateful**. This is the key word.

Stateful means: if you allow a connection in one direction, the return traffic is automatically allowed — AWS tracks the connection state and permits the response without you needing an explicit rule for it.

- Attached to an **instance** (or ENI), not a subnet
- Only support **allow** rules — there is no deny. If no rule matches, traffic is dropped
- Evaluate all rules together — most permissive match wins
- Control both inbound and outbound traffic
- Default: all inbound blocked, all outbound allowed

**Functional use:** Security Groups are your primary, fine-grained access control. You define exactly which ports and sources are allowed to reach each resource. Because they're stateful, you only need to think about one direction — the response takes care of itself.

```
Client → [SG allows port 443 inbound] → EC2
EC2   → [return traffic auto-allowed]  → Client  ✅ no extra rule needed
```

Avoid inline `ingress`/`egress` blocks — they conflict with `aws_security_group_rule` resources and cause plan drift. Use separate rule resources instead.

```hcl
# ALB security group — faces the internet
resource "aws_security_group" "alb" {
  name        = "alb-sg"
  description = "ALB — accepts HTTPS from internet"
  vpc_id      = aws_vpc.main.id

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "alb_https_in" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "alb_http_in" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"] # redirect to HTTPS at listener level
}

resource "aws_security_group_rule" "alb_egress" {
  type              = "egress"
  security_group_id = aws_security_group.alb.id
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
}

# App security group — only accepts traffic from the ALB, never from the internet
resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "App servers — accepts traffic from ALB only"
  vpc_id      = aws_vpc.main.id

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "app_from_alb" {
  type                     = "ingress"
  security_group_id        = aws_security_group.app.id
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.alb.id # SG-to-SG — no hardcoded IPs
}

resource "aws_security_group_rule" "app_egress" {
  type              = "egress"
  security_group_id = aws_security_group.app.id
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

> Using `source_security_group_id` instead of a CIDR is the right pattern for internal rules. It means "allow traffic from anything attached to the ALB security group" — no hardcoded IPs, and it automatically stays correct as ALB IPs change.

---

### NACLs (Subnet-level firewall)

NACLs are **stateless**. This is the key word.

Stateless means: every packet is evaluated independently. AWS does not track connection state. If you allow inbound traffic on port 443, you must also explicitly allow the outbound return traffic (on ephemeral ports 1024–65535), otherwise the response is dropped.

- Attached to a **subnet**, applies to all resources in it
- Support both **allow** and **deny** rules — evaluated in order by rule number, first match wins
- Rules are numbered — lower number = higher priority
- Default NACL: allows all traffic in and out (permissive by default)
- Custom NACL: denies everything until you add rules

**Functional use:** NACLs are a coarse, subnet-wide layer. Their real value is as a **secondary defense** — blocking known bad IP ranges or isolating subnets from each other.

```
Client → [NACL allows port 443 inbound]           → Subnet → EC2
EC2   → [NACL must also allow 1024-65535 outbound] → Client
         ↑ if this rule is missing, the response is silently dropped
```

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id
}

# Block a known malicious IP range — lower rule number = higher priority
resource "aws_network_acl_rule" "block_bad_cidr" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 50
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "192.0.2.0/24"
  egress         = false
}

# Allow HTTPS inbound
resource "aws_network_acl_rule" "allow_https_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
  egress         = false
}

# Allow HTTP inbound (for redirect)
resource "aws_network_acl_rule" "allow_http_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 110
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
  egress         = false
}

# NACL is stateless — must explicitly allow return traffic on ephemeral ports
resource "aws_network_acl_rule" "allow_ephemeral_out" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
  egress         = true
}
```

---

### Side-by-Side Comparison

| Feature          | Security Group                         | NACL                                       |
| ---------------- | -------------------------------------- | ------------------------------------------ |
| Operates at      | Instance / ENI level                   | Subnet level                               |
| State            | Stateful (return traffic auto-allowed) | Stateless (both directions need rules)     |
| Rule types       | Allow only                             | Allow and Deny                             |
| Rule evaluation  | All rules, most permissive wins        | In order by rule number, first match wins  |
| Default behavior | Deny all inbound, allow all outbound   | Allow all (default NACL)                   |
| Primary use      | Fine-grained per-resource control      | Subnet-wide coarse filtering / deny lists  |

### How They Work Together (Defence in Depth)

Both layers are evaluated for every packet. A packet must pass both to reach its destination.

```
Internet
   │
   ▼
[NACL on public subnet]        ← subnet border, stateless
   │
   ▼
[Security Group on ALB]        ← instance border, stateful
   │
   ▼
ALB
   │
   ▼
[NACL on private subnet]       ← subnet border again
   │
   ▼
[Security Group on EC2]        ← instance border
   │
   ▼
EC2 / EKS Node
```

> Practical rule: do most of your work in Security Groups — they're stateful, easier to reason about, and operate at the right granularity. Use NACLs to block specific IP ranges or enforce hard subnet isolation on top.

---

## Amazon Route 53

Managed DNS service. Translates domain names to IP addresses and supports advanced routing.

- **Routing policies**: Simple, Weighted (A/B traffic split), Latency-based (route to nearest region), Failover (primary/secondary), Geolocation
- **Health checks**: Route 53 can monitor endpoints and automatically stop routing to unhealthy ones
- **Private hosted zones**: DNS that only resolves inside your VPC — use this for internal service discovery instead of hardcoding IPs

> Security note: Route 53 is often the first thing a user hits. Enable DNSSEC on public hosted zones to prevent DNS spoofing. Use private hosted zones for internal services so they're never resolvable from outside the VPC.

```hcl
resource "aws_route53_zone" "internal" {
  name = "internal.myapp.local"

  vpc {
    vpc_id = aws_vpc.main.id
  }
}

resource "aws_route53_record" "db" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "db.internal.myapp.local"
  type    = "CNAME"
  ttl     = 60
  records = [aws_db_instance.postgres.address]
}
```

---

## Elastic Load Balancing (ELB)

Distributes incoming traffic across multiple targets (EC2 instances, EKS pods, Lambda functions).

- **ALB (Application Load Balancer)** — Layer 7 (HTTP/HTTPS). Understands request content — can route based on path (`/api` vs `/web`), hostname, headers, or query strings. The right choice for microservices and Kubernetes ingress.
- **NLB (Network Load Balancer)** — Layer 4 (TCP/UDP). Doesn't inspect content, just routes packets. Extremely low latency, handles millions of requests per second. Use for non-HTTP workloads or when you need to preserve the client's source IP.

> Security note: your ALB should be the only thing in a public subnet. EC2 instances and EKS nodes sit in private subnets. The ALB's security group allows 443 from the internet; the app's security group only allows traffic from the ALB's security group — not from the internet directly.

```
Internet → ALB (public subnet, SG allows 0.0.0.0/0:443)
               │
               ▼
           EC2 / EKS (private subnet, SG allows only ALB SG on app port)
```

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  security_groups    = [aws_security_group.alb.id]

  drop_invalid_header_fields = true # reject malformed HTTP headers

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate.app.arn
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06" # TLS 1.3 preferred

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Always redirect HTTP to HTTPS — never serve plaintext
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

### ➡️ Next: [Security and IAM](./05-security-and-iam.md)
