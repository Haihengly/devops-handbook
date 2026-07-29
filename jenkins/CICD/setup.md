# Setup — Onboarding a New Service to the Shared Library

## Table of Contents
- [Add libraries in jenkins global](#add-libraries-in-jenkins-global)
- [Add the Jenkinsfile](#add-the-jenkinsfile)
- [Add Environment Values](#add-environment-values)
- [Create the Multibranch Job](#create-the-multibranch-job)
- [Verify](#verify)

## Add libraries in jenkins global

1. Go to Manage Jenkins -> System (Configure global setting and paths)

2. Scroll down to Global Trusted Pipeline Libraries -> Click Add Button 

3. Leave it as default only edit the following

- Name : YOUR_LIBRARY_NAME

- Under Source Code Management -> Pick Git and add the project repository url (add credentials if your repo is private)

- Library Path leave it as default (if your library source live in custom path -> point to your custom path)

- Click Save

## Add the Jenkinsfile

In the `Jenkinsfile` repo, under `Jenkinsfile/YOUR_PROJECT_NAME/`, create `YOUR_SERVICE_NAME.jenkinsfile`:

```groovy
@Library(['YOUR_PIPELINE_LOGIC_LIBRARY_NAME@master', 'YOUR_ENV_CONFIG_LIBRARY_NAME@v1.0.0', 'YOUR_NOTIFY_LIBRARY_NAME@master']) _

def config = [
    PROJECT_NAME: 'YOUR_PROJECT_NAME',
    SERVICE_NAME: 'YOUR_SERVICE_NAME',
    PROJECT_URL: 'YOUR_GIT_REPO_URL',   
    stages: [
        [name: 'Check',  type: 'check',  enabled: 'true' ],
        [name: 'Deploy', type: 'deploy', enabled: 'false']   
    ]
]

corePipeline(config)
```

### Important :

- PROJECT_NAME and SERVICE_NAME must be exact same as project and service in environment_file Repo

- For the imported libraries can be use with tag or branch

## Add Environment Values

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

### Note : ENV values depend on how much of your env 

## Create the Multibranch Job

### Before Create Multibranch Job you need to install the follow plugin

1. Remote Jenkinsfile Provider 

2. Multibranch Scan Webhook Trigger

### After create multibranch job in jenkins, In multibranch job form enter the following :

1. New Item → Multibranch Pipeline

2. Branch Sources :

- Add source -> Git 

- Input your Project Repository (Add Credentials if your repo is private)

3. Behaviors : 

- Add -> Filter by name (with wildcards) input your brach that you want to deploy

5. Build Configuration : 

- Select by Remote Jenkinsfile Provider Plugin

- Under Script Path -> Input your path to Jenkins file

- Input your Jenkins file Repository URL (Add Credentials if private)

6. Under Scan Multibranch Pipeline Triggers : 

- Select Scan by webhook -> Input Your Trigger token

- Generate random hexadecimal string For Token 

```
openssl rand -hex <bytes> #bytes can be any number you want
```

7. Save : Scan the repo, confirm your branches are picked up

## Verify

- Trigger a build on the `dev` branch with only `Check` enabled first.
- Confirm in the build log:
  - `✅ Config loaded successfully!` (from `configEnvLoader`)
  - `✅ Config validation passed` (from `configValidate`)
  - `Executing stage: check` → `✅ Stage 'check' completed successfully`
- Confirm you got a Telegram notification (success/failure/unstable) from `telegramNotify`.
- Once `Check` is solid, flip `Deploy` to `enabled: 'true'` and re-test.
