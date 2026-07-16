# Webhook Setup Guide

## Table of Contents
- Prerequisites
- Deployment Flow
- Enable Webhook on Image Updater
- Fix DNS
- Expose the Webhook Port
- Allow Webhook Traffic
- Build and Run the Relay
- Configure the Registry
- Configure Application Annotations
- Verify Everything
- Test Update

## Prerequisites

- Argo CD Image Updater is already installed and working (polling mode) — see [Image Updater Setup Guide](../setup.md)
- `kubectl` configured and connected to the cluster
- Access to edit the registry's `config.yml`
- A machine to run the relay (Docker + Docker Compose)

## Deployment Flow

1. Enable the webhook listener on Image Updater
2. Fix DNS if the registry host isn't resolvable
3. Expose the webhook port via a Service
4. Allow the webhook port through NetworkPolicy
5. Build and run the relay
6. Point the registry's notifications at the relay
7. Configure Application annotations to match
8. Verify and test

## Enable Webhook on Image Updater

Patch the ConfigMap:
```bash
kubectl patch configmap argocd-image-updater-config -n argocd --type merge -p '{"data":{"webhook.enable":"true","webhook.port":"8082"}}'
```

Or edit manually:
```bash
kubectl edit configmap argocd-image-updater-config -n argocd
```
Add:
```yaml
data:
  webhook.enable: "true"
  webhook.port: "YOUR_CUSTOM_PORT"
  registries.conf: |
    registries:
      - name: YOUR_REGISTRY_NAME
        prefix: YOUR_REGISTRY_HOST:PORT
        api_url: http://YOUR_REGISTRY_HOST:PORT
        insecure: yes
```
`registries.conf` is only needed if the registry serves plain HTTP instead of HTTPS.

Add the webhook flags to the Deployment args, and set a longer polling fallback since the webhook now handles real-time updates.

Patch:
```bash
kubectl patch deployment argocd-image-updater-controller -n argocd --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--enable-webhook"},
  {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--disable-tls"}
]'
```

Or edit manually:
```bash
kubectl edit deployment argocd-image-updater-controller -n argocd
```
Find `args:` under the container spec and update it to:
```yaml
args:
  - --metrics-bind-address=:8443
  - run
  - --interval=6h
  - --enable-webhook
  - --disable-tls
```

Add the shared secret Image Updater checks against:

Using kubectl create:
```bash
kubectl create secret generic argocd-image-updater-secret -n argocd \
  --from-literal=webhook.harbor-secret=YOUR_SHARED_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -
```

Or write it as YAML and apply:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-image-updater-secret
  namespace: argocd
type: Opaque
stringData:
  webhook.harbor-secret: YOUR_SHARED_SECRET
```
```bash
kubectl apply -f YOUR_SECRET_FILE.yaml
```

Restart :
```bash
kubectl rollout restart deployment argocd-image-updater-controller -n argocd
```

## Fix DNS

Check if the controller can resolve the registry host:
```bash
kubectl exec -it -n argocd deployment/argocd-image-updater-controller -- getent hosts YOUR_REGISTRY_HOST
```

If it fails, patch the deployment with a manual DNS mapping:
```bash
kubectl patch deployment argocd-image-updater-controller -n argocd --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/hostAliases",
    "value": [
      {"ip": "YOUR_REGISTRY_INTERNAL_IP", "hostnames": ["YOUR_REGISTRY_HOST"]}
    ]
  }
]'
```
**Double-check the IP is correct before applying** — a wrong IP silently routes to an unrelated service and is very hard to debug.

Or edit manually:
```bash
kubectl edit deployment argocd-image-updater-controller -n argocd
```
Find `Kind: Deployment` and add this under `spec.template.spec`, at the same level as `containers:`:
```yaml
hostAliases:
  - ip: "YOUR_REGISTRY_INTERNAL_IP"
    hostnames:
      - "YOUR_REGISTRY_HOST"
```

If a `hostAliases` entry already exists (e.g. for the Git host), add this as a second entry in the same list instead of replacing it:
```yaml
hostAliases:
  - ip: "YOUR_GIT_INTERNAL_IP"
    hostnames:
      - "YOUR_GIT_HOST"
  - ip: "YOUR_REGISTRY_INTERNAL_IP"
    hostnames:
      - "YOUR_REGISTRY_HOST"
