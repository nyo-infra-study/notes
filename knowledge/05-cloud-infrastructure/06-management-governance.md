# Management and Governance

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Management and Governance

Tools to monitor, manage, and optimize your environment.

## Prerequisites

- [Compute Services](./01-compute.md) — CloudWatch monitors EC2, Lambda, and EKS metrics
- [Observability](../04-observability/01-plg-stack.md) — CloudWatch complements the PLG stack for AWS-native monitoring

## Amazon CloudWatch
Monitoring and observability service built for DevOps, developers, and IT managers.
*   **Metrics**: Data about the performance of your systems (e.g., CPU utilization).
*   **Logs**: Centralized location for log files from your resources (e.g., EC2, Lambda).
*   **Alarms**: Watch a metric and trigger an action (e.g., SNS notification, Auto Scaling) if a threshold is breached.

## AWS CloudTrail
Enables governance, compliance, operational auditing, and risk auditing.
*   Records API calls and user activity (Who did what, when, and from where?).

## AWS Organizations
Allows you to centrally manage and govern multiple accounts.
*   Consolidated billing.
*   Service Control Policies (SCPs) to restrict account privileges globally.

## AWS Cost Explorer
Lets you visualize, understand, and manage your AWS costs and usage over time. Helps forecast future costs and find saving opportunities.


---

## Terraform Examples

### CloudWatch Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80

  # Don't fire alarms on missing data — treat as not breaching
  treat_missing_data = "notBreaching"

  dimensions = {
    InstanceId = aws_instance.app_server.id
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn] # notify when it recovers too
}
```

### CloudWatch Log Group

```hcl
resource "aws_cloudwatch_log_group" "app" {
  name              = "/app/production"
  retention_in_days = 30        # don't keep logs forever — costs add up
  kms_key_id        = aws_kms_key.app.arn

  tags = { Name = "app-logs" }
}
```

### AWS Budget Alert

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-budget"
  budget_type  = "COST"
  limit_amount = "500"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["[email]"]
  }
}
```

---

### ➡️ Next: [Terraform + Kubernetes + GitOps](./07-terraform-kubernetes-gitops.md)
