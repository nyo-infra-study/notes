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

### ➡️ Next: [Security and IAM](./05-security-and-iam.md)
