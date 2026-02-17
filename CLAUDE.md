# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kubernetes homelab infrastructure repository using GitOps with ArgoCD. Runs on Talos OS (v1.10.3) with Kubernetes v1.33.1, Cilium CNI, and Longhorn distributed storage. The cluster is named "homelab" with 3 control plane nodes (scheduling allowed on control planes) and worker nodes.

## Architecture

**GitOps Flow:** Git commit → ArgoCD detects change → syncs to cluster automatically.

**Bootstrap chain:** Talos OS boots → applies `infra/cluster-template.yaml` with patches (Cilium CNI, ArgoCD, monitoring) → ArgoCD starts → `apps/argocd/bootstrap-app-set.yaml` ApplicationSet discovers all directories under `apps/*` and creates an ArgoCD Application for each one → each app auto-syncs with server-side apply.

**App structure:** Each subdirectory under `apps/` becomes its own ArgoCD Application deployed to a namespace matching the directory name. Apps are either:
- **Helm wrapper charts** (most common): `Chart.yaml` declaring upstream chart dependency + `values.yaml` with overrides
- **Kustomize deployments**: `kustomization.yaml` with remote bases or raw manifests

**Infrastructure layer** (`infra/`): Talos cluster template and patches applied at OS level before Kubernetes starts. The `patches/cilium.yaml` is large (~100KB) containing full Cilium manifests including CRDs, RBAC, ConfigMaps, and DaemonSets.

## Key Conventions

- Domain pattern: `*.2411.house` with Cloudflare tunnel + cert-manager (LetsEncrypt prod)
- Timezone: `America/Los_Angeles` across all apps
- Ingress annotations: external-dns for Cloudflare, hajimari for app launcher, cert-manager for TLS
- Node targeting: specialized workloads use `nodeSelector` (e.g., `nas-worker` for Home Assistant)
- Persistence: Longhorn PVCs, typically 4Gi for config, larger for data (150Gi for Prometheus)
- ArgoCD sync: automated with ServerSideApply, CreateNamespace, infinite retry backoff

## Common Operations

There are no build scripts, Makefiles, or CI pipelines in this repo. All changes deploy via Git push → ArgoCD sync.

**Adding a new app:** Create a directory under `apps/<app-name>/` with either a Helm chart (`Chart.yaml` + `values.yaml`) or Kustomize layout (`kustomization.yaml`). The bootstrap ApplicationSet auto-discovers it.

**Updating an app version:** Change the chart version in `Chart.yaml` or image tag in `values.yaml`. Renovate handles this automatically for most dependencies.

**Cluster infrastructure changes:** Edit `infra/cluster-template.yaml` or files under `infra/patches/`. These require Talos apply (not ArgoCD).

## Renovate Dependency Management

`renovate.json` tracks: Helm chart versions, Helm values image tags, Kustomize remote git refs, inline container images, Docker mods, and GitHub release URLs. Patch-level Helm updates auto-merge. Docker images using `latest` tag are pinned to digest.
