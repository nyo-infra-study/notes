# Introduction to AWS

AWS (Amazon Web Services) is the world's most comprehensive and broadly adopted cloud platform. This guide covers the core foundational services.

## Cloud Computing Concepts

- **CapEx vs. OpEx**: Cloud computing shifts your spending from Capital Expenditure (buying servers upfront) to Operational Expenditure (pay for what you use).
- **Deployment Models**: Public Cloud (AWS, Azure), Private Cloud (on-premises), Hybrid (mix of both).
- **Service Models**:
  - **IaaS (Infrastructure as a Service)**: You manage the OS, apps, data (e.g., EC2).
  - **PaaS (Platform as a Service)**: You manage the apps, cloud provider manages underlying OS/servers (e.g., Elastic Beanstalk).
  - **SaaS (Software as a Service)**: Complete software solution managed by the provider.

## AWS Global Infrastructure

- **Regions**: Physical locations around the world where data centers cluster. Entirely independent.
- **Availability Zones (AZs)**: One or more discrete data centers within a region with redundant power, networking, and connectivity. They are built to be highly available and fault-tolerant. (e.g., us-east-1a, us-east-1b).
- **Edge Locations**: Points of presence used by Amazon CloudFront (CDN) to cache content closer to the end-users to reduce latency.

## AWS Core Services Guides

Explore the core AWS services in detail through the following guides:

- **[02. Compute Services](./02-compute.md)**: Details processing power solutions, including virtual machines (EC2), serverless (Lambda), and containers (ECS/EKS).
- **[03. Storage Services](./03-storage.md)**: Explains the different storage layers available, comparing object (S3), block (EBS), and file (EFS) storage.
- **[04. Database Services](./04-database.md)**: An overview of purpose-built databases ranging from relational (RDS, Aurora) to NoSQL (DynamoDB) and caching (ElastiCache).
- **[05. Networking Services](./05-networking.md)**: Covers how to isolate and connect your cloud infrastructure using VPCs, subnets, Route 53 DNS, and Load Balancers.
- **[06. Security and IAM](./06-security-and-iam.md)**: Explains the Shared Responsibility Model, controlling access with IAM, encryption (KMS), and threat protection (WAF/Shield).
- **[07. Management and Governance](./07-management-governance.md)**: Tools to operate, monitor, and cost-optimize your cloud environment via CloudWatch, CloudTrail, and Organizations.


---

### ➡️ Next: [Compute Services](./02-compute.md)
