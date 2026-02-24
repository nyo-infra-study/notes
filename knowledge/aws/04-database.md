# Database Services

AWS offers purpose-built databases for various application needs.

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

### ➡️ Next: [Networking Services](./05-networking.md)
