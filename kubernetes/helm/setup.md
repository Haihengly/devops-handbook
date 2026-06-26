# Setup Guide

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deployment Flow](#deployment-flow)
- [ArgoCD Secret](#create-secret-for-argocd)
- [Build the Library Chart](#build-the-library-chart)
- [Push the Library Chart to the Registry](#push-the-library-chart-to-the-registry)
- [Build a Service Chart](#build-a-service-chart)
- [Render Locally with Helm Template](#render-locally-with-helm-template)
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

# Create Secret for ArgoCD

To let argocd access private registry you need credential.

Create file for secret anywhere

```
apiVersion: v1
kind: Secret
metadata:
  name: oci-registry
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: oci-private-registry
  url: <your-registry>/charts
  enableOCI: "true"
  username: <your-registry-username>
  password: <your-registry-password>
```

> Replace stringData with data for base64.

Apply : 

```
kubectl apply -f <your-file-name>
```

Verify : 

```
kubectl get secret -n argocd
or
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

Restart argocd-repo-server :

```
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd
```

## Build the Library Chart

Structure

```
library/
├── service/
│    ├── Chart.yaml
│    └── templates/
│        └── -service.yaml                
├── secret/
│    ├── Chart.yaml
│    └── templates/
│        └── -secret.yaml 
└── <N-template>/
     ├── Chart.yaml
     └── templates/
         └── -<N>.yaml 
```

In `service-template/Chart.yaml`, set:

```yaml
apiVersion: v2
name: service-template
version: 1.0.0
type: library
```

Inside `service-template/templates/`, define the shared resource as a named template rather than a normal output file. Files meant to be included (not rendered directly)

```yaml
{{- define "service-template.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}

spec:
  type: {{ .Values.service.type | default "ClusterIP" }}

  selector:
    app: {{ .Values.app.name }}

  ports:
    - port: {{ .Values.service.port }}
      protocol: {{ .Values.service.protocol | default "TCP" }}
      targetPort: {{ .Values.service.targetPort }}
      nodePort: {{ .Values.service.nodePort }}
{{- end }}
```

## Push the Library Chart to the Registry

Log in, package, and push:

```
helm registry login <your-registry> -u <username> -p <password>
helm package ./service-template
```
#you will get package name service-template-1.0.0.tgz (build from Chart.yaml and ./templates)
```
helm push service-template-1.0.0.tgz oci://<your-registry>/charts
```

Only the library chart needs to be pushed this way. The charts itself stay in git and get pulled by ArgoCD directly from there.

## Build a Service Chart

> Structure

A service chart and its values live together in one repo:

```
<service>/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml      # hand-written, specific to this service
│   ├── service.yaml         # includes the shared library chart
│   └── secret.yaml          # includes the shared library chart
├── values.yaml               # defaults only, not real environment config
└── CUSTOM-FOLDER/
    ├── dev.yaml
    └── uat.yaml
```

The chart itself never knows which environment it's being deployed to. Everything environment-specific lives in `CUSTOM-FOLDER/`, not in `templates/`.

> Now Create Your Chart 

```
helm create <your-service-name>
```

In `<your-service-name>/Chart.yaml`, declare the library chart as a dependency:

```yaml
apiVersion: v2
name: <your-service-name>
version: 1.0.0

dependencies:
  - name: <your-template-name>
    version: 1.0.0
    repository: oci://<your-registry>/charts
```

Pull the dependency down:

```
cd ./<your-service-name>
helm dependency build
```
`use helm dependency update if chart version change or chart being update`

Reference the shared template from inside the service chart's own templates:

```yaml
# <your-service-name>/templates/service.yaml
{{ include "service-template.service" . }}
```

`can be also include directly inside deployment.yaml`

In `<your-service-name>/templates/deployment.yaml` manual writing, this part is specific to the service and isn't shared.

## Render Locally with Helm Template

Before pushing anything, confirm the chart renders correctly for a given environment:

```
helm template . -f CUSTOM-FOLDER/dev.yaml
```

This only prints the resulting YAML to the terminal. It doesn't touch the cluster, so it's safe to run as often as needed while iterating.

To check just one resource:

```
helm template . -f CUSTOM-FOLDER/dev.yaml --show-only templates/deployment.yaml
```

To combine the chart's own defaults with an environment override (later file wins on any overlapping key):

```
helm template . -f values.yaml -f CUSTOM-FOLDER/dev.yaml
```

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
          - service: <service-01>
            repository: <service-01-repo-url>
            env: dev
            namespace: <your-namespace>
          - service: <service-01>
            repository: <service-01-repo-url>
            env: uat
            namespace: <your-namespace>
          - service: <service-02>
            repository: <service-02-repo-url>
            env: dev
            namespace: <your-namespace>
          - service: <service-02>
            repository: <service-02-repo-url>
            env: uat
            namespace: <your-namespace>

  template:
    metadata:
      name: "{{service}}-{{env}}"
    spec:
      project: default
      source:
        repoURL: "{{repository}}"
        targetRevision: master
        path: "." # path to the chart
        helm:
          valueFiles:
            - "CUSTOM-FOLDER/{{env}}.yaml" # path to to env vlaues 
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
kubectl apply -f <your-file-name>.yaml
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

Confirm the rendered output in the cluster matches what `helm template` produced locally:

```
kubectl get pod,configmap,deployment,svc,secret -n <your-name-space>
```

## Notes

- Always run `helm template` locally before pushing — it catches template errors before ArgoCD ever sees them
- Re-run `helm dependency update` any time the library chart's version changes
- After creating or editing an ArgoCD repository `Secret` (for registry or git auth), restart `argocd-repo-server` so it picks up the change:

```
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd
```