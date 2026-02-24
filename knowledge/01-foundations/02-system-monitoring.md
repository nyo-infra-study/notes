# System Monitoring

[Knowledge Base](../README.md) > [Foundations](./README.md) > System Monitoring

System monitoring tools are essential for diagnosing performance issues, identifying resource constraints, and understanding what's happening on a Linux system. This guide covers `lsof` for tracking open files and network connections, and `vmstat` for real-time system performance metrics.

## Prerequisites

- [Text Manipulation](./01-text-manipulation.md) — filtering and parsing command output with grep, awk, cut

## Next Steps

- [Networking Basics](./03-networking-basics.md) — file transfers and SSH-based connectivity
- [Kubernetes](../02-container-orchestration/01-kubernetes.md) — apply these skills to debug pods and nodes

## lsof - List Open Files

lsof (List Open Files) is a diagnostic tool for identifying which files are being accessed by which processes. In Unix/Linux systems, everything is treated as a file—including pipes, sockets, directories, and devices—making lsof particularly powerful for system monitoring and troubleshooting.

Source: [Neverending Security - lsof Commands Cheatsheet](https://neverendingsecurity.wordpress.com/2015/04/13/lsof-commands-cheatsheet)

### Understanding lsof Output

#### Common Columns

- **COMMAND**: Process name
- **PID**: Process ID
- **USER**: Owner of the process
- **FD**: File descriptor
- **TYPE**: File type
- **DEVICE**: Device numbers
- **SIZE/OFF**: File size or offset
- **NODE**: Inode number
- **NAME**: File name or description

#### File Descriptor (FD) Values

- `cwd` - current working directory
- `rtd` - root directory
- `txt` - program text/code
- `mem` - memory-mapped file
- `[number]r` - read access
- `[number]w` - write access
- `[number]u` - read and write access

#### File Types

- `DIR` - Directory
- `REG` - Regular file
- `CHR` - Character special file
- `FIFO` - First In First Out pipe

### Common Commands

#### Basic Listing

```bash
# List all open files
lsof

# List files opened by specific user
lsof -u username

# Exclude specific user
lsof -u^root
```

#### Network Connections

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

#### Process Information

```bash
# Show files opened by specific PID
lsof -p 1234

# Show what a user is accessing
lsof -i -u username

# Get PIDs only (useful for scripting)
lsof -t -u username
```

#### Practical Examples

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

### Use Cases for lsof

- Identifying port conflicts
- Debugging network connectivity issues
- Finding processes holding deleted files
- Monitoring user activity
- Troubleshooting "device busy" errors
- Security auditing of open connections

---

## vmstat - Virtual Memory Statistics

vmstat (Virtual Memory Statistics) is a command-line tool that provides real-time system performance metrics including memory, CPU, processes, I/O, and disk activity. It's particularly useful for identifying performance bottlenecks and resource constraints.

Source: [Red Hat - Linux commands: exploring virtual memory with vmstat](https://www.redhat.com/en/blog/linux-commands-vmstat)

### Basic Usage

```bash
vmstat [options] [delay [count]]
```

- **delay**: Time interval between updates (in seconds)
- **count**: Number of updates to display

The first report shows averages since the last system reboot. Subsequent reports reflect the specified delay interval.

### Understanding Output

#### Standard Output

```bash
vmstat
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 6012852   2120 817048    0    0  2805   289  797  657 21  7 71  1  0
```

#### Column Definitions

**Procs (Processes)**
- `r` - Runnable processes (running or waiting for CPU)
- `b` - Processes in uninterruptible sleep

**Memory (in KB)**
- `swpd` - Virtual memory used
- `free` - Idle memory
- `buff` - Memory used as buffers
- `cache` - Memory used as cache

**Swap (in KB/s)**
- `si` - Memory swapped in from disk
- `so` - Memory swapped out to disk

**IO (blocks/s)**
- `bi` - Blocks received from block device
- `bo` - Blocks sent to block device

**System**
- `in` - Interrupts per second
- `cs` - Context switches per second

**CPU (% of total CPU time)**
- `us` - User time (non-kernel code)
- `sy` - System time (kernel code)
- `id` - Idle time
- `wa` - Time waiting for I/O
- `st` - Time stolen from virtual machine

### Common Options

#### Active/Inactive Memory

```bash
vmstat -a
```

Shows active and inactive memory instead of buffer/cache breakdown.

#### Memory Statistics

```bash
vmstat -s
```

Displays detailed memory statistics and event counters including total memory, swap usage, page faults, and CPU ticks.

#### Disk Statistics

```bash
vmstat -d
```

Shows read/write statistics for all disks including total operations, merged operations, sectors, and milliseconds spent.

#### Continuous Monitoring with Timestamps

```bash
vmstat -t 5 10
```

Displays 10 updates at 5-second intervals with timestamps for each reading.

#### Fork Statistics

```bash
vmstat -f
```

Shows the number of process forks since boot. A fork occurs when a process spawns another process while remaining active.

### Practical Examples

```bash
# Monitor system every 2 seconds
vmstat 2

# Check memory pressure
vmstat -s | grep -E 'memory|swap'

# Watch for I/O bottlenecks (look for high 'wa' values)
vmstat 1 10

# Identify CPU bottlenecks (watch 'us' and 'sy' columns)
vmstat 1
```

### Interpreting Results

#### Memory Issues
- High `swpd` with active `si`/`so` indicates memory pressure
- Low `free` memory isn't necessarily bad if `cache` is high (Linux uses free memory for caching)

#### CPU Issues
- High `us` (user time) suggests application CPU usage
- High `sy` (system time) suggests kernel/system call overhead
- High `wa` (I/O wait) indicates disk bottleneck

#### I/O Issues
- High `bi`/`bo` with high `wa` suggests disk performance problems
- Many processes in `b` (uninterruptible sleep) often indicates I/O wait

#### Context Switching
- Very high `cs` (context switches) can indicate too many processes competing for CPU

### Use Cases for vmstat

- Diagnosing memory pressure and swap activity
- Identifying CPU bottlenecks (user vs system time)
- Detecting I/O performance issues
- Monitoring system load patterns over time
- Troubleshooting performance degradation
- Capacity planning and resource allocation

---

Content was rephrased for compliance with licensing restrictions.
