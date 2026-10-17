---
name: ci
description: Add a CI job — GitHub Actions, GitLab CI or Azure Pipelines — that runs this repository's tester-e plan on pull/merge requests and pushes.
disable-model-invocation: true
argument-hint: "[github | gitlab | azure] [plan key]"
---

Add a CI job that runs a tester-e plan and fails when its tests fail.

1. **Provider:** `github`, `gitlab` or `azure` from `$ARGUMENTS`; otherwise detect it —
   `git remote get-url origin` (github.com → github; gitlab → gitlab; dev.azure.com or
   visualstudio.com → azure) or an existing `.github/workflows/`, `.gitlab-ci.yml`,
   `azure-pipelines.yml`. If it is still unclear, ask.
2. **What to run:** read `.tester-e/config.yaml`. Plan: a plan key in `$ARGUMENTS` or the
   first plan; find its uuid with `list_test_plans` (not TP-…). Environment:
   `config.environment`. Work out the automation types the plan contains (WEB, API,
   PERFORMANCE) from the test files; each type is triggered as its own run. MOBILE stays
   out: the trigger cannot carry a device, so a mobile run in CI needs a tester-e pipeline
   with its device settings, triggered with `pipeline_id` — say so if the plan has mobile tests.
3. **Default branch:** `git symbolic-ref --short refs/remotes/origin/HEAD`, else the
   current branch. Put it where the template says `<default branch>`.
4. Write the file for the provider (templates below), filling `<plan uuid>`,
   `<environment>`, `<default branch>` and the type list, and replacing
   `<the step above>` with the step, indented to its place. If the file already exists,
   show the change and ask before writing (GitLab: add the job to the existing file).
5. Tell the user the one-time setup:
   - tester-e → Settings → Integrations → CI/CD API Keys → create a key. The run is
     attributed to the key's creator and billed like their runs. CI/CD needs a PRO or
     ENTERPRISE plan.
   - Store it as `TESTER_E_API_KEY`:
     - **GitHub:** Settings → Secrets and variables → Actions → New repository secret.
     - **GitLab:** Settings → CI/CD → Variables → Add variable, **Masked**. If you also mark
       it Protected, merge-request pipelines from unprotected branches will not see it.
     - **Azure:** Pipelines → the pipeline → Edit → Variables → New variable, **Keep this
       value secret**. Secret variables reach a script only through the `env:` mapping
       the template already has.
   - **Azure Repos:** pull-request runs come from a branch policy (Repos → Branches →
     … → Branch policies → Build validation), not from the YAML. GitHub-hosted repos
     built in Azure Pipelines use the `pr:` block.
6. Say that a CI run uses the platform's self-healing defaults — the trigger takes no
   healing switch, unlike `/tester-e:execute`, which runs with healing off.
7. Be honest about what a PR run proves: the tests drive the environment's `BASE_URL`, so
   unless that is a per-PR preview deployment, a green run says the deployed app works,
   not that this PR does.

## The step every template runs

```sh
set -eu
BODY=$(curl -sS -f -X POST "$TESTER_E_URL/api/test-management/api/v1/webhooks/trigger" \
  -H "Authorization: Bearer $TESTER_E_API_KEY" -H "Content-Type: application/json" \
  -d "{\"test_plan_id\":\"$PLAN_ID\",\"execution_type\":\"$TYPE\",\"environment\":\"$ENVIRONMENT\",\"metadata\":{\"commit_sha\":\"$COMMIT\",\"branch\":\"$BRANCH\",\"run_id\":\"$RUN_ID\"}}")
ID=$(echo "$BODY" | jq -r '.batch_id')
i=0
while [ $i -lt 90 ]; do
  i=$((i + 1))
  R=$(curl -sS "$TESTER_E_URL/api/test-management/api/v1/webhooks/status/$ID" \
      -H "Authorization: Bearer $TESTER_E_API_KEY") || true
  S=$(echo "$R" | jq -r '.status // "UNKNOWN"')
  echo "[$i] $TYPE $S — passed $(echo "$R" | jq -r '.passed // 0') / failed $(echo "$R" | jq -r '.failed // 0')"
  case "$S" in COMPLETED) exit 0;; FAILED|CANCELLED) exit 1;; esac
  sleep 20
done
echo "timed out"; exit 1
```

## GitHub — `.github/workflows/tester-e-regression.yml`

```yaml
name: tester-e regression
on:
  pull_request:
  push:
    branches: [<default branch>]
  workflow_dispatch:
jobs:
  regression:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    strategy:
      fail-fast: false
      matrix:
        type: [WEB, API]          # the plan's types (WEB, API, PERFORMANCE)
    env:
      TESTER_E_URL: https://app.tester-e.com
      TESTER_E_API_KEY: ${{ secrets.TESTER_E_API_KEY }}
      PLAN_ID: <plan uuid>
      ENVIRONMENT: <environment>
      TYPE: ${{ matrix.type }}
      COMMIT: ${{ github.sha }}
      BRANCH: ${{ github.ref_name }}
      RUN_ID: ${{ github.run_id }}
    steps:
      - name: Trigger and wait
        run: |
          <the step above>
```

## GitLab — job in `.gitlab-ci.yml`

```yaml
tester-e-regression:
  image: alpine:3.20
  timeout: 45m
  parallel:
    matrix:
      - TYPE: [WEB, API]          # the plan's types (WEB, API, PERFORMANCE)
  variables:
    TESTER_E_URL: https://app.tester-e.com
    PLAN_ID: <plan uuid>
    ENVIRONMENT: <environment>
    COMMIT: $CI_COMMIT_SHA
    BRANCH: $CI_COMMIT_REF_NAME
    RUN_ID: $CI_PIPELINE_ID
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "<default branch>"
    - if: $CI_PIPELINE_SOURCE == "web"
  before_script:
    - apk add --no-cache curl jq
  script:
    - |
      <the step above>
```

## Azure DevOps — `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include: [<default branch>]
pr:                                # GitHub-hosted repos only; Azure Repos → branch policy
  branches:
    include: ['*']
pool:
  vmImage: ubuntu-latest
strategy:
  matrix:                          # the plan's types (WEB, API, PERFORMANCE)
    WEB: { TYPE: WEB }
    API: { TYPE: API }
variables:
  TESTER_E_URL: https://app.tester-e.com
  PLAN_ID: <plan uuid>
  ENVIRONMENT: <environment>
steps:
  - bash: |
      <the step above>
    displayName: tester-e regression ($(TYPE))
    timeoutInMinutes: 45
    env:
      TESTER_E_API_KEY: $(TESTER_E_API_KEY)   # secret variables are not visible without this
      COMMIT: $(Build.SourceVersion)
      BRANCH: $(Build.SourceBranchName)
      RUN_ID: $(Build.BuildId)
```