```

## Expose the Webhook Port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-image-updater-webhook
  namespace: argocd
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: argocd-image-updater
  ports:
    - port: YOUR_CUSTOM_PORT #must match webhook.port in config map
      targetPort: YOUR_CUSTOM_PORT #must match webhook.port in config map
      nodePort: NODEPORT_RANGE
```
```bash
kubectl apply -f YOUR_SERVICE_FILE.yaml
```

Confirm the selector matches the pod's real labels:
```bash
kubectl get pods -n argocd --show-labels | grep image-updater
```

Confirm the Service has real endpoints:
```bash
kubectl get endpoints argocd-image-updater-webhook -n argocd
```

## Allow Webhook Traffic

Check for any NetworkPolicy restricting the Image Updater pod:
```bash
kubectl get networkpolicy -n argocd
```

If one exists and only allows metrics traffic, add the webhook port as its own separate ingress rule (not nested inside the existing rule).

Edit manually:
```bash
kubectl edit networkpolicy YOUR_POLICY_NAME -n argocd
```
```yaml
spec:
  ingress:
  - from: [...]           # existing rule, untouched
    ports:
    - port: 8443
      protocol: TCP
  - ports:                 # new, separate rule
    - port: 8082
      protocol: TCP
```

Or write it as YAML and apply:
```bash
kubectl apply -f YOUR_NETWORKPOLICY_FILE.yaml
```

Verify both rules are live:
```bash
kubectl get networkpolicy YOUR_POLICY_NAME -n argocd -o yaml
```

## Build and Run the Relay

A small Node/Express app that receives the registry's native push notification and forwards it to Image Updater in the format its webhook expects.

Key points:
- Must explicitly accept the registry's content type:
```js
app.use(express.json({
  type: ['application/json', 'application/vnd.docker.distribution.events.v1+json']
}));
```
- Only forward `action: "push"` events that have a tag (skip untagged/digest-only pushes)
- Forward with the shared secret in the `Authorization` header

Run persistently with Docker Compose:
```yaml
services:
  registry-relay:
    build: .
    container_name: registry-relay
    restart: unless-stopped
    ports:
      - "3031:3031"
    environment:
      HARBOR_WEBHOOK_SECRET: "YOUR_SHARED_SECRET"
      IMAGE_UPDATER_URL: "http://YOUR_WEBHOOK_ENDPOINT/webhook?type=harbor"
      REGISTRY_HOST: "YOUR_REGISTRY_HOST"
```
```bash
docker compose up -d --build
```

## Configure the Registry

Add to `registry:2`'s `config.yml`:
```yaml
notifications:
  endpoints:
    - name: relay
      url: "http://YOUR_RELAY_ADDRESS:3031/notify"
      timeout: 2s
      threshold: 5
      backoff: 1s
```
Restart the registry container to apply.

## Configure Application Annotations

Confirm the existing `image-list` annotation matches the registry path the relay sends:
```yaml
argocd-image-updater.argoproj.io/image-list: "{{service}}=YOUR_REGISTRY_HOST:PORT/YOUR_NAMESPACE/YOUR_REPO"
```
This must match exactly, or the webhook is accepted (200 OK) but finds nothing to update.

If this is set directly on an Application (not templated via ApplicationSet), you can also edit it manually:
```bash
kubectl edit application YOUR_APPLICATION_NAME -n argocd
```

## Verify Everything

Check the relay is running without errors:
```bash
docker compose logs -f registry-relay
```

Check the controller is receiving webhooks:
```bash
kubectl logs -n argocd deployment/argocd-image-updater-controller --tail=50
```

Check the webhook endpoint responds:
```bash
curl http://YOUR_WEBHOOK_ENDPOINT/healthz
```

## Test Update

Push a new image tag, then watch both logs:
```bash
docker push YOUR_REGISTRY_HOST/YOUR_NAMESPACE/YOUR_REPO:test1
```
```bash
docker compose logs -f registry-relay
kubectl logs -n argocd deployment/argocd-image-updater-controller -f
```

Look for the relay forwarding the event, Image Updater processing the webhook, and `images_updated=1` in the log line. Then confirm the app updated without waiting for the polling interval:
```bash
kubectl get application YOUR_APPLICATION_NAME -n argocd -o jsonpath='{.status.summary.images}'
```

If something doesn't work, check Troubleshooting.
