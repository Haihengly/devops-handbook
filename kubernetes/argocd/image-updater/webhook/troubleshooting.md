# Webhook Troubleshooting

## Webhook received (200 OK) but nothing updates

Check the controller logs for `images_updated=0`:
```bash
kubectl logs -n argocd deployment/argocd-image-updater-controller --tail=50
```
This means the webhook was accepted, but the repository name in the payload didn't match the app's `image-list` annotation. Compare:
- What the relay is actually sending (add a debug log of the payload right before it's forwarded)
- The exact value in `argocd-image-updater.argoproj.io/image-list` on the Application

They need to match exactly, including registry host/port if your annotation includes it.

## `dial tcp: lookup YOUR_HOST: no such host`

Cluster DNS (CoreDNS) can't resolve the registry's hostname. Confirm with:
```bash
kubectl exec -it -n argocd deployment/argocd-image-updater-controller -- getent hosts YOUR_REGISTRY_HOST
```
Fix with a `hostAliases` entry on the deployment (see Setup Guide → Fix DNS).

**If this only happens on `argocd-repo-server`, not `argocd-image-updater-controller`** — each Deployment needs its own separate `hostAliases` entry. They don't share `/etc/hosts` across pods.

**Double, triple check the IP you enter.** A typo'd IP here doesn't cause a clean failure — it silently routes traffic to a completely unrelated service, which can look like a totally different, more confusing bug (auth errors, wrong HTML responses, etc) with no obvious connection back to a wrong IP.

## Webhook port unreachable, even though the Service and endpoints look correct

Check if a NetworkPolicy is silently blocking it:
```bash
kubectl get networkpolicy -n argocd
kubectl get networkpolicy YOUR_POLICY_NAME -n argocd -o yaml
```
If a policy selects the Image Updater pod and only allows a different port (e.g. metrics on 8443), any port not explicitly listed is blocked — the pod itself, the Service, and the endpoints can all look perfectly fine while the actual traffic is silently dropped.

Add the webhook port as its own separate ingress rule (see Setup Guide → Allow Webhook Traffic). Don't nest it under an existing rule's `ports:` — a duplicate `ports:` key inside the same rule silently keeps only the last one and drops the first.

## `http: server gave HTTP response to HTTPS client`

The client (Docker, Helm, or Image Updater itself) is trying HTTPS against a registry that only serves plain HTTP.

- **Docker** — add the registry to `insecure-registries` in `daemon.json`, then restart the daemon.
- **Helm CLI** — use `--plain-http` on the specific command (`helm pull`, `helm push`, `helm registry login`, `helm dependency build`). There's no persistent config file for this — the flag is needed every time.
- **Image Updater** (checking tags from the registry) — add a `registries.conf` entry in the ConfigMap with `api_url: http://...` and `insecure: yes` (see Setup Guide → Enable Webhook on Image Updater).

## Relay logs "Cannot POST /webhook" (Express 404 page)

This means the request isn't reaching Image Updater at all — it's hitting some other Express app, most likely the relay itself. Check `IMAGE_UPDATER_URL` is actually set correctly:
```bash
docker compose exec registry-relay env | grep IMAGE_UPDATER_URL
```
If you just changed the `.env` file, remember env var changes don't apply to an already-running container — restart it:
```bash
docker compose up -d --force-recreate registry-relay
```

## Relay parses the payload as `{}` (empty)

`registry:2` sends its notification with `Content-Type: application/vnd.docker.distribution.events.v1+json`, not `application/json`. Express's default JSON parser only accepts `application/json` — it silently skips parsing anything else. Make sure `express.json()` explicitly allows both:
```js
app.use(express.json({
  type: ['application/json', 'application/vnd.docker.distribution.events.v1+json']
}));
```

## Helm chart dependencies (Chart.yaml `dependencies:`) fail with auth or HTTPS errors, even though the main Application source works fine

This is a known limitation on ArgoCD 2.x. Application main sources and Chart.yaml dependencies use two different, independent credential/config lookups:

- Main Application source → uses a `repository`-type Secret (exact URL match)
- Chart.yaml dependency → uses a `repo-creds`-type Secret (URL prefix match)

Having only a `repository`-type Secret means the dependency pull finds no credentials at all ("no basic auth credentials"), even if the main source authenticates fine.

Even after fixing the credential type, ArgoCD 2.x has a further bug where `helm dependency build` and `helm pull` are called with `--insecure-skip-tls-verify` instead of `--plain-http` — so a plain-HTTP-only registry still fails with `http: server gave HTTP response to HTTPS client`, no matter what the repository Secret's `insecure` field is set to. This affects single-source Applications, multi-source Applications, and Chart.yaml dependencies alike.

**There is no workaround on ArgoCD 2.x.** Confirmed fixed in 3.1+ for `type: oci` credentials (not `type: helm`), with a dedicated flag (`--insecure-oci-force-http`) added in 3.5+. The only real fixes are:
1. Put real TLS (nginx + certbot) in front of the chart registry, or
2. Upgrade ArgoCD to 3.1+ and switch to `type: oci` credentials
