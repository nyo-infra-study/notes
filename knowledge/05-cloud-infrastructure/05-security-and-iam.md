# Security and Identity

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Security and IAM

AWS operates on a Shared Responsibility Model: AWS is responsible for security *OF* the cloud (infrastructure, hardware), and the customer is responsible for security *IN* the cloud (data, OS, firewall config).

## Prerequisites

- [Networking Services](./04-networking.md) — security groups and NACLs are part of VPC networking
- [Compute Services](./01-compute.md) — IAM roles are commonly attached to EC2 instances and EKS service accounts

---

## AWS IAM (Identity and Access Management)

Securely manage access to AWS services and resources.

- **Users** — individuals or applications needing access.
- **Groups** — collections of users. Attach policies to the group, not individual users.
- **Roles** — assumable identities with specific permissions. Used by EC2 instances, Lambda functions, EKS pods — anything that needs to call AWS APIs without hardcoding credentials.
- **Policies** — JSON documents defining exactly what is allowed or denied (Actions, Resources, Conditions).

The principle of least privilege applies here: every role and policy should grant only the minimum permissions needed, scoped to the exact resources it needs to touch.

```hcl
resource "aws_iam_role" "app" {
  name        = "app-role"
  path        = "/app/"
  description = "Role assumed by the app EC2 instances"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = { Name = "app-role" }
}

resource "aws_iam_policy" "s3_read" {
  name        = "s3-read-policy"
  path        = "/app/"
  description = "Allows read access to the app assets bucket"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        aws_s3_bucket.assets.arn,
        "${aws_s3_bucket.assets.arn}/*"
      ]
    }]
  })
}

resource "aws_iam_role_policy_attachment" "app_s3" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.s3_read.arn
}

resource "aws_iam_instance_profile" "app" {
  name = "app-instance-profile"
  role = aws_iam_role.app.name
}
```

---

## AWS KMS (Key Management Service)

Managed service to create and control encryption keys used to encrypt your data. Integrated with most AWS services — S3, EBS, RDS, Secrets Manager, CloudWatch Logs.

- Keys never leave KMS unencrypted — your app never handles the raw key material
- Every encrypt/decrypt call is logged in CloudTrail — full audit trail of who accessed what data and when
- `enable_key_rotation = true` rotates the key material annually without changing the key ID or re-encrypting your data

```hcl
resource "aws_kms_key" "app" {
  description             = "App encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "Enable IAM policies"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      }
    ]
  })

  tags = { Name = "app-key" }
}

resource "aws_kms_alias" "app" {
  name          = "alias/app-key"
  target_key_id = aws_kms_key.app.key_id
}
```

---

## AWS Shield and AWS WAF

- **AWS Shield** — managed DDoS protection. Shield Standard is on automatically for all AWS accounts at no cost. Shield Advanced adds 24/7 response team and cost protection for large-scale attacks.
- **AWS WAF (Web Application Firewall)** — protects web apps from common exploits like SQL injection and Cross-Site Scripting by defining rules that inspect HTTP requests before they reach your ALB or API Gateway.

WAF sits in front of your ALB. Requests that match a rule are blocked before they ever hit your application.

```hcl
resource "aws_wafv2_web_acl" "app" {
  name  = "app-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "AppWAF"
    sampled_requests_enabled   = true
  }
}

resource "aws_wafv2_web_acl_association" "app" {
  resource_arn = aws_lb.app.arn
  web_acl_arn  = aws_wafv2_web_acl.app.arn
}
```

---

### ➡️ Next: [Management and Governance](./06-management-governance.md)
