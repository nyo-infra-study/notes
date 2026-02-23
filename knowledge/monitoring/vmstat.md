# vmstat Command Reference

vmstat (Virtual Memory Statistics) is a command-line tool that provides real-time system performance metrics including memory, CPU, processes, I/O, and disk activity. It's particularly useful for identifying performance bottlenecks and resource constraints.

Source: [Red Hat - Linux commands: exploring virtual memory with vmstat](https://www.redhat.com/en/blog/linux-commands-vmstat)

## Basic Usage

```bash
vmstat [options] [delay [count]]
```

- **delay**: Time interval between updates (in seconds)
- **count**: Number of updates to display

The first report shows averages since the last system reboot. Subsequent reports reflect the specified delay interval.

## Understanding Output

### Standard Output

```bash
vmstat
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 6012852   2120 817048    0    0  2805   289  797  657 21  7 71  1  0
```

### Column Definitions

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

## Common Options

### Active/Inactive Memory

```bash
vmstat -a
```

Shows active and inactive memory instead of buffer/cache breakdown.

### Memory Statistics

```bash
vmstat -s
```

Displays detailed memory statistics and event counters including total memory, swap usage, page faults, and CPU ticks.

### Disk Statistics

```bash
vmstat -d
```

Shows read/write statistics for all disks including total operations, merged operations, sectors, and milliseconds spent.

### Continuous Monitoring with Timestamps

```bash
vmstat -t 5 10
```

Displays 10 updates at 5-second intervals with timestamps for each reading.

### Fork Statistics

```bash
vmstat -f
```

Shows the number of process forks since boot. A fork occurs when a process spawns another process while remaining active.

## Practical Examples

### Monitor system every 2 seconds

```bash
vmstat 2
```

### Check memory pressure

```bash
vmstat -s | grep -E 'memory|swap'
```

### Watch for I/O bottlenecks

```bash
vmstat 1 10
```

Look for high `wa` (I/O wait) values and `bi`/`bo` (block I/O) activity.

### Identify CPU bottlenecks

```bash
vmstat 1
```

Watch the `us` and `sy` columns. High values indicate CPU saturation.

## Interpreting Results

### Memory Issues
- High `swpd` with active `si`/`so` indicates memory pressure
- Low `free` memory isn't necessarily bad if `cache` is high (Linux uses free memory for caching)

### CPU Issues
- High `us` (user time) suggests application CPU usage
- High `sy` (system time) suggests kernel/system call overhead
- High `wa` (I/O wait) indicates disk bottleneck

### I/O Issues
- High `bi`/`bo` with high `wa` suggests disk performance problems
- Many processes in `b` (uninterruptible sleep) often indicates I/O wait

### Context Switching
- Very high `cs` (context switches) can indicate too many processes competing for CPU

## Use Cases for Monitoring

- Diagnosing memory pressure and swap activity
- Identifying CPU bottlenecks (user vs system time)
- Detecting I/O performance issues
- Monitoring system load patterns over time
- Troubleshooting performance degradation
- Capacity planning and resource allocation

---

Content was rephrased for compliance with licensing restrictions.
