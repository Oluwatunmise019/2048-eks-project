# Incident Report — 2048 EKS Stress Test

**Date:** 2026-05-14
**Author:** Asalu Oluwatunmise
**Environment:** Amazon EKS — 2048-prod-cluster (us-east-1)
**Cluster:** 2 x t3.medium nodes, 2 replicas, HPA min=2 max=6

---

## Overview

This document records two incidents encountered during the 
deployment and stress testing of the 2048 game on Amazon EKS. 
Both were diagnosed, root-caused, and resolved. This is not a 
failure report — it is a record of real engineering decisions 
made under real conditions.

---

## Incident 1 — Container Image Incompatibility

**Phase:** Initial deployment  
**Severity:** High — blocked all pods from starting  
**Status:** Resolved

### What Happened

Two Docker images failed to pull onto the EKS nodes:

- `blackicebird/2048` — original tutorial image
- `alexwhen/docker-2048` — first replacement attempt

Both returned the same error:
failed to pull and unpack image: not implemented:
media type "application/vnd.docker.distribution.manifest.v1+prettyjws"
is no longer supported since containerd v2.1

### Root Cause

EKS 1.30 uses containerd v2.1 as its container runtime. 
Both images were built years ago using the legacy Docker 
image manifest format (v1). containerd v2.1 dropped support 
for this format entirely. The images themselves were not 
broken — they were simply too old for the runtime.

### Diagnosis Method

```bash
kubectl describe pod <pod-name>
# Events section revealed the exact containerd error
```

### Fix

Switched to `public.ecr.aws/l6m2t8p7/docker-2048:latest` — 
a modern image hosted on AWS Elastic Container Registry, 
built with the current OCI manifest format. ECR also pulls 
faster on EKS nodes since both are within the AWS network.

### Lesson

Never assume a public Docker Hub image is maintained or 
compatible with current runtimes. Always verify the image's 
last push date and check containerd compatibility when using 
EKS 1.28+.

---

## Incident 2 — CrashLoopBackOff from Over-Hardening

**Phase:** Security context configuration  
**Severity:** High — all pods crashed on startup  
**Status:** Resolved

### What Happened

After switching to the correct image, all pods entered 
`CrashLoopBackOff` immediately on startup. The deployment 
showed 10+ restarts within minutes.

### Root Cause

The deployment manifest included an aggressive security 
hardening rule:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

This dropped every Linux kernel capability from the container 
process. The nginx web server inside the container requires 
write access to `/var/lib/nginx/tmp/` to create working 
directories on startup. With all capabilities dropped, nginx 
could not create these directories and crashed immediately.

### Diagnosis Method

```bash
kubectl logs <pod-name>
# Revealed the exact failure:
# nginx: [alert] could not open error log file: 
# open() "/var/lib/nginx/logs/error.log" failed (13: Permission denied)
# mkdir() "/var/lib/nginx/tmp/client_body" failed (13: Permission denied)
```

### Fix

Removed `capabilities: drop: ALL` from the container security 
context. Retained the safer, less destructive restriction:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

This prevents the container process from ever gaining more 
permissions than it started with — closing the privilege 
escalation attack vector — without stripping permissions the 
container legitimately needs to function.

### Lesson

Security hardening is not one-size-fits-all. Blanket capability 
dropping must be validated against each container's actual 
runtime requirements. The correct workflow is: apply hardening, 
test, read logs when it breaks, make targeted fixes. A security 
rule that crashes your app protects nothing.

---

## Incident 3 — Prometheus Installation Timeout

**Phase:** Observability setup  
**Severity:** Medium — blocked monitoring stack  
**Status:** Resolved with adjusted approach

### What Happened

Full `kube-prometheus-stack` Helm installation failed twice 
with timeout errors:
failed to install CRD crds/crd-prometheuses.yaml:
server-side apply failed: the server was unable to return
a response in the time allotted

### Root Cause

The full stack installs 40+ Kubernetes resources simultaneously 
including large CRDs. On a fresh 2-node t3.medium cluster 
already running the game pods, the Kubernetes API server was 
too slow to process all resource creation requests within 
Helm's default timeout window.

### Fix Applied

Split the installation into two steps:

1. Installed the Prometheus operator and CRDs separately first 
   via `kubectl apply`, giving the API server time to process them
2. Installed a lighter Prometheus-only chart (`prometheus-community/prometheus`) 
   with alertmanager and pushgateway disabled
3. Installed Grafana separately via its own Helm chart
4. Disabled PersistentVolumeClaims on both — avoiding the need 
   for EBS CSI driver configuration

### Tradeoff Acknowledged

Disabling persistence means metrics are lost if the Prometheus 
pod restarts. For a production environment, the correct fix 
would be installing the AWS EBS CSI driver add-on on the EKS 
cluster and configuring a proper StorageClass. This was 
descoped for the purposes of this project.

---

## Stress Test Results

**Tool:** `hey` HTTP load generator  
**Target:** AWS ELB endpoint (HTTP port 80)  
**Concurrent users:** 100  
**Duration:** 3 minutes per run

### Run 1
| Metric | Value |
|---|---|
| Requests/sec | 60.97 |
| Total requests | 11,536 |
| Successful (200) | 11,533 (99.97%) |
| Average response time | 1.57s |
| 50th percentile | 1.00s |
| 90th percentile | 3.47s |
| 99th percentile | 10.83s |
| Failures | 3 timeouts |

### Run 2
| Metric | Value |
|---|---|
| Requests/sec | 44.46 |
| Total requests | 8,149 |
| Successful (200) | 8,148 (99.99%) |
| Average response time | 2.22s |
| 50th percentile | 1.26s |
| 90th percentile | 5.18s |
| 99th percentile | 11.44s |
| Failures | 1 timeout |

### Key Finding — Bottleneck Was Not the Pods

**The HPA did not scale up during either test run.**

HPA CPU readings throughout:
cpu: 1%/50% → 2%/50% → 5%/50% → 4%/50% → 3%/50%

Pod CPU never approached the 50% scale threshold. The pods 
were healthy and barely working hard. Yet response times 
degraded significantly at the tail end — 99th percentile 
exceeded 10 seconds on both runs.

**Conclusion:** The bottleneck was the AWS Classic Load 
Balancer and DNS resolution latency under 100 concurrent 
connections from a single origin, not the application pods 
themselves.

### What I Would Do in Production

1. Replace Classic ELB with AWS Application Load Balancer 
   (ALB) — better connection handling and HTTP/2 support
2. Add CloudFront CDN in front for static asset caching — 
   the 2048 game assets never change, they should be cached 
   at the edge
3. Lower HPA CPU threshold from 50% to 30% to react earlier 
   to sustained load
4. Add pod disruption budgets to guarantee minimum pod 
   availability during node maintenance
5. Configure EBS CSI driver for persistent Prometheus storage

---

## What Worked Well

- Rolling updates — zero downtime during all config changes. 
  New pods were healthy before old ones were terminated.
- Liveness probes — automatically detected and restarted 
  crashing containers without manual intervention
- HPA monitoring — correctly identified that CPU was not the 
  bottleneck and did not scale unnecessarily
- 99.97% request success rate under sustained 100-user load
- Grafana dashboard showed real-time network I/O spikes 
  during stress test
- eksctl cluster config — entire cluster reproducible from 
  a single YAML file
