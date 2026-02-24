# Compute Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Compute

Compute services provide the processing power required to run applications.

## Prerequisites

- [Kubernetes](../02-container-orchestration/01-kubernetes.md) — EKS is the managed Kubernetes service covered here
- [Deployment Automation](../03-deployment-automation/01-argocd.md) — understand what runs on top of this infrastructure

## Amazon EC2 (Elastic Compute Cloud)

Provides resizable compute capacity (Virtual Machines) in the cloud.

- **Instance Types**: Optimized for different use cases (General Purpose, Compute Optimized, Memory Optimized, Storage Optimized).
- **Pricing Models**:
  - **On-Demand**: Pay by the second with no long-term commitment.
  - **Reserved Instances (RI)**: Commit to 1 or 3 years for significant discounts.
  - **Spot Instances**: Bid on unused EC2 capacity at steep discounts (up to 90%).
    - _Note on Interruption_: AWS can "interrupt" (reclaim) a Spot Instance with only a 2-minute warning if they need the capacity back for full-paying customers, or if the market price exceeds your bid.
    - _Best for_: Stateless, fault-tolerant workloads (e.g., batch processing, background workers, containers) where it's okay if a server suddenly shuts down.
  - **Dedicated Hosts**: Physical servers dedicated to your use (good for compliance).
- **AMI (Amazon Machine Image)**: Template containing the OS and software setup needed to launch an instance.

## Serverless Compute

- **AWS Lambda**: Run code without provisioning servers. Pay only for the compute time consumed (measured in milliseconds). Event-driven (e.g., triggered by S3, API Gateway).
- **AWS Fargate**: Serverless compute engine for containers. You don't have to provision, configure, or scale clusters of virtual machines to run containers.

## Containers

- **Amazon ECS (Elastic Container Service)**: Highly scalable, high-performance container orchestration service that supports Docker.
- **Amazon EKS (Elastic Kubernetes Service)**: Managed Kubernetes service for running Kubernetes on AWS without installing and operating your own control plane.


---

## Terraform Examples

The examples below follow a natural progression — from launching a single VM, to scaling it, to going serverless, to running a full managed Kubernetes cluster.

---

### Step 1 — Look Up an AMI First

Before launching any EC2-based resource you need an AMI ID. Never hardcode it — IDs differ per region and rotate when AWS releases updates. Use the `aws_ami` data source instead.

```hcl
# Latest Amazon Linux 2023
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Latest Ubuntu 24.04 LTS
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical's official AWS account ID

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}
```

Where to find AMI values manually:
- AWS Console → EC2 → AMIs → filter "Public images"
- AWS CLI: `aws ec2 describe-images --owners amazon --filters "Name=name,Values=al2023-ami-*" --query 'Images[*].[ImageId,Name]'`
- Each OS vendor publishes their official owner ID (Canonical = `099720109477`, Amazon = `amazon`)

---

### Step 2 — Launch a Single EC2 Instance

The simplest compute unit. This is your baseline — a single VM in a private subnet with IMDSv2 enforced, an encrypted root volume, and detailed monitoring on.

```hcl
resource "aws_instance" "app_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  subnet_id              = aws_subnet.private.id
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.app.name

  ebs_optimized = true # better EBS throughput

  # Enforce IMDSv2 — prevents SSRF-based metadata attacks
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  # Encrypt the root volume
  root_block_device {
    encrypted   = true
    volume_type = "gp3"
    volume_size = 20
  }

  monitoring = true # detailed CloudWatch metrics

  tags = { Name = "app-server" }
}
```

---

### Step 3 — Choose a Pricing Model

Once you know how to launch an instance, you can control what you pay for it.

**On-Demand** — the default, no extra config needed. Pay by the second.

**Spot** — up to 90% cheaper, but AWS can reclaim the instance with 2 min notice. Good for stateless/batch workloads.

```hcl
resource "aws_instance" "worker" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  instance_market_options {
    market_type = "spot"
    spot_options {
      max_price                      = "0.05"       # max $/hr you'll pay
      spot_instance_type             = "persistent"  # keep trying if interrupted
      instance_interruption_behavior = "stop"
    }
  }
}
```

**Reserved Instances** — not managed in Terraform. You purchase them in the AWS Console; AWS silently applies the discount to any matching On-Demand instance already running.

---

