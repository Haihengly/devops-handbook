# Setup — Onboarding a New Service to the Shared Library

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Add Environment Values](#step-1-add-environment-values)
- [Step 2: Add the Jenkinsfile](#step-2-add-the-jenkinsfile)
- [Step 3: Create the Multibranch Job](#step-3-create-the-multibranch-job)
- [Step 4: Verify](#step-4-verify)

## Prerequisites
- `gitlab-account` credential exists in Jenkins (used by `stageCheckout`)
- `hengly-ssh-creds` (username/password) credential exists in Jenkins (used by `stageDeploy`)
- The 3 shared libraries are registered in Jenkins global config: `env-configs`, `pipeline`, `notify`
- Target deploy server is reachable via SSH from the Jenkins host, and already has the project cloned at the path you'll set below

## Step 1: Add Environment Values

In the `environment-file` repo, create (or edit) `resources/YOUR_PROJECT_NAME/YOUR_SERVICE_NAME.yml`:

```yaml
dev:
  SERVER: "YOUR_DEV_SERVER_IP_OR_HOST"
  PROJECT_PATH: "/path/to/project/on/server"
  SOURCE_BRANCH: "dev"

uat:
  SERVER: "YOUR_UAT_SERVER_IP_OR_HOST"
  PROJECT_PATH: "/path/to/project/on/server"
  SOURCE_BRANCH: "uat"

prod:
  SERVER: "YOUR_PROD_SERVER_IP_OR_HOST"
  PROJECT_PATH: "/path/to/project/on/server"
  SOURCE_BRANCH: "main"
```

Notes:
- File path must be exactly `resources/<PROJECT_NAME>/<SERVICE_NAME>.yml` — this is looked up dynamically via `libraryResource()`, so a typo in either name means an empty/missing config, not a clear "file not found."
- `SOURCE_BRANCH` here is what `stageCheckout` uses for `GitSCM` branches — keep it consistent with your branch strategy (`dev`/`uat`/`main`).
- Branch name → env section mapping is hardcoded in `envLoader.groovy`: `main → prod`, `uat → uat`, `dev → dev`. Any other branch name will hard-fail the build. If a service needs a different branch name, that mapping has to change in the shared lib, not per-service.

## Step 2: Add the Jenkinsfile

In the `Jenkinsfile` repo, under `Jenkinsfile/YOUR_PROJECT_NAME/`, create `YOUR_SERVICE_NAME.jenkinsfile`:

```groovy
@Library(['env-configs@master', 'pipeline@v1.0.0', 'notify@master']) _

def config = [
    PROJECT_NAME: 'YOUR_PROJECT_NAME',
    SERVICE_NAME: 'YOUR_SERVICE_NAME',
    PROJECT_URL: 'YOUR_GIT_REPO_URL',   // must NOT be blank — configEnvLoader fails on empty string
    stages: [
        [name: 'Check',  type: 'check',  enabled: 'true' ],
        [name: 'Deploy', type: 'deploy', enabled: 'false']   // flip on when ready
    ]
]

corePipeline(config)
```

Important:
- `PROJECT_URL` must be a real, non-empty value — `configEnvLoader` does `if (!config.PROJECT_URL) error(...)`, and an empty string is falsy in Groovy, so this will hard-fail if left blank.
- Only `check` and `deploy` stage types currently work. Do not set `build`, `cleanup`, or `test` to `enabled: 'true'` yet — see [troubleshooting.md](./troubleshooting.md).
- `enabled` should be the string `'true'` or `'false'` (matching the existing pattern) — `configSanitize` converts it to a real Boolean for you.

## Step 3: Create the Multibranch Job

1. New Item → Multibranch Pipeline
2. Branch source → your git repo for the service
3. Build Configuration → "by Jenkinsfile" → Script Path → point at `YOUR_SERVICE_NAME.jenkinsfile` in the Jenkinsfile repo (this repo is separate from the service's own source repo, since you're using centralized Jenkinsfiles)
4. Scan the repo, confirm `dev`/`uat`/`main` branches are picked up

## Step 4: Verify

- Trigger a build on the `dev` branch with only `Check` enabled first.
- Confirm in the build log:
  - `✅ Config loaded successfully!` (from `configEnvLoader`)
  - `✅ Config validation passed` (from `configValidate`)
  - `Executing stage: check` → `✅ Stage 'check' completed successfully`
- Confirm you got a Telegram notification (success/failure/unstable) from `telegramNotify`.
- Once `Check` is solid, flip `Deploy` to `enabled: 'true'` and re-test. Watch for the deploy stage's hardcoded `docker compose up -d --build web` — see troubleshooting for what that means for non-`web` services.
