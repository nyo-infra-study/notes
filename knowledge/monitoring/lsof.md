# lsof Command Reference

lsof (List Open Files) is a diagnostic tool for identifying which files are being accessed by which processes. In Unix/Linux systems, everything is treated as a file—including pipes, sockets, directories, and devices—making lsof particularly powerful for system monitoring and troubleshooting.

Source: [Neverending Security - lsof Commands Cheatsheet](https://neverendingsecurity.wordpress.com/2015/04/13/lsof-commands-cheatsheet)

## Understanding lsof Output

### Common Columns

- **COMMAND**: Process name
- **PID**: Process ID
- **USER**: Owner of the process
- **FD**: File descriptor
- **TYPE**: File type
- **DEVICE**: Device numbers
- **SIZE/OFF**: File size or offset
- **NODE**: Inode number
- **NAME**: File name or description

### File Descriptor (FD) Values

- `cwd` - current working directory
- `rtd` - root directory
- `txt` - program text/code
- `mem` - memory-mapped file
- `[number]r` - read access
- `[number]w` - write access
- `[number]u` - read and write access

### File Types

- `DIR` - Directory
- `REG` - Regular file
- `CHR` - Character special file
- `FIFO` - First In First Out pipe

## Common Commands

### Basic Listing

```bash
# List all open files
lsof

# List files opened by specific user
lsof -u username

# Exclude specific user
lsof -u^root
```

### Network Connections

```bash
# Show all network connections
lsof -i

# Show IPv4 connections only
lsof -i 4

# Show IPv6 connections only
lsof -i 6

# Show specific port
lsof -i TCP:22

# Show port range
lsof -i TCP:1-1024
```

### Process Information

```bash
# Show files opened by specific PID
lsof -p 1234

# Show what a user is accessing
lsof -i -u username

# Get PIDs only (useful for scripting)
lsof -t -u username
```

### Practical Examples

```bash
# Find which process is using a specific port
lsof -i :8080

# Kill all processes for a user
kill -9 $(lsof -t -u username)

# Check established connections
lsof -i | grep ESTABLISHED

# Monitor specific file access
lsof /path/to/file
```

## Use Cases for Monitoring

- Identifying port conflicts
- Debugging network connectivity issues
- Finding processes holding deleted files
- Monitoring user activity
- Troubleshooting "device busy" errors
- Security auditing of open connections

---

Content was rephrased for compliance with licensing restrictions.
