# Security and Identity

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Security and IAM

AWS operates on a Shared Responsibility Model: AWS is responsible for security *OF* the cloud (infrastructure, hardware), and the customer is responsible for security *IN* the cloud (data, OS, firewall config).

## Prerequisites

- [Networking Services](./04-networking.md) — security groups and NACLs are part of VPC networking
- [Compute Services](./01-compute.md) — IAM roles are commonly attached to EC2 instances and EKS service accounts

## AWS IAM (Identity and Access Management)
Securely manage access to AWS services and resources.
*   **Users**: Individuals or applications needing access.
*   **Groups**: Collections of users.
*   **Roles**: Assumable identities with specific permissions (often used by EC2 instances to access S3 seamlessly without hardcoding credentials).
*   **Policies**: JSON documents attached to identities defining exactly what they can or cannot do (Allow/Deny, Actions, Resources).

## AWS KMS (Key Management Service)
Managed service to easily create and control the encryption keys used to encrypt your data.
*   Integrated with most AWS services (S3, EBS, RDS).

## AWS Shield and AWS WAF
*   **AWS Shield**: Managed Distributed Denial of Service (DDoS) protection service. Shield Standard is enabled automatically for all AWS customers at no extra cost.
*   **AWS WAF (Web Application Firewall)**: Helps protect web apps from common web exploits (e.g., SQL injection, Cross-Site Scripting) by defining customizable web security rules.


---

## Terraform Examples

### IAM Role with Policy

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

# Attach the instance profile so EC2 can assume the role
resource "aws_iam_instance_profile" "app" {
  name = "app-instance-profile"
  role = aws_iam_role.app.name
}
```

### KMS Key

```hcl
resource "aws_kms_key" "app" {
  description             = "App encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true # rotate annually — required for compliance

  # Explicit key policy — never rely solely on the default policy
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM policies"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
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

### WAF Web ACL

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
```

---

### ➡️ Next: [Management and Governance](./06-management-governance.md)
