# Observability

[← Back to Knowledge Base](../README.md)

## Overview

See what's happening in your systems and debug issues effectively. When things break in production — and they will — you need centralized logs, dashboards, and the right diagnostic tools to find the root cause fast.

## Learning Path

1. **[PLG Stack](./01-plg-stack.md)** - Centralized logging and visualization with Promtail, Loki, and Grafana
2. **[Diagnostic Tools](./02-diagnostic-tools.md)** - System-level debugging with lsof, vmstat, and Kubernetes-native tools

## Prerequisites

Before diving into this section, you should understand:
- [Container Orchestration](../02-container-orchestration/README.md) — you need running applications to observe
- [Deployment Automation](../03-deployment-automation/README.md) — understanding how apps are deployed helps interpret what you see

## What You'll Learn

- How to ship logs from every pod to a central store and query them with Loki
- How to build Grafana dashboards that surface the metrics that matter
- How to diagnose resource contention, open file handles, and network issues at the OS level

## Estimated Time

⏱️ 2-4 hours total

## Difficulty

🎯 Intermediate

## Next Steps

After completing this section, continue to:
- [Cloud Infrastructure](../05-cloud-infrastructure/README.md)
