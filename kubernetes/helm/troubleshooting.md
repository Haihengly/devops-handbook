# Troubleshooting - Helm Environment Management

This document lists common issues and how to identify them in a Helm + ArgoCD setup using library charts, per-environment values, and an OCI registry.

## Chart Pull Fails with Auth Error

- ArgoCD repository `Secret` missing, or not labeled `argocd.argoproj.io/secret-type: repository`
- Secret's `url` field doesn't match the registry path the chart actually requests
- `argocd-repo-server` hasn't been restarted since the secret was created or edited
- Credentials are correct but were never tested manually with `helm registry login` to rule out a typo

## Chart Pull Fails with a DNS / Hostname Error

- Secret's `url` field has an `oci://` prefix on it — this field should be the bare host and path only, no scheme
- The `oci://` prefix belongs only in `Chart.yaml`'s `dependencies[].repository` field, not in the secret
- Mixing these two up produces a broken hostname lookup, since the scheme gets parsed as if it were part of the host

## Sync Shows the Same Error After a Fix Was Already Applied

- Error message includes `(cached)` — this means a stale manifest-generation result is being shown, not a fresh retry
- A hard refresh on the Application wasn't triggered
- `argocd-repo-server` wasn't restarted after the secret/config change, so it's still working from what it loaded at startup

## Helm Template Fails to Render

- `Error: library charts are not installable` — the chart being rendered has `type: library` in its `Chart.yaml` and was never meant to be deployed standalone; point the command at a chart that consumes it instead
- `parse error ... unexpected EOF` — an `{{- if }}`, `{{- range }}`, `{{- define }}`, or `{{- with }}` block is missing its matching `{{- end }}`; check the reported line and trace backward
- Dependency not pulled yet — run `helm dependency update` before templating if the chart declares any dependencies

## Wrong Values Being Applied

- `-f` flag pointing at the wrong file or wrong path
- Multiple `valueFiles` listed — remember later files override earlier ones on any overlapping key, so order matters
- Forgetting that `helm template .` alone only uses the chart's own default `values.yaml`, not anything inside a `values/` subfolder, unless explicitly passed with `-f`

## ApplicationSet Generates Wrong or Unwanted Applications

- Used a `matrix` generator when environments are actually asymmetric across services (e.g. one service has no `uat`) — matrix always produces the full cross-product of both lists
- A `list` generator has a typo in `service`, `env`, or `repository` that silently produces a malformed Application name or path
- Forgot `CreateNamespace=true` in `syncOptions`, so the Application fails because the target namespace doesn't exist yet

## Registry Accepts a Push That Shouldn't Have Worked

- Self-hosted `registry:2` does not enforce tag/version immutability by default — pushing the same version twice with different content silently overwrites it, no error or warning
- This is a registry policy choice, not a Helm/OCI spec requirement — managed registries often behave differently
- If this happens unintentionally, any cached copy of that version elsewhere (a teammate's machine, a different ArgoCD instance) has no way to know the content changed underneath the same version tag

## General Debug Checklist

Check the secret ArgoCD is actually using:

```
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

Check repo-server logs for the real pull attempt and error:

```
kubectl logs -n argocd deploy/argocd-repo-server --tail=50
```

Check an Application's actual sync and health state:

```
kubectl describe application <app-name> -n argocd
```

Test registry credentials directly, bypassing ArgoCD entirely:

```
helm registry login <registry-url> -u <username> -p <password>
```

Force a clean retry after any secret or config change:

```
kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd
```