### Step 4 — Scale It with a Launch Template + Auto Scaling Group

A single instance isn't resilient. The next step is wrapping it in an ASG so it self-heals and scales. Always use a Launch Template — Launch Configurations are deprecated.

#### How Auto Scaling Works

An ASG constantly watches the health and load of your instances and adjusts the count between `min_size` and `max_size`. There are three things that can trigger a scaling action:

**Scale Out (add instances)**
- A CloudWatch alarm fires — e.g. average CPU > 70% for 2 consecutive minutes
- A scheduled action triggers — e.g. pre-warm every weekday at 08:00
- A target tracking policy detects the metric drifting above the target — e.g. average request count per instance > 1000
- An instance fails its health check and gets terminated — ASG replaces it to maintain `desired_capacity`

**Scale In (remove instances)**
- A CloudWatch alarm fires in the opposite direction — e.g. average CPU < 30% for 10 minutes
- Target tracking policy detects the metric is well below target
- A scheduled action reduces desired capacity during off-hours

**Self-Heal (replace a broken instance)**
- The ELB health check marks an instance unhealthy (failed HTTP check)
- Or the EC2 status check fails (OS-level issue)
- ASG terminates it and launches a replacement automatically — no manual intervention needed

#### The Scaling Process Step by Step

```
1. CloudWatch collects metrics (CPU, memory, request count, custom metrics)
         │
         ▼
2. Alarm threshold breached (e.g. CPU > 70% for 2 periods of 60s)
         │
         ▼
3. Alarm triggers the Auto Scaling Policy
         │
         ├── Scale Out → ASG increases desired_capacity by N
         │                    │
         │                    ▼
         │              Launches new instance(s) using the Launch Template
         │                    │
         │                    ▼
         │              Instance passes health check → joins the target group
         │
         └── Scale In  → ASG decreases desired_capacity by N
                              │
                              ▼
                        Picks instance to terminate (default: oldest launch config)
                              │
                              ▼
                        Deregisters from load balancer → waits for connections to drain
                              │
                              ▼
                        Terminates instance
```

**Cooldown period** — after a scale-out, the ASG waits (default 300s) before evaluating another alarm. This prevents it from launching 10 instances because a spike lasted 30 seconds.

**Warm-up period** — new instances take time to boot and start serving traffic. Target tracking policies respect a warm-up window so new instances aren't counted in the metric average until they're ready.

#### Terraform: Launch Template + ASG

```hcl
# Reusable template — defines how every instance in the ASG is launched
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required" # IMDSv2
    http_put_response_hop_limit = 1
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }
}

# ASG with mixed On-Demand + Spot — most production-friendly pattern
resource "aws_autoscaling_group" "app" {
  desired_capacity    = 4
  min_size            = 2
  max_size            = 10
  vpc_zone_identifier = aws_subnet.private[*].id
  health_check_type   = "ELB" # use ELB health checks, not just EC2 status

  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 1   # always keep 1 on-demand
      on_demand_percentage_above_base_capacity = 25  # 25% on-demand, 75% spot
      spot_allocation_strategy                 = "capacity-optimized"
    }

    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
        version            = "$Latest"
      }

      # Fallback instance types if primary isn't available
      override { instance_type = "t3.medium" }
      override { instance_type = "t3.large"  }
    }
  }

  # Rolling replacement when the launch template changes
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }
}
```

#### Terraform: Scaling Policies

The ASG alone doesn't scale automatically — you need to attach policies that define the trigger and the action.

