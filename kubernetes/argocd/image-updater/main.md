# Argo CD Image Updater — GitOps Automated Deployment

This project demonstrates an automated image tag update pipeline using Argo CD, Argo CD Image Updater, and GitLab, replacing manual Jenkins-based manifest updates with a fully GitOps-driven flow.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [What It Includes](#what-it-includes)
- [Update Strategy Design](#update-strategy-design)
- [Write-Back Flow](#write-back-flow)
- [Monitoring](#monitoring)
- [Deployment Flow](#deployment-flow)
- [Documentation](#documentation)
- [Notes](#notes)

## Overview
This project automates image tag updates for Kubernetes workloads deployed via Argo CD. Instead of a CI pipeline manually checking out the manifest repo, editing the tag, and pushing a commit, Argo CD Image Updater watches the container registry directly and commits tag changes to Git on its own. Argo CD then syncs the change automatically, completing a fully GitOps-driven deployment loop.

## Architecture
```
Container Registry → Image Updater Controller → Git Repository (GitLab) → Argo CD Sync → Kubernetes Deployment
```

## What It Includes
- Argo CD Image Updater controller (CRD-based, `v1.2.2`)
- ApplicationSet-driven multi-environment deployment (dev / uat)
- Private registry authentication via `pull-secret`
- Git write-back authentication via dedicated GitLab credentials
- `ImageUpdater` CR with global `namePattern` coverage
- Custom Git commit message templating for audit-friendly history
- Prometheus metrics integration for observability

## Update Strategy Design
Image tags are plain incrementing build numbers (not semver), so the `newest-build` strategy is used — it selects the image with the most recent registry push timestamp rather than relying on tag-name sorting.

| Strategy | Use Case |
|---|---|
| `newest-build` | Plain/incrementing tags, most recent push wins (used here) |
| `semver` | Tags following `X.Y.Z` format |
| `digest` | Pin to a specific immutable image digest |

**Known limitation:** if the currently deployed tag is deleted from the registry, the strategy automatically falls back to the next newest available tag on the following poll cycle, with no approval step. Currently-deployed tags should never be deleted — see [Troubleshooting](./troubleshooting.md).

## Write-Back Flow
Image Updater writes tag updates directly into each environment's existing Helm values file (`env/dev.yaml`, `env/uat.yaml`) rather than creating a separate override file, keeping the deployed version visible in one place.

```
1. Registry polled every ~2 min
2. New tag detected → matches update strategy
3. Repo cloned using Argo CD's registered git credential
4. env/{env}.yaml updated directly (helmvalues write-back target)
5. Commit pushed to master (custom commit message template)
6. Argo CD detects new commit → auto-syncs (selfHeal + automated)
```

## Monitoring
Image Updater exposes Prometheus metrics for observability, since update failures (bad credentials, registry auth, git push rejections) fail silently in the background otherwise.

- `argocd_image_updater_images_errors_total` — failed update attempts
- `argocd_image_updater_images_updated_total` — successful updates
- `argocd_image_updater_applications_watched_total` — apps currently tracked

Metrics require RBAC access granted to Prometheus's ServiceAccount against the controller's `metrics-reader` / `metrics-auth` ClusterRoleBindings — see [Setup Guide](./setup.md).

## Deployment Flow
1. Install Argo CD Image Updater controller
2. Configure private registry and Git credentials
3. Apply `ImageUpdater` CR (global `namePattern: "*"`, `useAnnotations: true`)
4. Add Image Updater annotations to each Application/ApplicationSet template
5. Grant GitLab protected-branch push access to the credential in use
6. Verify end-to-end with a test tag push

## Notes
- Currently scoped to **dev and uat only** — not yet promoted to production
- One shared `ImageUpdater` CR covers all applications; onboarding a new service only requires adding annotations, no CR changes
- Rollbacks require both a Git revert **and** an `ignore-tags` annotation to prevent the controller from re-promoting a reverted tag — see [Troubleshooting](./troubleshooting.md)
- Designed to scale across ~30 microservices without per-service pipeline duplication

## Documentation
- [Setup Guide](./setup.md)
- [Troubleshooting](./troubleshooting.md)

## Repository
GitHub Repository: _(add your repo link here)_
