# Setup Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [Deployment Flow](#deployment-flow)
- [Install Image Updater](#install-image-updater)
- [Fix DNS](#fix-dns)
- [Apply Secrets](#apply-secrets)
- [Apply ImageUpdater CR](#apply-imageupdater-cr)
- [Configure Application Annotations](#configure-application-annotations)
- [Allow Git Push](#allow-git-push)
- [Configure Commit Message](#configure-commit-message)
- [Verify Everything](#verify-everything)
- [Test Update](#test-update)

## Prerequisites
- Kubernetes cluster is running
- Argo CD is already installed and syncing your application
- `kubectl` configured and connected to cluster
- GitLab bot account/token with repo access ready
- Private registry credentials ready (if applicable)

## Deployment Flow
1. Install Argo CD Image Updater
2. Fix DNS if the Git host isn't resolvable
3. Apply registry and Git secrets
4. Apply the ImageUpdater CR
5. Add annotations to the Application
6. Allow Git push on protected branch
7. Configure commit message template
8. Verify and test

## Install Image Updater
Install directly using the official manifest:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml
```

Or install manually — visit the URL below, copy the content into a local file, then apply it:
```
https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml
```
```
kubectl apply -n argocd -f install.yaml
```

Verify the pod is running:
```
kubectl get pod -n argocd -l control-plane=argocd-image-updater-controller
```

## Fix DNS
Check if the controller can resolve your Git host:
```
kubectl exec -it -n argocd deployment/argocd-image-updater-controller -- getent hosts your-gitlab-host.com
```

If it fails, patch the deployment with a manual DNS mapping:
```yaml
kubectl patch deployment argocd-image-updater-controller -n argocd --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/hostAliases",
    "value": [
      {"ip": "YOUR_GITLAB_INTERNAL_IP", "hostnames": ["your-gitlab-host.com"]}
    ]
  }
]'
```

Or edit manually in install file or CLI
```
kubectl edit deployment argocd-image-updater-controller -n argocd 
```
Find Kind:Deployment add this under `spec.template.spec`, at the same level as `containers:`:
```yaml
      hostAliases:
        - ip: "YOUR_GITLAB_INTERNAL_IP"
          hostnames:
            - "your-gitlab-host.com"
```

## Apply Secrets

Apply private registry pull-secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: "YOUR_SECRET_NAME"
  namespace: argocd
data:
  .dockerconfigjson: "YOUR_SECRET_BASE64"
type: kubernetes.io/dockerconfigjson
```

```
kubectl apply -f YOUR_SECRET_FILE.yml
```

Apply Git credential secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: "YOUR_GIT_SECRET_NAME"
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git 
  url: "YOUR_REPO_URL"
  username: "YOUR_BOT_NAME"  
  password: "YOUR_GIT_TOKEN_FOR_BOT_TO_ACCESS_REPO"
type: Opaque
```

```
kubectl apply -f YOUR_SECRET_FILE.yml
```

Verify secrets exist:
```
kubectl get secret -n argocd
```

## Apply ImageUpdater CR

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: "YOUR_IMAGE_UPDATER_NAME"
  namespace: argocd
spec:
  applicationRefs:
    - namePattern: "<PREFIXES_OF_YOUR_SERVICE_NAME>-*"
      useAnnotations: true
    - namePattern: "<PREFIXES_OF_YOUR_SERVICE_NAME>-*"
      useAnnotations: true
    - namePattern: "<SERVICE_N>-*"
      useAnnotations: true

```

```
kubectl apply -f YOUR_IMAGE_UPDATER_CR_FILE_NAME.yaml
```

Verify it's picking up your application:
```
kubectl get imageupdater -n argocd
```

## Configure Application Annotations
Add to your Application or ApplicationSet under `template.metadata`
```yaml
annotations:
    argocd-image-updater.argoproj.io/image-list: "{{service}}={{image}}"
    argocd-image-updater.argoproj.io/{{service}}.update-strategy: newest-build
    argocd-image-updater.argoproj.io/write-back-method: "git"
    argocd-image-updater.argoproj.io/write-back-target: "helmvalues:PATH_TO_YOUR_ENV_FILE/{{env}}.yaml"
    argocd-image-updater.argoproj.io/git-branch: YOUR_REPO_BRANCH
    argocd-image-updater.argoproj.io/{{service}}.helm.image-tag: app.tag
    argocd-image-updater.argoproj.io/{{service}}.helm.image-name: app.image
    argocd-image-updater.argoproj.io/{{service}}.pull-secret: "pullsecret:argocd/YOUR_PRIVATE_REGISTRY_URL"
    # argocd-image-updater.argoproj.io/ui.ignore-tags: "YOUR_IMAGE_TAG" # User for Rollback
```

Apply the updated Application/ApplicationSet:
```
kubectl apply -f YOUR_APPLICATIONSET_FILE_NAME.yml
```

## Allow Git Push
If the branch is protected, go to GitLab repo → **Settings → Repository → Protected Branches**, and add the bot account/token to **"Allowed to push and merge"** for the target branch.

## Configure Commit Message
Apply a custom commit message template using patch:
```yaml
kubectl patch configmap argocd-image-updater-config -n argocd --type merge -p '{"data":{"git.commit-message-template":"{{ range .AppChanges -}}\nUpdate {{ .Image }} image tag to {{ .NewTag }}\n{{ end -}}\n"}}'
```

Or edit manually:
```
kubectl edit configmap argocd-image-updater-config -n argocd
```
Add:
```yaml
data:
  git.commit-message-template: |
    {{ range .AppChanges -}}
    Update {{ .Image }} image tag to {{ .NewTag }}
    {{ end -}}
```

Restart to pick up the new config:
```
kubectl rollout restart deployment argocd-image-updater-controller -n argocd
```

## Verify Everything
Check the controller is running without errors:
```
kubectl logs -n argocd deployment/argocd-image-updater-controller --tail=50
```

Check the ImageUpdater CR status:
```
kubectl get imageupdater -n argocd -o wide
```

Check the Application is synced:
```
kubectl get application YOUR_APPLICATION_NAME -n argocd -o jsonpath='{.status.sync.status}'
```

## Test Update
Push a new image tag to the registry, then watch the logs:
```
kubectl logs -n argocd deployment/argocd-image-updater-controller -f
```

Check the Git repo for a new commit updating the values file, then check Argo CD synced to the new tag:
```
kubectl get application YOUR_APPLICATION_NAME -n argocd -o jsonpath='{.status.summary.images}'
```

If something doesn't work, check [Troubleshooting](./troubleshooting.md).
