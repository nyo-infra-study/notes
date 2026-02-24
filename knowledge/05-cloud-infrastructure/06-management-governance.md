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

### ➡️ Next: [Back to Cloud Infrastructure](./README.md)
