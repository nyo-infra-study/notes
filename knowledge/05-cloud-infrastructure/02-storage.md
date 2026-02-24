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

## Terraform Examples

### S3 Bucket

```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "my-app-assets"
}

# Block all public access — must be explicit since AWS doesn't enforce it by default
resource "aws_s3_bucket_public_access_block" "assets" {
  bucket                  = aws_s3_bucket.assets.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms" # prefer KMS over AES256 for audit trail
      kms_master_key_id = aws_kms_key.app.arn
    }
    bucket_key_enabled = true # reduces KMS API call costs
  }
}
```

### EBS Volume

```hcl
# Derive AZ from the instance rather than hardcoding it
resource "aws_ebs_volume" "data" {
  availability_zone = aws_instance.app_server.availability_zone
  size              = 20 # GB
  type              = "gp3"
  encrypted         = true
  kms_key_id        = aws_kms_key.app.arn

  tags = { Name = "app-data" }
}

resource "aws_volume_attachment" "data" {
  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.app_server.id
}
```

### EFS File System

```hcl
resource "aws_efs_file_system" "shared" {
  encrypted        = true
  kms_key_id       = aws_kms_key.app.arn
  performance_mode = "generalPurpose" # use "maxIO" only for highly parallel workloads
  throughput_mode  = "bursting"

  # Automatically move files not accessed in 30 days to cheaper IA storage
  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }

  tags = { Name = "shared-fs" }
}

resource "aws_efs_mount_target" "shared" {
  for_each = toset(aws_subnet.private[*].id)

  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = each.value
  security_groups = [aws_security_group.efs.id]
}
```

---

### ➡️ Next: [Database Services](./03-database.md)
