# Networking Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Networking

Networking in AWS is how you isolate resources, control traffic flow, and expose services safely. Every decision here has a security implication — this note covers both the technical mechanics and the functional reasoning behind each layer.

## Prerequisites

- [Compute Services](./01-compute.md) — EC2 instances and EKS clusters live inside VPCs
- [Security and IAM](./05-security-and-iam.md) — IAM controls who can call AWS APIs; networking controls what traffic can reach your resources

---

## Amazon VPC (Virtual Private Cloud)

A VPC is a logically isolated network you define inside AWS. Nothing gets in or out unless you explicitly allow it. Think of it as your private data center network, but software-defined.

### Subnets

Subnets divide your VPC's IP range into smaller segments. The key distinction is whether a subnet has a route to the internet.

- **Public Subnet** — has a route to an Internet Gateway. Resources here can be reached from the internet (if their security group allows it). Use for load balancers, bastion hosts.
- **Private Subnet** — no route to the internet. Resources here are unreachable from outside. Use for app servers, databases, EKS nodes — anything that shouldn't be directly exposed.

> Security principle: put everything in private subnets by default. Only move something to a public subnet if it explicitly needs to be internet-facing. Your EC2 instances, RDS, and EKS nodes should never be in a public subnet.

### Internet Gateway (IGW)

Attaches to a VPC and enables two-way communication with the internet. A subnet is only "public" if it has a route table entry pointing to an IGW.

### NAT Gateway

Sits in a public subnet and lets instances in private subnets initiate outbound connections (e.g. pulling OS updates, calling external APIs) without being reachable from the internet. Traffic flows out through the NAT, but nothing can initiate a connection inward.

> This is the correct pattern for EKS nodes — they need to pull container images and reach AWS APIs, but they should never accept inbound connections from the internet.

---

## Security Groups vs NACLs — The Real Difference

This is one of the most misunderstood areas in AWS networking. The video framing of "NACLs = what can't get in, Security Groups = what can get in" is a rough shortcut that misses the functional reality.

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

### NACLs (Subnet-level firewall)

NACLs are **stateless**. This is the key word.

Stateless means: every packet is evaluated independently. AWS does not track connection state. If you allow inbound traffic on port 443, you must also explicitly allow the outbound return traffic (on ephemeral ports 1024–65535), otherwise the response is dropped.

- Attached to a **subnet**, applies to all resources in it
- Support both **allow** and **deny** rules — evaluated in order by rule number, first match wins
- Rules are numbered — lower number = higher priority
- Default NACL: allows all traffic in and out (permissive by default)
- Custom NACL: denies everything until you add rules

**Functional use:** NACLs are a coarse, subnet-wide layer. Because they're stateless and require explicit rules for both directions, they're more complex to manage. Their real value is as a **secondary defense** — blocking known bad IP ranges, isolating subnets from each other, or adding an extra layer of protection that Security Groups alone can't provide.

```
Client → [NACL allows port 443 inbound]  → Subnet → EC2
EC2   → [NACL must also allow 1024-65535 outbound] → Client
         ↑ if this rule is missing, the response is silently dropped
```

### Side-by-Side Comparison

| Feature              | Security Group                        | NACL                                      |
| -------------------- | ------------------------------------- | ----------------------------------------- |
| Operates at          | Instance / ENI level                  | Subnet level                              |
| State                | Stateful (return traffic auto-allowed)| Stateless (both directions need rules)    |
| Rule types           | Allow only                            | Allow and Deny                            |
| Rule evaluation      | All rules, most permissive wins       | In order by rule number, first match wins |
| Default behavior     | Deny all inbound, allow all outbound  | Allow all (default NACL)                  |
| Primary use          | Fine-grained per-resource control     | Subnet-wide coarse filtering / deny lists |

### How They Work Together (Defence in Depth)

Both layers are evaluated for every packet. A packet must pass both to reach its destination.

```
Internet
   │
   ▼
[NACL on public subnet]        ← subnet border, stateless
   │  checks inbound rules
   ▼
[Security Group on ALB]        ← instance border, stateful
   │  checks inbound rules
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

> Practical rule: do most of your work in Security Groups — they're stateful, easier to reason about, and operate at the right granularity. Use NACLs to block specific IP ranges (e.g. known malicious CIDRs) or to enforce hard subnet isolation that you don't want to rely on Security Groups alone for.

---

## Amazon Route 53

Managed DNS service. Translates domain names to IP addresses and supports advanced routing.

- **Routing policies**: Simple, Weighted (A/B traffic split), Latency-based (route to nearest region), Failover (primary/secondary), Geolocation
- **Health checks**: Route 53 can monitor endpoints and automatically stop routing to unhealthy ones
- **Private hosted zones**: DNS that only resolves inside your VPC — use this for internal service discovery instead of hardcoding IPs

> Security note: Route 53 is often the first thing a user hits. Enable DNSSEC on public hosted zones to prevent DNS spoofing. Use private hosted zones for internal services so they're never resolvable from outside the VPC.

---

## Elastic Load Balancing (ELB)

Distributes incoming traffic across multiple targets (EC2 instances, EKS pods, Lambda functions).

- **ALB (Application Load Balancer)** — Layer 7 (HTTP/HTTPS). Understands request content — can route based on path (`/api` vs `/web`), hostname, headers, or query strings. The right choice for microservices and Kubernetes ingress.
- **NLB (Network Load Balancer)** — Layer 4 (TCP/UDP). Doesn't inspect content, just routes packets. Extremely low latency, handles millions of requests per second. Use for non-HTTP workloads or when you need to preserve the client's source IP.

> Security note: your ALB should be the only thing in a public subnet. EC2 instances and EKS nodes sit in private subnets. The ALB's security group allows 443 from the internet; the app's security group only allows traffic from the ALB's security group — not from the internet directly. This way even if someone finds your instance's IP, they can't reach it.

```
Internet → ALB (public subnet, SG allows 0.0.0.0/0:443)
               │
               ▼
           EC2 / EKS (private subnet, SG allows only ALB SG on app port)
```

---

## Terraform Examples

### VPC with Public and Private Subnets

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-${count.index}" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id # NAT lives in public subnet
  depends_on    = [aws_internet_gateway.main]
}
```

### Security Groups

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
  source_security_group_id = aws_security_group.alb.id # SG-to-SG reference, not a CIDR
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

### NACL — Blocking a Known Bad CIDR

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id
}

# Block a known malicious IP range first (lower rule number = higher priority)
resource "aws_network_acl_rule" "block_bad_cidr" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 50
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "192.0.2.0/24" # example bad range
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
  egress         = true # outbound rule for return traffic
}
```

### Application Load Balancer

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  security_groups    = [aws_security_group.alb.id]

  drop_invalid_header_fields = true # reject malformed HTTP headers

  # Ship access logs to S3 for auditing and incident response
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