```hcl
# Target tracking — simplest and recommended for most cases.
# AWS automatically creates the CloudWatch alarms for you.
# This keeps average CPU at 60% by adding/removing instances as needed.
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value     = 60.0  # scale out when avg CPU > 60%, scale in when well below
    disable_scale_in = false # allow automatic scale-in (set true to only scale out)
  }
}

# Step scaling — more control, you define the alarms and the step sizes.
# Useful when you want different responses at different severity levels.
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2       # must breach for 2 consecutive periods
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60      # each period is 60 seconds
  statistic           = "Average"
  threshold           = 70
  treat_missing_data  = "notBreaching"

  dimensions = { AutoScalingGroupName = aws_autoscaling_group.app.name }

  alarm_actions = [aws_autoscaling_policy.scale_out.arn]
  ok_actions    = [aws_autoscaling_policy.scale_in.arn]
}

resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "StepScaling"
  adjustment_type        = "ChangeInCapacity"

  step_adjustment {
    scaling_adjustment          = 1   # add 1 instance when CPU 70–85%
    metric_interval_lower_bound = 0
    metric_interval_upper_bound = 15
  }

  step_adjustment {
    scaling_adjustment          = 3   # add 3 instances when CPU > 85%
    metric_interval_lower_bound = 15
  }
}

resource "aws_autoscaling_policy" "scale_in" {
  name                   = "scale-in"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "StepScaling"
  adjustment_type        = "ChangeInCapacity"

  step_adjustment {
    scaling_adjustment          = -1  # remove 1 instance when CPU drops back below threshold
    metric_interval_upper_bound = 0
  }
}

# Scheduled scaling — pre-warm before known traffic spikes (e.g. business hours)
resource "aws_autoscaling_schedule" "scale_up_morning" {
  scheduled_action_name  = "scale-up-morning"
  autoscaling_group_name = aws_autoscaling_group.app.name
  recurrence             = "0 7 * * MON-FRI" # 07:00 UTC, weekdays
  desired_capacity       = 8
  min_size               = 4
  max_size               = 20
}

resource "aws_autoscaling_schedule" "scale_down_night" {
  scheduled_action_name  = "scale-down-night"
  autoscaling_group_name = aws_autoscaling_group.app.name
  recurrence             = "0 20 * * MON-FRI" # 20:00 UTC, weekdays
  desired_capacity       = 2
  min_size               = 2
  max_size               = 10
}
```

> Use target tracking for general workloads, step scaling when you need fine-grained control over how aggressively you respond to spikes, and scheduled scaling when you know your traffic pattern in advance (e.g. business hours, batch jobs).

For larger Spot pools across multiple subnets, use a Spot Fleet instead:

```hcl
resource "aws_spot_fleet_request" "workers" {
  iam_fleet_role      = aws_iam_role.spot_fleet.arn
  target_capacity     = 4
  allocation_strategy = "capacityOptimized" # fewer interruptions than lowestPrice

  launch_template_config {
    launch_template_specification {
      id      = aws_launch_template.app.id
      version = aws_launch_template.app.latest_version
    }

    overrides {
      instance_type = "t3.medium"
      subnet_id     = aws_subnet.private[0].id
    }

    overrides {
      instance_type = "t3.large"
      subnet_id     = aws_subnet.private[1].id
    }
  }
}
```

---

### Step 5 — Go Serverless with Lambda

When you don't want to manage instances at all, Lambda runs your code on-demand. You pay only for execution time (per ms), and it scales automatically.

```hcl
resource "aws_lambda_function" "handler" {
  function_name = "my-handler"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "nodejs20.x"
  handler       = "index.handler"

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  # Run inside a VPC to access private resources (RDS, ElastiCache)
  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }

  # Enable X-Ray tracing for request visibility
  tracing_config {
    mode = "Active"
  }

  # Cap concurrency — prevents one function from consuming the account limit
  reserved_concurrent_executions = 100

  environment {
    variables = {
      ENV = "production"
    }
  }
}
```

---

### Step 6 — Run Containers at Scale with EKS

When your workloads are containerized and you need orchestration, EKS is the managed Kubernetes option. Terraform provisions the control plane and node groups; ArgoCD and Karpenter take it from there.

```hcl
# The control plane
resource "aws_eks_cluster" "main" {
  name     = "main-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = false # no public API server
  }

  # Ship all control plane logs to CloudWatch
  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}

# Launch template for worker nodes — reuses the same best practices as Step 4
resource "aws_launch_template" "workers" {
  name_prefix = "eks-workers-"

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required" # IMDSv2
    http_put_response_hop_limit = 1
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }
}

# Managed node group — Terraform handles the initial pool of nodes
resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = aws_subnet.private[*].id

  launch_template {
    id      = aws_launch_template.workers.id
    version = aws_launch_template.workers.latest_version
  }

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 5
  }

  # Limit disruption during rolling node upgrades
  update_config {
    max_unavailable = 1
  }
}
```

> From here, Karpenter takes over dynamic node scaling based on actual pod demand. See [Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md) for the full setup.

---

### ➡️ Next: [Storage Services](./02-storage.md)
