# Networking Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Networking

Networking allows you to isolate infrastructure, scale efficiently, and route traffic.

## Prerequisites

- [Compute Services](./01-compute.md) — EC2 instances and EKS clusters live inside VPCs

## Amazon VPC (Virtual Private Cloud)
A logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define.
*   **Subnets**: Divide your VPC IP range into smaller chunks.
    *   **Public Subnet**: Has a route to the Internet Gateway.
    *   **Private Subnet**: No direct internet access.
*   **Internet Gateway (IGW)**: Allows access to the internet from the VPC.
*   **NAT Gateway**: Allows instances in a private subnet to connect to the internet (e.g., for updates) while preventing external connections.
*   **Security Groups**: Stateful firewalls acting at the *instance* level.
*   **NACLs (Network Access Control Lists)**: Stateless firewalls acting at the *subnet* level.

## Amazon Route 53
Highly available and scalable cloud DNS (Domain Name System) web service.
*   Used to route users to internet applications by translating names like www.example.com into IP addresses (e.g., 192.0.2.1).
*   Supports health checks and complex routing policies (Weighted, Latency, Failover, Geolocation).

## Elastic Load Balancing (ELB)
Automatically distributes incoming application traffic across multiple targets.
*   **ALB (Application Load Balancer)**: Layer 7 (HTTP/HTTPS). Great for microservices.
*   **NLB (Network Load Balancer)**: Layer 4 (TCP/UDP). Ultra-high performance, ultra-low latency.


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
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
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
  subnet_id     = aws_subnet.public[0].id
  depends_on    = [aws_internet_gateway.main]
}
```

### Security Group

Avoid inline `ingress`/`egress` blocks — they conflict with `aws_security_group_rule` resources and cause plan drift. Use separate rules instead.

```hcl
resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "App server security group" # required — never leave this as default
  vpc_id      = aws_vpc.main.id

  # No inline ingress/egress — use aws_security_group_rule resources below
  lifecycle {
    create_before_destroy = true # prevents downtime on SG replacement
  }
}

resource "aws_security_group_rule" "app_https_in" {
  type              = "ingress"
  security_group_id = aws_security_group.app.id
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
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

### Application Load Balancer

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  security_groups    = [aws_security_group.alb.id]

  drop_invalid_header_fields = true # reject malformed HTTP headers

  # Ship access logs to S3 for auditing
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

# Redirect HTTP → HTTPS
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
