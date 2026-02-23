# Monitoring & Troubleshooting

This directory contains documentation for monitoring and troubleshooting tools used in our infrastructure.

## Overview

Our monitoring strategy follows a layered approach:

1. **Application-level logging** via PLG Stack (Promtail, Loki, Grafana)
2. **System-level diagnostics** via command-line tools like lsof

## When to Use Each Tool

### PLG Stack (Primary)

Use the PLG stack for:
- Viewing application logs across all services
- Historical log analysis and debugging
- Tracing requests across microservices
- Monitoring error rates and patterns
- Persistent log storage and search

The PLG stack should be your first stop for most debugging scenarios. See [plg-stack.md](../plg-stack.md) for setup and usage.

### lsof (Secondary/Diagnostic)

Use lsof when you encounter specific system-level issues that logs don't explain:

- **Port conflicts**: "Address already in use" errors
- **File locks**: "Device or resource busy" errors
- **Network issues**: Identifying which process is using a port
- **Resource leaks**: Finding processes holding deleted files
- **Connection debugging**: Checking established TCP connections

See [lsof.md](./lsof.md) for command reference.

## Typical Troubleshooting Flow

1. **Check PLG Stack first**
   - View logs in Grafana at `http://localhost:9000/grafana`
   - Filter by namespace, pod, or error level
   - Look for error messages and stack traces

2. **If logs don't reveal the issue, check system state**
   - Use `kubectl get pods` to verify pod status
   - Use `kubectl describe pod` for events and resource issues

3. **For specific system-level problems, use lsof**
   - Port conflicts: `lsof -i :PORT`
   - File access issues: `lsof /path/to/file`
   - Network connections: `lsof -i -u username`

## Available Documentation

- [lsof.md](./lsof.md) - Command-line tool for listing open files and network connections

## Related Documentation

- [PLG Stack](../plg-stack.md) - Centralized logging setup
- [Kubernetes](../kubernetes.md) - Container orchestration basics
- [ArgoCD](../argocd.md) - GitOps deployment tool
