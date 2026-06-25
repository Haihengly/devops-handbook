# Helm Management

This project demonstrates how to structure Helm charts and environment-specific values so the same chart can be rendered and deployed consistently across multiple environments (dev, uat) using ArgoCD.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [What It Includes](#what-it-includes)
- [Chart Design](#chart-design)
- [Environment Management](#environment-management)
- [Deployment Flow](#deployment-flow)
- [Documentation](#documentation)
- [Notes](#notes)

## Overview

This project sets up a repeatable pattern for deploying services to Kubernetes via Helm, where chart structure and environment-specific configuration are kept separate. ArgoCD watches the chart and values, renders them together with `helm template`, and applies the result to the cluster — automatically, per environment.

## Architecture

```
Library Charts (service-template, secret-template) → pushed to OCI Registry

Git Repo (chart + values) → ArgoCD ApplicationSet → Application (per service+env)
        → helm dependency build (pulls library charts from OCI Registry)
        → helm template (chart + values merged)
        → kubectl apply
```

## What It Includes

- Helm charts structured for reuse across environments
- Shared library charts (`service-template`, `secret-template`) for common resources
- Per-environment values files (`dev.yaml`, `uat.yaml`)
- ArgoCD `ApplicationSet` generating one Application per service+environment combination
- OCI registry as the chart distribution method

## Chart Design

Charts are kept environment-agnostic — they define structure only (Deployments, Services, Secrets), never environment-specific values. Shared, reusable pieces are split into library charts (`type: library`) and consumed as dependencies by each service's own chart.

- Service charts is deployable, contain their own `deployment.yaml`
- Library charts (`service-template`, `secret-template`) → not deployable on their own, included by service charts

## Environment Management

Each environment (dev, uat) gets its own values file, layered on top of the chart at render/deploy time.

- Values live alongside the chart in the same repo, under a any custom folder
- `helm template . -f path to env file` renders the chart for a specific environment locally, before anything touches the cluster
- ArgoCD's `ApplicationSet` automates this per service, per environment, using a generator (`list` or `matrix`) to avoid hand-writing one Application file per combination

## Deployment Flow

1. Write/update the chart's templates (structure only)
2. Write/update the environment's values file (`values/dev.yaml`, `values/uat.yaml`)
3. Validate locally with `helm template . -f values/<env>.yaml`
4. Push to the chart's repo
5. ArgoCD's `ApplicationSet` detects the change and syncs the corresponding Application
6. ArgoCD renders and applies the result to the target namespace

## Notes

- Charts and values are versioned independently in intent — a chart version bump means structure changed, not a config tweak
- Library charts cannot be deployed standalone (`helm template`/`helm install` on one directly fails with `library charts are not installable`) — this is expected
- One `ApplicationSet` is scoped per project, not one global file across all services

## Documentation

- [Setup Guide](./setup.md)
- [Troubleshooting](./troubleshooting.md)
