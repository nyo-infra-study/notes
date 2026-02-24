# Database Services

[Knowledge Base](../README.md) > [Cloud Infrastructure](./README.md) > Database

AWS offers purpose-built databases for various application needs.

## Prerequisites

- [Compute Services](./01-compute.md) — applications running on EC2/EKS connect to these databases
- [Networking Services](./04-networking.md) — databases live in VPCs and require proper subnet/security group configuration

## Amazon RDS (Relational Database Service)
Managed service for setting up, operating, and scaling a relational database.
*   Supports engines: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server.
*   Features automated backups, patching, and scaling.
*   **Multi-AZ**: Synchronous replication to another AZ for disaster recovery (High Availability).
*   **Read Replicas**: Asynchronous replication to handle read-heavy workloads (Scalability).

## Amazon Aurora
MySQL and PostgreSQL-compatible relational database built for the cloud.
*   Up to 5x faster than standard MySQL, 3x faster than standard PostgreSQL.
*   Storage automatically grows up to 128 TB.

## Amazon DynamoDB
Fully managed key-value and document database (NoSQL).
*   Delivers single-digit millisecond performance at any scale.
*   Serverless: No servers to provision, patch, or manage.

## Amazon ElastiCache
In-memory data store and cache service.
*   Improves application performance by retrieving data from fast, managed, in-memory caches, instead of relying entirely on slower disk-based databases.
*   Supports Redis and Memcached.

## Amazon Redshift
Fully managed, petabyte-scale data warehouse service in the cloud. Optimized for OLAP (Online Analytical Processing) workloads.


---

## Terraform Examples

### RDS PostgreSQL

```hcl
resource "aws_db_instance" "postgres" {
  identifier        = "app-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  storage_type      = "gp3"

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password # use a secret, never hardcode

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az            = true  # synchronous standby in another AZ
  storage_encrypted   = true
  kms_key_id          = aws_kms_key.app.arn

  backup_retention_period = 7     # keep 7 days of automated backups
  deletion_protection     = true  # prevent accidental destroy
  skip_final_snapshot     = false
  final_snapshot_identifier = "app-db-final"

  auto_minor_version_upgrade  = true  # auto-patch minor versions
  performance_insights_enabled = true  # query-level performance visibility
}
```

### DynamoDB Table

```hcl
resource "aws_dynamodb_table" "sessions" {
  name         = "user-sessions"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "session_id"

  attribute {
    name = "session_id"
    type = "S"
  }

  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  # Protect against accidental deletes/overwrites
  deletion_protection_enabled = true

  # Encrypt with a customer-managed KMS key
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.app.arn
  }

  point_in_time_recovery {
    enabled = true # enables PITR for up to 35 days
  }
}
```

### ElastiCache Redis

```hcl
resource "aws_elasticache_replication_group" "cache" {
  replication_group_id = "app-cache"
  description          = "App Redis cache"
  node_type            = "cache.t3.micro"
  num_cache_clusters   = 2 # 1 primary + 1 replica

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  kms_key_id                 = aws_kms_key.app.arn

  automatic_failover_enabled = true # promote replica automatically on failure
  snapshot_retention_limit   = 5   # keep 5 days of daily snapshots
  snapshot_window            = "03:00-04:00"
}
```

---

### ➡️ Next: [Networking Services](./04-networking.md)
