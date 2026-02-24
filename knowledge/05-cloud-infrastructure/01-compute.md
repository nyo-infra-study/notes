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

### ➡️ Next: [Storage Services](./02-storage.md)
