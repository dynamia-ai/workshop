# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

HAMi GPU virtualization workshop — a series of hands-on labs teaching HAMi commercial edition features. Target audience: trainees learning GPU sharing, scheduling policies, memory management, and monitoring on Kubernetes clusters.

## Repository Structure

- `lab0-install-hami.md` through `lab12-configuration.md` — Sequential lab guides (13 labs)
- `sources/` — Kubernetes YAML manifests referenced by labs (Pod, Deployment, ResourceQuota, etc.)
- `screenshot/` — PNG images used in lab documentation
- `exam.md` — Final exam with 11 practical tasks
- `FAQ.md` — Common troubleshooting Q&A
- `support.md` — Commercial support contact info

## Editing Conventions

- All content is in Chinese (Simplified)
- Labs reference YAML files from `sources/` by relative path
- YAML manifests use HAMi-specific annotations for GPU resource allocation:
  - `hami.io/node-nvidia-register` — device info format: `{UUID},{split count},{mem MB},{core %},{type},{numa},{healthy},{index}`
  - `nvidia.com/gpumem`, `nvidia.com/gpumem-percentage`, `nvidia.com/gpucores` — resource requests
  - `nvidia.com/use-gpu-uuid`, `nvidia.com/nouse-gpu-uuid` — UUID-based scheduling
  - `nvidia.com/use-gpu-type`, `nvidia.com/nouse-gpu-type` — GPU type filtering
  - `nvidia.com/priority` — scheduling priority (0=high, 1=low)
  - `hami.io/node-scheduler-policy` — node-level binpack/spread
  - `hami.io/gpu-scheduler-policy` — card-level binpack/spread
- Screenshots named descriptively (e.g., `memory-scaling01.png`, `gpu-binpack.png`)

## Lab Progression

Lab order matters — each builds on the previous:

1. Install HAMi (offline, helm-based) → Load license → Enable GPU nodes
2. GPU sharing basics → Specified card scheduling → Binpack/Spread policies
3. Priority scheduling → Memory scaling → Resource quotas → Memory analysis/override
4. Monitoring → Configuration (overcommit ratio, split count, default scheduler policy)

## Prerequisites (from labs)

- NVIDIA driver >= 440, nvidia-docker > 2.0, K8s >= 1.18, helm > 3.0
- gpu-operator deployed with `devicePlugin.enabled=false` before HAMi install
- HAMi commercial offline package (images + helm chart) required
