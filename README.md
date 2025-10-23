# Reusable Static Web App Deployment Workflow

A lightweight reusable GitHub Actions workflow for deploying an Azure Static Web App (Basic plan) to TEST or PROD, with a two-step safeguard for Production.

---

## Core Purpose
- Build, test, and lint a Node.js static web app.
- Deploy automatically from test branches (`dev`, `devfeatures`, `development`) using the TEST token.
- Deploy to Production from `master` only when explicitly confirmed (`deploy-prod=true`) and after environment approval.
- Reduce duplication by centralizing deployment logic.

---

## Branch & Deployment Matrix

| Branch                   | Needs `deploy-prod=true` | Environment       | Token Secret                    | Approval Required |
|--------------------------|--------------------------|-------------------|---------------------------------|------------------|
| dev / devfeatures / development | No                       | auto-approve       | STATIC_WEB_APP_TOKEN_TEST        | No (unless reviewers added) |
| master (deploy-prod = false)    | Yes (not set)            | (validate only)    | -                               | No               |
| master (deploy-prod = true)     | Yes (set)                | Production         | STATIC_WEB_APP_TOKEN_PROD        | Yes (reviewers)  |

---

## Three Jobs

1. **validate (Test & Build)**
   - Runs on allowed branches.
   - Steps: checkout → setup Node → install → test → lint → build.
   - Working directory set to `app-location`.

2. **approval (Approval Gate)**
   - Runs for test branches (auto-approve) OR for `master` when a production deploy is requested.
   - Uses GitHub Environments:
     - `auto-approve` (no reviewers, passes immediately).
     - `Production` (requires reviewers for manual approval).

3. **deploy (Deploy)**
   - Runs only if:
     - Test branch, or
     - `master` with `deploy-prod=true`.
   - Chooses token:
     - PROD: `STATIC_WEB_APP_TOKEN_PROD`
     - TEST: `STATIC_WEB_APP_TOKEN_TEST`
   - Action: `Azure/static-web-apps-deploy@v1` with `action: upload`.

---

## Inputs (workflow_call)

| Input            | Default | Description |
|------------------|---------|-------------|
| node-version     | 20.x    | Node.js runtime for build/test. |
| app-location     | (req)   | Path containing `package.json`. |
| output-location  | (req)   | Build output directory (e.g. `build`, `dist`). |
| deploy-prod      | false   | Must be `true` to deploy `master` to Production. |

## Secrets

| Secret                      | Usage |
|-----------------------------|-------|
| STATIC_WEB_APP_TOKEN_TEST   | Deploy from test branches. |
| STATIC_WEB_APP_TOKEN_PROD   | Deploy to Production when confirmed. |

Acquire tokens from Azure Portal → Static Web App → Deployment tokens.

---


## Safety Controls

- `deploy-prod` flag prevents accidental master deploys.
- Required reviewers enforce manual human approval.
- Separate tokens isolate TEST vs PROD subscriptions.
- Master pushes without confirmation only perform validation.

---

## Minimal Calling Workflow Example

```yaml
on:
  push:
    branches: [master, dev, devfeatures, development]
  workflow_dispatch:
    inputs:
      deploy-prod:
        description: "Set to true to deploy master to Production"
        type: choice
        options: ["false","true"]
        default: "false"

jobs:
  deploy-staticwebapp:
    uses: ./.github/workflows/Reusable-workflow.yml
    with:
      node-version: 20.x
      app-location: app1/frontend
      output-location: build
      deploy-prod: ${{ github.event_name == 'workflow_dispatch' && inputs.deploy-prod || 'false' }}
    secrets:
      STATIC_WEB_APP_TOKEN_TEST: ${{ secrets.STATIC_WEB_APP_TOKEN_TEST }}
      STATIC_WEB_APP_TOKEN_PROD: ${{ secrets.STATIC_WEB_APP_TOKEN_PROD }}
```

---

## Key Points

- Simple, clear separation: build → approval → deploy.
- Explicit PROD confirmation + environment reviewers for safety.
- Easily extend (artifacts, notifications, matrix) without complicating base logic.
- Encourages consistent CI/CD across front-end repos.

---
