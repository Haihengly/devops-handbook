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
export KUBE_EDITOR=nano
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
export KUBE_EDITOR=nano
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
export KUBE_EDITOR=nano
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
export KUBE_EDITOR=nano
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
    - port: YOUR_CUSTOM_PORT #match webhook.port on config map
      protocol: TCP
```

Verify both rules are live:
```bash
kubectl get networkpolicy YOUR_POLICY_NAME -n argocd -o yaml
```

## Build and Run the Relay

A small Node/Express app that receives the registry's native push notification, transforms it into the payload format Image Updater's webhook expects, and forwards it.

Project structure:
```
registry-relay/
├── docker-compose.yml
├── Dockerfile
├── package.json
├── server.js
└── .env
```

**`package.json`**
```json
{
  "name": "registry-relay",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.19.2"
  }
}
```

**`Dockerfile`**
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json .
RUN npm install --production

COPY server.js .

EXPOSE 3031

CMD ["node", "server.js"]
```

**`server.js`**
```javascript
const express = require('express');
const app = express();

app.use(express.json({
  type: ['application/json', 'application/vnd.docker.distribution.events.v1+json']
}));

const HARBOR_SECRET = process.env.HARBOR_WEBHOOK_SECRET;
if (!HARBOR_SECRET) {
  console.error('FATAL: HARBOR_WEBHOOK_SECRET not set');
  process.exit(1);
}

const IMAGE_UPDATER_URL = process.env.IMAGE_UPDATER_URL;
if (!IMAGE_UPDATER_URL) {
  console.error('FATAL: IMAGE_UPDATER_URL not set');
  process.exit(1);
}

const REGISTRY_HOST = process.env.REGISTRY_HOST;
if (!REGISTRY_HOST) {
  console.error('FATAL: REGISTRY_HOST not set');
  process.exit(1);
}

app.post('/notify', async (req, res) => {
  try {
    const events = req.body.events || [];

    for (const event of events) {
      if (event.action !== 'push') continue;
      if (!event.target?.tag) continue; // skip untagged/digest-only pushes

      const repoFull = event.target.repository; // e.g. "namespace/repo"
      const slashIdx = repoFull.indexOf('/');
      const namespace = slashIdx !== -1 ? repoFull.slice(0, slashIdx) : '';
      const name = slashIdx !== -1 ? repoFull.slice(slashIdx + 1) : repoFull;

      const harborPayload = {
        type: 'PUSH_ARTIFACT',
        occur_at: Math.floor(Date.now() / 1000),
        operator: event.actor?.name || 'registry',
        event_data: {
          resources: [
            {
              digest: event.target.digest,
              tag: event.target.tag,
              resource_url: `${REGISTRY_HOST}/${repoFull}:${event.target.tag}`
            }
          ],
          repository: {
            name,
            namespace,
            repo_full_name: repoFull
          }
        }
      };

      console.log(`Push detected: ${repoFull}:${event.target.tag} -> forwarding to Image Updater`);

      const resp = await fetch(IMAGE_UPDATER_URL, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': HARBOR_SECRET
        },
        body: JSON.stringify(harborPayload)
      });

      console.log(`Image Updater response: ${resp.status}`);
      if (!resp.ok) {
        console.error(`Rejected: ${await resp.text()}`);
      }
    }

    res.status(200).json({ status: 'ok' });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'failed to forward event' });
  }
});

app.get('/healthz', (req, res) => res.status(200).send('ok'));

app.listen(3031, () => console.log('Relay listening on 3031'));
```

**`docker-compose.yml`**
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

**`.env`** (same folder, keep out of git)
```
HARBOR_WEBHOOK_SECRET=YOUR_SHARED_SECRET
IMAGE_UPDATER_URL=http://YOUR_WEBHOOK_ENDPOINT/webhook?type=harbor
REGISTRY_HOST=YOUR_REGISTRY_HOST
```

Run it:
```bash
docker compose up -d --build
```

Check it's running:
```bash
docker compose ps
docker compose logs -f registry-relay
```


## Configure the Registry

If `config.yml` is host-mounted (check your registry's `docker-compose.yml` for a volume like `./docker/registry:/etc/docker/registry`), edit it directly on the host — no need to go into the container:
```bash
nano ./docker/registry/config.yml
```

If it's not mounted, copy it out of the container, edit it, then copy it back:
```bash
docker cp YOUR_REGISTRY_CONTAINER:/etc/docker/registry/config.yml ./config.yml
nano ./config.yml
docker cp ./config.yml YOUR_REGISTRY_CONTAINER:/etc/docker/registry/config.yml
```

Add the `notifications` block (same indentation level as `storage`, `http`, `health` — not nested inside any of them):
```yaml
notifications:
  endpoints:
    - name: relay
      url: "http://YOUR_RELAY_ADDRESS:3031/notify"
      timeout: 2s
      threshold: 5
      backoff: 1s
```

Restart the registry container to apply:
```bash
docker restart YOUR_REGISTRY_CONTAINER
```
Or if using Docker Compose:
```bash
docker compose restart registry
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

If something doesn't work, check [Troubleshooting](./troubleshooting.md).
