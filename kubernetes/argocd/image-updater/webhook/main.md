# Image Updater Webhook

This adds a webhook trigger on top of Image Updater, so a new image tag is picked up the moment it's pushed, instead of waiting on the polling interval. It also takes load off the registry, since Image Updater no longer needs to check it constantly.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [What It Includes](#what-it-includes)
- [How It Works](#how-it-works)
- [Documentation](#documentation)
- [Notes](#notes)

## Overview

Before, Image Updater checked the registry on a fixed interval (e.g. every 2 minutes) to look for new tags. This works, but it means every push waits up to one interval before it's picked up, and the registry gets polled constantly whether or not anything changed.

Now, the registry notifies a relay the moment an image is pushed. The relay forwards that event to Image Updater's webhook, which checks the tag immediately and updates the app. Polling is kept as a long-interval fallback (e.g. every 6 hours), in case a push notification is ever missed.

## Architecture

```
docker push → registry:2 → Relay (Node.js) → Image Updater webhook → Git Repo (GitLab) → Argo CD Sync → Kubernetes Deployment
```

## What It Includes

- Image Updater's built-in webhook listener (enabled alongside its normal polling)
- A custom relay that translates the registry's native push notification into the payload format Image Updater's webhook expects
- A Kubernetes Service exposing the webhook port so the relay (running outside the cluster) can reach it
- A shared secret between the relay and Image Updater, so the webhook endpoint can't be triggered by anyone else
- `hostAliases` entries where cluster DNS can't resolve the registry's domain directly

## How It Works

1. An image is pushed to the registry
2. The registry fires its native push notification to the relay
3. The relay transforms the notification into the payload format Image Updater's webhook expects, and forwards it with the shared secret
4. Image Updater receives the webhook, checks the registry for the new tag
5. Image Updater updates the environment's values file and pushes the commit to Git, same as it does on a normal polling cycle
6. Argo CD sees the new commit and syncs automatically

This sits on top of the existing Image Updater setup — everything about how tags get written back to Git, update strategy, and Argo CD syncing stays the same. The only thing this changes is *when* Image Updater checks the registry.

## Notes

- Polling is still enabled as a fallback, just set to a much longer interval, since the webhook now handles the real-time case
- The relay runs as its own persistent service (Docker Compose), separate from the cluster
- This only affects timing — everything downstream (Git write-back, ignore-tags for rollback, etc) is unaffected and still documented under [Image Updater's own troubleshooting](../troubleshooting.md)

## Documentation

- [Setup Guide](./setup.md)
- [Troubleshooting](./troubleshooting.md)
