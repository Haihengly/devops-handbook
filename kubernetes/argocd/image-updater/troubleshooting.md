# Troubleshooting

## Table of Contents
- [No ImageUpdater CRs to process](#no-imageupdater-crs-to-process)
- [DNS: could not resolve Git host](#dns-could-not-resolve-git-host)
- [could not get creds for repo: unknown repository type](#could-not-get-creds-for-repo-unknown-repository-type)
- [git push rejected on protected branch](#git-push-rejected-on-protected-branch)
- [could not find an image-name for image](#could-not-find-an-image-name-for-image)
- [write-back-target not taking effect](#write-back-target-not-taking-effect)
- [Commit message not changing](#commit-message-not-changing)
- [Rollback gets overwritten automatically](#rollback-gets-overwritten-automatically)
- [Deleted the latest tag by accident](#deleted-the-latest-tag-by-accident)

## No ImageUpdater CRs to process
**Symptom:** Controller logs show `No ImageUpdater CRs to process`, annotations on the Application are completely ignored.

**Cause:** This version of Image Updater requires an actual `ImageUpdater` CR to exist. Annotations alone do nothing without it.

**Fix:** Apply an `ImageUpdater` CR with a `namePattern` matching your app(s):
```
kubectl apply -f imageupdater-cr.yml
kubectl get imageupdater -n argocd
```

## DNS: could not resolve Git host
**Symptom:** `wget` or `git clone` from inside the controller pod fails with `bad address` or `NXDOMAIN`.

**Cause:** Internal/company Git hosts are often not resolvable by the cluster's default DNS.

**Check:**
```
kubectl exec -it -n argocd deployment/argocd-image-updater-controller -- getent hosts your-gitlab-host.com
```

**Fix:** Add a `hostAliases` entry — see [Setup Guide](./setup.md#fix-dns).

## could not get creds for repo: unknown repository type
**Symptom:** Log shows this exact error when using `write-back-method: "git:secret:namespace/secret"`, even though the secret and URL are correct.

**Cause:** This is a known issue in the credential-lookup code path used specifically by the `git:secret:` write-back method.

**Fix:** Use plain `git` as the write-back method instead of `git:secret:...`. This reuses Argo CD's own already-registered credential for the repo instead:
```yaml
argocd-image-updater.argoproj.io/write-back-method: "git"
```
Requirement: Argo CD's existing repo credential must have write (push) access, not just read.

## git push rejected on protected branch
**Symptom:**
```
remote: GitLab: You are not allowed to push code to protected branches on this project.
! [remote rejected] master -> master (pre-receive hook declined)
```

**Cause:** The Git credential is valid, but the branch is protected and the account/token isn't allowed to push to it. Having a project role like Developer is not enough — protected branch rules are separate from project role permissions.

**Fix:** GitLab repo → **Settings → Repository → Protected Branches** → add the bot account/token to **"Allowed to push and merge"** for the target branch.

## could not find an image-name for image
**Symptom:** Log shows `could not find an image-name for image <image>` when using `write-back-target: helmvalues`.

**Cause:** The `helmvalues` write-back target writes both image name and tag by default. If only `helm.image-tag` is set and `helm.image-name` is missing, it has nowhere to write the name.

**Fix:** Add the missing annotation, pointing to a key in your values file:
```yaml
argocd-image-updater.argoproj.io/ui.helm.image-name: app.image
```

## write-back-target not taking effect
**Symptom:** Set `writeBackTarget` correctly in the ImageUpdater CR, but it still creates the default `.argocd-source-<app>.yaml` file instead of editing the values file directly.

**Cause:** If the CR uses `useAnnotations: true`, config comes from the Application's annotations, not the CR's `writeBackConfig`. Without a matching annotation, it falls back to default behavior.

**Fix:** Set it via annotation instead:
```yaml
argocd-image-updater.argoproj.io/write-back-target: "helmvalues:env/dev.yaml"
```

## Commit message not changing
**Symptom:** Updated `git.commit-message-template` in the ConfigMap, but commits still show the old default message.

**Cause:** The controller reads this ConfigMap once at startup. It does not hot-reload when the ConfigMap changes.

**Fix:** Restart the controller after any ConfigMap change:
```
kubectl rollout restart deployment argocd-image-updater-controller -n argocd
```

## Rollback gets overwritten automatically
**Symptom:** Reverted the bad commit or ran `argocd app rollback`, but the broken tag comes back within a few minutes.

**Cause:** Two automated processes are still working against the rollback:
- Argo CD's `selfHeal` re-syncs to whatever git says
- Image Updater re-detects the same "newest" tag on its next poll and writes it back again

**Fix:** Do both of these together, not just one:
1. Revert the commit in git
2. Add an `ignore-tags` annotation for the broken tag so Image Updater stops considering it:
```yaml
argocd-image-updater.argoproj.io/ui.ignore-tags: "YOUR_IMAGE_TAG"
```

## Deleted the latest tag by accident
**Symptom:** Deleted the currently-deployed image tag from the registry, and the deployment automatically changed to the previous tag with no warning.

**Cause:** `newest-build` (and `semver`) strategies re-evaluate "what's newest" on every poll cycle. If the current tag disappears, it just falls back to whatever's next — this is expected behavior, not a bug.

**Fix / Prevention:** Never delete the currently-deployed tag. Only clean up tags that are several versions behind the current one. Consider registry-level tag protection if supported.
