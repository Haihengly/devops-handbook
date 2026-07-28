# Jenkins Shared Library — DevOps Pipeline

## Table of Contents
- [What This Is](#what-this-is)
- [Repo Layout](#repo-layout)
- [How It Works (High Level)](#how-it-works-high-level)
- [Pipeline Flow](#pipeline-flow)
- [Stage Types](#stage-types)
- [Status](#status)
- [Related Docs](#related-docs)

## What This Is
A dynamic, config-driven Jenkins pipeline shared across every service. Instead of writing a full `Jenkinsfile` per service, each service has a tiny Jenkinsfile that just declares a config map (project name, service name, which stages to run) and hands it off to `corePipeline()`. All the actual logic (checkout, deploy, notify, etc.) lives centrally in the shared library, so fixing or improving the pipeline once fixes it everywhere.

This setup is split across **three repos**:

| Repo | Purpose |
|---|---|
| `Jenkinsfile` repo | One folder per project, one Jenkinsfile per service, used with Jenkins Multibranch |
| `environment-file` repo | Per-project, per-service YAML files with env-specific values (dev/uat/prod), loaded via `envLoader.groovy` |
| `share-library` repo | The actual pipeline logic — `corePipeline` + all `stageX` / `configX` steps |

## Repo Layout

**Jenkinsfile repo** (per project folder):
```
Jenkinsfile_Repo
├── Project_A
│   ├── Service_A.jenkinsfile
│   ├── Service_B.jenkinsfile
│   └── Service_N.jenkinsfile
├── Project_B
│   ├── Service_A.jenkinsfile
│   ├── Service_B.jenkinsfile
│   └── Service_N.jenkinsfile
└── Project_N
    ├── Service_A.jenkinsfile
    ├── Service_B.jenkinsfile
    └── Service_N.jenkinsfile
```

**environment-file repo**:
```
environment-file_Repo
├── vars/
│   └── envLoader.groovy
└── resources/
    ├── Project_A/
    │   ├── Service_A.yml
    │   ├── Service_B.yml
    │   └── Service_N.yml
    ├── Project_B/
    │   ├── Service_A.yml
    │   ├── Service_B.yml
    │   └── Service_N.yml
    └── Project_N/
        ├── Service_A.yml
        ├── Service_B.yml
        └── Service_N.yml
```

**share-library repo**:
```
share-library/vars/
├── corePipeline.groovy      # entrypoint, calls everything below
├── configEnvLoader.groovy   # pulls env-specific config in
├── configSanitize.groovy    # normalizes stage config (e.g. enabled → real Boolean)
├── configValidate.groovy    # checks config is well-formed before running
├── config_N.groovy          # config_N_thing 
├── stageExecutor.groovy     # routes stage type → stage function
├── stageCheckout.groovy     # git checkout
├── stageDeploy.groovy       # SSH deploy via docker compose
└── stage_N.groovy           # stage_N
```

## How It Works (High Level)

```
Jenkinsfile (per service)
   ↓ defines config map, calls corePipeline(config)
corePipeline
   ↓ 1. configEnvLoader   → merges in env.yml values for current branch
   ↓ 2. configSanitize    → cleans up stage config (enabled: "true" → true)
   ↓ 3. configValidate    → fails fast if config is malformed
   ↓ 4. loop config.stages → stageExecutor(type, config) per enabled stage
   ↓ 5. post block         → telegramNotify.notify(SUCCESS/FAILURE/UNSTABLE)
```

## Pipeline Flow

1. **Jenkinsfile** loads 3 libraries (`env-configs`, `pipeline`, `notify`), builds a `config` map:
   ```groovy
   def config = [
       PROJECT_NAME: 'YOUR_PROJECT_NAME',
       SERVICE_NAME: 'YOUR_SERVCICE_NAME',
       PROJECT_URL: 'YOUR_PROJECT_REPO_URL',
       stages: [
           [name: 'Check',  type: 'check',  enabled: 'true'],
           [name: 'Deploy', type: 'deploy', enabled: 'false'],
           [name: 'N', type: 'N', enabled: 'true,false,TRUE,FALSE'] # doesn't care about upercase or lowercase 
       ]
   ]
   corePipeline(config)
   ```
   Every service's Jenkinsfile is identical in structure, only the values change.

2. **`configEnvLoader`** checks `PROJECT_NAME`, `SERVICE_NAME`, `PROJECT_URL` are set, then calls `envLoader.getEnv(PROJECT_NAME, SERVICE_NAME, env.BRANCH_NAME)` from the `env-configs` library. That function maps the current git branch (`main`/`uat`/`dev`) to a YAML section in `resources/<project>/<service>.yml`, and returns that section (e.g. `SERVER`, `PROJECT_PATH`, `SOURCE_BRANCH`). Those values get merged into `config`.

3. **`configSanitize`** rebuilds `config.stages`, converting `enabled` from a string (`"true"`/`"false"`) into an actual Groovy Boolean, and rejects anything that isn't one of those two values.

4. **`configValidate`** makes sure `config.stages` isn't empty and every stage has `name`, `type`, and a proper Boolean `enabled`.

5. **Dynamic stage loop** — `corePipeline` iterates `config.stages`. Disabled stages are skipped with a log line. Enabled stages run inside `catchError` (so one failing stage doesn't hard-crash the whole build) and get dispatched via `stageExecutor(type, config)`.

6. **`stageExecutor`** is a registry/router: it maps a `type` string to a stage function and calls it, logging success/failure.

7. **`post` block** always fires `telegramNotify.notify(...)` with the build result.

## Stage Types

| type | Function | Status |
|---|---|---|
| `check` | `stageCheckout` | ✅ implemented — GitSCM checkout using `SOURCE_BRANCH` + `PROJECT_URL` |
| `deploy` | `stageDeploy` | ✅ implemented — SSH into `SERVER`, pull latest, `docker compose up -d --build` |
| `build` | `stageBuild` | ⚠️ **not implemented** — referenced in registry, function doesn't exist yet |
| `cleanup` | `stageCleanUp` | ⚠️ **not implemented** — same |
| `test` | `stageTest` | ⚠️ **not implemented** — same |

If a Jenkinsfile enables a `build`, `cleanup`, or `test` stage right now, the pipeline will fail at `stageExecutor` with an unknown-function error (not a clean "unknown stage type" — Groovy will blow up trying to call a function that doesn't exist). Keep those stages `enabled: 'false'` until the functions are written.

## Status
This library is **~60–80% complete**, functional for the core check → deploy flow across ~30 services in dev/UAT. Known gaps and rough edges are tracked in `troubleshooting.md`.

## Related Docs
- [`setup.md`](./setup.md) — how to wire up a new service to use this library
- [`troubleshooting.md`](./troubleshooting.md) — known issues, gotchas, gaps
