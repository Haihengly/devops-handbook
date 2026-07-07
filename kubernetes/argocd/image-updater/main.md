# Argo CD Image Updater

This project sets up automatic image tag updates using Argo CD Image Updater and GitLab. Instead of Jenkins manually updating the manifest repo, Image Updater watches the image registry and updates the tag by itself.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [What It Includes](#what-it-includes)
- [Update Strategy](#update-strategy)
- [How It Works](#how-it-works)
- [Deployment Flow](#deployment-flow)
- [Documentation](#documentation)
- [Notes](#notes)

## Overview
Before, Jenkins had to checkout the manifest repo, update the image tag, and push it back to Git. Now, Argo CD Image Updater does this automatically. It watches the image registry, and when a new tag shows up, it updates the Git repo itself. Argo CD then syncs the change like normal.

## Architecture
```
Image Registry → Image Updater → Git Repo (GitLab) → Argo CD Sync → Kubernetes Deployment
```

## What It Includes
- Argo CD Image Updater controller 
- ApplicationSet for environments
- Private registry login using a pull-secret
- Git credentials so it can push tag updates
- One `ImageUpdater` CR that use with every app
- Custom commit message so Git history is easy to read

## Update Strategy
If your image tags are just plain numbers (1, 2, 3...), not version numbers like `v1.2.0`. Then use the `newest-build` strategy — it picks whichever image was pushed most recently, instead of sorting by tag name.

| Strategy | When to use |
|---|---|
| `newest-build` | Plain/incrementing tag numbers (what we use) |
| `semver` | Tags like `x.x.x` e.g. `1.2.0` |
| `digest` | Only update to one specific image, manually |

**Important:** if someone deletes the currently deployed tag, Image Updater will automatically go back to the next available tag on its own — no warning, no approval. So never delete the tag that's currently deployed. See [Troubleshooting](./troubleshooting.md).

## How It Works
Image Updater edits the environment's values file directly (`what ever your custom env file`), so the current tag is always visible in one place.

```
1. Checks the registry every ~2 minutes
2. Finds a new tag
3. Clones the Git repo
4. Updates your custom env file with the new tag
5. Commits and pushes to git
6. Argo CD sees the new commit and syncs automatically
```

## Deployment Flow
1. Install Image Updater
2. Set up registry login and Git credentials
3. Apply the `ImageUpdater` CR
4. Add Image Updater annotations to the Application or Applicationset
5. Allow the Git credential to push to the protected branch
6. Test by pushing a new tag and checking it updates correctly

## Notes
- Only used in **dev and uat** for now, not production yet
- One `ImageUpdater` CR use with every app — adding a new service just means adding `namePattern` for CR and `annotations` for Application or Applicationset
- To roll back, you need to both revert the Git commit **and** add an `ignore-tags` annotation, or Image Updater will just re-apply the same tag again. See [Troubleshooting](./troubleshooting.md)
- Built with the goal of scaling a lot of services without repeating pipeline code for each one

## Documentation
- [Setup Guide](./setup.md)
- [Troubleshooting](./troubleshooting.md)

## Repository
GitHub Repository: 
