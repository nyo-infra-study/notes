# Storage Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Storage

AWS provides storage options tailored for object, block, and file storage.

## Prerequisites

- [Compute Services](./01-compute.md) — storage is typically attached to or accessed by compute resources

## Amazon S3 (Simple Storage Service)
Object storage service offering industry-leading scalability, data availability, security, and performance.
*   **Buckets & Objects**: Files are stored as "objects" inside "buckets" (must have globally unique names).
*   **Storage Classes**:
    *   **S3 Standard**: High availability, frequently accessed data.
    *   **S3 Intelligent-Tiering**: Automatically moves data to the most cost-effective tier.
    *   **S3 Standard-IA (Infrequent Access)**: Lower cost storage for data accessed less frequently but needs rapid access when needed.
    *   **S3 Glacier Flexible Retrieval**: Low-cost storage for archiving (retrieval takes minutes to hours).
    *   **S3 Glacier Deep Archive**: Lowest cost storage class (retrieval takes >12 hours).

## Amazon EBS (Elastic Block Store)
High-performance block level storage volumes designed for use with EC2 instances.
*   Think of it as a physical hard drive you attach to your VM.
*   Can only be attached to ONE EC2 instance at a time (usually). Must be in the same AZ as the EC2 instance.

## Amazon EFS (Elastic File System)
A serverless, set-and-forget elastic file system.
*   Uses NFS (Network File System) protocol.
*   Can be mounted to MULTIPLE EC2 instances simultaneously.
*   Automatically scales as you add or remove files.


---

### ➡️ Next: [Database Services](./03-database.md)
