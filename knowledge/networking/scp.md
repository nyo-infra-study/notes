# scp Command Reference

scp (Secure Copy Protocol) is a command-line tool for securely transferring files between hosts over SSH. While rarely needed in cloud-native environments with S3, it's useful for direct server-to-server transfers or legacy system migrations.

Source: [TecMint - SCP Commands Examples](https://www.tecmint.com/scp-commands-examples)

## When to Use SCP vs S3

### Use S3 (Preferred for our infrastructure)

For CDN and static asset management:
- `aws s3 cp` - Upload/download files to S3
- `aws s3 sync` - Synchronize directories with S3
- `aws s3 mb` - Create new buckets
- S3 provides versioning, CDN integration, and scalability

### Use SCP (Rare cases)

- Direct server-to-server file transfers
- Legacy system migrations without S3 access
- Quick one-off file transfers between known hosts
- Backup transfers to non-cloud storage

## Basic Syntax

```bash
scp [options] source_file username@destination_host:destination_folder
```

## Common Operations

### Copy File to Remote Server

```bash
# Basic copy
scp file.pdf user@192.168.1.10:/home/user/

# With verbose output
scp -v file.pdf user@192.168.1.10:/home/user/
```

### Copy File from Remote Server

```bash
scp user@192.168.1.10:/home/user/file.pdf /local/path/
```

### Copy Between Two Remote Hosts

```bash
scp user1@host1:/path/file.pdf user2@host2:/path/
```

### Copy Directory Recursively

```bash
scp -r documents/ user@192.168.1.10:/home/user/
```

## Useful Options

### Preserve Timestamps

```bash
scp -p file.pdf user@host:/path/
```

Maintains original modification and access times.

### Enable Compression

```bash
scp -C large-file.log user@host:/path/
```

Compresses data during transfer. Most effective for text files; minimal benefit for already-compressed files (.zip, .jpg, .mp4).

### Limit Bandwidth

```bash
scp -l 400 file.pdf user@host:/path/
```

Limits bandwidth to 50 KB/s (400 Kbps). Value is in kilobits/sec, so multiply KB/s by 8.

### Use Different Port

```bash
scp -P 2249 file.pdf user@host:/path/
```

Note: Capital `-P` for port (lowercase `-p` is for preserving timestamps).

### Quiet Mode

```bash
scp -q file.pdf user@host:/path/
```

Suppresses progress meter and diagnostic messages.

## SCP vs SFTP

Both use SSH for secure file transfer, but serve different purposes:

### SCP (Secure Copy Protocol)

- **Purpose**: Quick, one-off file transfers
- **Interface**: Command-line only
- **Use case**: Scripting and automation
- **Speed**: Generally faster for simple transfers
- **Features**: Basic copy operations only
- **Resume**: Cannot resume interrupted transfers

```bash
# Example: Copy single file
scp backup.tar.gz user@host:/backups/
```

### SFTP (SSH File Transfer Protocol)

- **Purpose**: Interactive file management
- **Interface**: Interactive shell or GUI clients
- **Use case**: Browsing, managing remote files
- **Speed**: Slightly slower due to protocol overhead
- **Features**: Full file operations (ls, rm, mkdir, chmod, etc.)
- **Resume**: Can resume interrupted transfers

```bash
# Example: Interactive session
sftp user@host
sftp> ls
sftp> cd /backups
sftp> put backup.tar.gz
sftp> get remote-file.log
sftp> exit
```

### When to Choose Which

Use SCP when:
- Scripting automated transfers
- Simple one-time copy operations
- Speed is critical for large files

Use SFTP when:
- Need to browse remote directories
- Performing multiple file operations
- Need to resume interrupted transfers
- Using GUI file transfer clients

## S3 Equivalent Commands

For our cloud infrastructure, prefer AWS CLI over SCP:

```bash
# Upload file to S3
aws s3 cp file.pdf s3://bucket-name/path/

# Download from S3
aws s3 cp s3://bucket-name/path/file.pdf ./

# Sync directory to S3 (like rsync)
aws s3 sync ./local-dir s3://bucket-name/path/

# Copy between S3 buckets
aws s3 cp s3://source-bucket/file s3://dest-bucket/file

# Recursive copy
aws s3 cp ./directory s3://bucket-name/path/ --recursive
```

## Practical Examples

### Backup Database Dump

```bash
# Create and transfer backup
mysqldump -u root database > backup.sql
scp -C backup.sql backup-server:/backups/$(date +%Y%m%d).sql
```

### Transfer Logs for Analysis

```bash
# Copy logs with compression
scp -C /var/log/app.log analyst@workstation:/analysis/
```

### Migrate Files Between Servers

```bash
# Copy entire directory structure
scp -r /var/www/html user@new-server:/var/www/
```

## Security Considerations

- Always use key-based authentication instead of passwords
- Verify host fingerprints on first connection
- Use non-standard SSH ports when possible
- Limit bandwidth for non-critical transfers to avoid network saturation
- Consider using rsync over SSH for large transfers (supports resume)

## Troubleshooting

### Permission Denied

Ensure SSH key is properly configured and destination directory is writable.

### Connection Timeout

Check firewall rules and verify SSH service is running on destination.

### Slow Transfer Speed

Use `-C` for compression on text files, or check network bandwidth with `-l` parameter.

---

Content was rephrased for compliance with licensing restrictions.
