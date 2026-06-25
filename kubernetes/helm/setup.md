# Setup Guide

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deployment Flow](#deployment-flow)
- [Chart Structure](#chart-structure)
- [Build the Library Chart](#build-the-library-chart)
- [Build a Service Chart](#build-a-service-chart)
- [Render Locally with Helm Template](#render-locally-with-helm-template)
- [Push the Library Chart to the Registry](#push-the-library-chart-to-the-registry)
- [Set Up the ArgoCD ApplicationSet](#set-up-the-argocd-applicationset)
- [Verify Everything](#verify-everything)
- [Notes](#notes)

## Prerequisites

- Kubernetes cluster is running
- Helm installed
- `kubectl` configured and connected to the cluster
- ArgoCD installed and accessible
- An OCI-compatible registry available for storing shared library charts
- A git repo ready to hold each service's chart and values

## Deployment Flow

1. Build the shared library chart(s) for common resources (Service, Secret, etc.)
2. Push the library chart to the OCI registry
3. Build each service's own chart, declaring the library chart as a dependency
4. Add a values file per environment for each service
5. Validate locally with `helm template`
6. Set up an ArgoCD `ApplicationSet` to generate one Application per service and environment
7. Push everything to git and let ArgoCD sync

## Chart Structure

A service chart and its values live together in one repo:

```
<service>/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml      # hand-written, specific to this service
│   ├── service.yaml         # includes the shared library chart
│   └── secret.yaml          # includes the shared library chart
├── values.yaml               # defaults only, not real environment config
└── values/
    ├── dev.yaml
    └── uat.yaml
```

The chart itself never knows which environment it's being deployed to. Everything environment-specific lives in `values/`, not in `templates/`.

## Build the Library Chart

Create the chart and mark it as a library:

```
helm create service-template
```

In `service-template/Chart.yaml`, set:

```yaml
apiVersion: v2
name: service-template
version: 1.0.0
type: library
```

Inside `service-template/templates/`, define the shared resource as a named template rather than a normal output file. Files meant to be included (not rendered directly) conventionally start with an underscore:

```yaml
# service-template/templates/_service.yaml
{{- define "service-template.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.name }}-svc
spec:
  selector:
    app: {{ .Values.name }}
  ports:
    - port: {{ .Values.port | default 80 }}
{{- end -}}
```

## Build a Service Chart

```
helm create api
```

In `api/Chart.yaml`, declare the library chart as a dependency:

```yaml
apiVersion: v2
name: api
version: 1.0.0

dependencies:
  - name: service-template
    version: 1.0.0
    repository: oci://<your-registry>/charts
```

Pull the dependency down:

```
cd api
helm dependency update
```

Reference the shared template from inside the service chart's own templates:

```yaml
# api/templates/service.yaml
{{ include "service-template.service" . }}
```

Write `api/templates/deployment.yaml` by hand — this part is specific to the service and isn't shared.

## Render Locally with Helm Template

Before pushing anything, confirm the chart renders correctly for a given environment:

```
helm template . -f values/dev.yaml
```

This only prints the resulting YAML to the terminal. It doesn't touch the cluster, so it's safe to run as often as needed while iterating.

To check just one resource:

```
helm template . -f values/dev.yaml --show-only templates/deployment.yaml
```

To combine the chart's own defaults with an environment override (later file wins on any overlapping key):

```
helm template . -f values.yaml -f values/dev.yaml
```

## Push the Library Chart to the Registry

Log in, package, and push:

```
helm registry login <your-registry> -u <username> -p <password>
helm package ./service-template
helm push service-template-1.0.0.tgz oci://<your-registry>/charts
```

Only the library chart needs to be pushed this way. Service charts like `api` stay in git and get pulled by ArgoCD directly from there.

## Set Up the ArgoCD ApplicationSet

Create one `ApplicationSet` per project, generating an Application for each service and environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: <project-name>
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - service: api
            repository: <api-repo-url>
            env: dev
            namespace: <project>-dev
          - service: api
            repository: <api-repo-url>
            env: uat
            namespace: <project>-uat

  template:
    metadata:
      name: "{{service}}-{{env}}"
    spec:
      project: default
      source:
        repoURL: "{{repository}}"
        targetRevision: master
        path: "."
        helm:
          valueFiles:
            - "values/{{env}}.yaml"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Apply it:

```
kubectl apply -f <project-name>-applicationset.yaml
```

## Verify Everything

Check the ApplicationSet generated the expected Applications:

```
kubectl get applications -n argocd | grep <project-name>
```

Check a specific Application's sync and health status:

```
kubectl get application <service>-<env> -n argocd -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}'
```

Check the repo-server actually pulled the library chart dependency without errors:

```
kubectl logs -n argocd deploy/argocd-repo-server --tail=50 | grep -i <library-chart-name>
```

Confirm the rendered output in the cluster matches what `helm template` produced locally:

```
kubectl get deployment,service,secret -n <project>-dev
```

## Notes

- Always run `helm template` locally before pushing — it catches template errors before ArgoCD ever sees them
- Re-run `helm dependency update` any time the library chart's version changes
- After creating or editing an ArgoCD repository `Secret` (for registry or git auth), restart `argocd-repo-server` so it picks up the change:

```
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd
```

- If a sync seems to ignore a fix that was just applied, check for `(cached)` in the error message — that means a hard refresh is needed, not another edit
