---
name: ci
description: Add a GitHub Actions workflow that runs this repository's tester-e regression plan on pull requests and pushes.
disable-model-invocation: true
argument-hint: "[plan key]"
---

Write `.github/workflows/tester-e-regression.yml` for this repository.

1. Read `.tester-e/config.yaml`. Plan: `$0` or the first plan. Find its id with
   `list_test_plans` (use the uuid, not TP-…). Environment: `config.environment`.
   Work out which automation types the plan contains (WEB, API, PERFORMANCE) from the test
   files. Each type is triggered as its own run.
2. Write the workflow below with the plan id, environment name and the matrix
   filled in. Replace `main` under `push: branches:` with this repository's default
   branch (`git symbolic-ref --short refs/remotes/origin/HEAD`, else the current
   branch — a push to any other name would never trigger it). Ask before
   overwriting an existing file.
3. Tell the user the one-time setup: create an API key in tester-e → Settings →
   Integrations → CI/CD API Keys (the run is attributed to the key's creator and
   billed like their runs), add it as the repository secret `TESTER_E_API_KEY`,
   and note that CI/CD needs a PRO or ENTERPRISE plan.
4. Say that a CI run uses the platform's self-healing defaults. The webhook takes no
   healing switch, unlike `/tester-e:execute`, which runs with healing off.
5. Be honest about what a PR run proves. The tests drive the environment's
   `BASE_URL`, so unless that is a per-PR preview deployment, a green run says the
   deployed app works, not that this PR does.

```yaml
name: tester-e regression
on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  regression:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    strategy:
      fail-fast: false
      matrix:
        type: [WEB, API]          # only the types the plan contains (WEB, API, PERFORMANCE)
    env:
      TESTER_E_URL: https://app.tester-e.com
      PLAN_ID: <plan uuid>
      ENVIRONMENT: <environment name>
    steps:
      - name: Trigger and wait
        run: |
          set -euo pipefail
          BODY=$(curl -sS -f -X POST "$TESTER_E_URL/api/test-management/api/v1/webhooks/trigger" \
            -H "Authorization: Bearer ${{ secrets.TESTER_E_API_KEY }}" -H "Content-Type: application/json" \
            -d "{\"test_plan_id\":\"$PLAN_ID\",\"execution_type\":\"${{ matrix.type }}\",\"environment\":\"$ENVIRONMENT\",\"metadata\":{\"commit_sha\":\"${{ github.sha }}\",\"branch\":\"${{ github.ref_name }}\",\"run_id\":\"${{ github.run_id }}\"}}")
          ID=$(echo "$BODY" | jq -r '.batch_id')
          for i in $(seq 1 90); do
            R=$(curl -sS "$TESTER_E_URL/api/test-management/api/v1/webhooks/status/$ID" \
                -H "Authorization: Bearer ${{ secrets.TESTER_E_API_KEY }}") || true
            S=$(echo "$R" | jq -r '.status // "UNKNOWN"')
            echo "[$i] $S — passed $(echo "$R" | jq -r '.passed // 0') / failed $(echo "$R" | jq -r '.failed // 0')"
            case "$S" in COMPLETED) exit 0;; FAILED|CANCELLED) exit 1;; esac
            sleep 20
          done
          echo "::error::timed out"; exit 1
```
