# Webhook Setup Guide

## 1. Enable the webhook listener on Image Updater

**ConfigMap** (`argocd-image-updater-config`):
```yaml
data:
  webhook.enable: "true"
  webhook.port: "8082"
  registries.conf: |
    registries:
      - name: <Registry Name>
        prefix: <registry-host>:<port>
        api_url: http://<registry-host>:<port>
        insecure: yes
```
`registries.conf` is only needed if the registry serves plain HTTP instead of HTTPS.

**Deployment args**:
```
--enable-webhook
--disable-tls
--interval=6h
```
(`--interval` is the polling fallback — set it long since the webhook now handles real-time updates)

**Secret** (`argocd-image-updater-secret`):
```yaml
stringData:
  webhook.harbor-secret: <shared-secret>
```

If the registry's domain isn't resolvable by cluster DNS, add a `hostAliases` entry to the deployment:
```yaml
spec:
  template:
    spec:
      hostAliases:
        - ip: "<registry-host-ip>"
          hostnames: ["<registry-domain>"]
```

Apply and restart:
```bash
kubectl apply -f install-image-updater.yaml -n argocd
kubectl rollout restart deployment argocd-image-updater-controller -n argocd
```

## 2. Expose the webhook port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-image-updater-webhook
  namespace: argocd
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: argocd-image-updater   # must match the pod's actual label
  ports:
    - port: 8082
      targetPort: 8082
      nodePort: 30082
```

Confirm the selector matches the pod's real labels:
```bash
kubectl get pods -n argocd --show-labels | grep image-updater
```

## 3. Check NetworkPolicy allows the webhook port

If a NetworkPolicy already restricts traffic to the Image Updater pod, add the webhook port as its own separate ingress rule:
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
Apply and confirm both rules show up live:
```bash
kubectl get networkpolicy <policy-name> -n argocd -o yaml
```

## 4. Build and run the relay

A small Node/Express app that receives the registry's native push notification and forwards it to Image Updater in the format its webhook expects.

Key points:
- Must explicitly accept the registry's content type:
  ```js
  app.use(express.json({
    type: ['application/json', 'application/vnd.docker.distribution.events.v1+json']
  }));
  ```
- Only forward `action: "push"` events with a tag (skip untagged/digest-only pushes)
- Forward with the shared secret in the `Authorization` header

Run it persistently with Docker Compose:
```yaml
services:
  registry-relay:
    build: .
    container_name: registry-relay
    restart: unless-stopped
    ports:
      - "3031:3031"
    environment:
      HARBOR_WEBHOOK_SECRET: ${HARBOR_WEBHOOK_SECRET}
      IMAGE_UPDATER_URL: ${IMAGE_UPDATER_URL}
      REGISTRY_HOST: ${REGISTRY_HOST}
```
```bash
docker compose up -d --build
```

## 5. Point the registry at the relay

In `registry:2`'s `config.yml`:
```yaml
notifications:
  endpoints:
    - name: relay
      url: http://<relay-reachable-address>:3031/notify
      timeout: 2s
      threshold: 5
      backoff: 1s
```
Restart the registry container to apply.

## 6. Confirm the Application's image-list annotation matches

```yaml
argocd-image-updater.argoproj.io/image-list: ui=<registry-host>:<port>/<namespace>/<repo>
```
This must exactly match the repository name the relay sends, or Image Updater accepts the webhook (200 OK) but finds nothing to update.

## 7. Test end to end

```bash
docker tag alpine <registry>/<namespace>/<repo>:test1
docker push <registry>/<namespace>/<repo>:test1
```
Then check:
```bash
docker compose logs -f registry-relay
kubectl logs -n argocd deployment/argocd-image-updater-controller -f
```
Look for the relay forwarding the event, Image Updater processing the webhook, and `images_updated=1` in the log line. Confirm the app updates in the Argo CD UI without waiting for the polling interval.
