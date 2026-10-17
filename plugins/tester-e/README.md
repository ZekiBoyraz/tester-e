# tester-e plugin for Claude Code

Claude writes WEB, API, performance and mobile tests for your app, keeps them in your repository
under `.tester-e/`, registers them in a tester-e project as test cases → suites →
plans, and runs them as a regression.

## Install

```text
/plugin marketplace add ZekiBoyraz/tester-e
/plugin install tester-e@tester-e
```

Then run `/mcp`, choose **plugin:tester-e:tester-e** and **Authenticate**. Sign in
with your tester-e account.

If the server does not connect, add it by hand once:

```bash
claude mcp add --transport http tester-e https://app.tester-e.com/mcp \
  --client-id tester-e-mcp --callback-port 33418
```

## Use

| Command | What it does | Credits |
|---|---|---|
| `/tester-e:init` | Pick the project, write `.tester-e/config.yaml`, create the environment | none |
| `/tester-e:pull [plan or suite]` | Bring tests that already exist in tester-e into the repo as files | none |
| *"write tests for the checkout flow"* | Claude writes `.tester-e/tests/**.yaml` (WEB, API, performance, mobile) | none |
| `/tester-e:sync [--prune]` | Validate, then create/update cases, suites and plans | creating a case costs its steps; unchanged cases cost nothing |
| `/tester-e:execute [plan \| suite \| test key …] [env=…]` | Run with self-healing **off**, report failures | executed steps |
| `/tester-e:ci [github \| gitlab \| azure]` | Write a GitHub Actions, GitLab CI or Azure Pipelines job | (CI runs: executed steps) |

Tests that disappear from the repo are tagged `tester-e:stale` and set to Draft
(`--prune`). They are never deleted; run history and linked bugs stay.

Passwords and tokens never go into `.tester-e/`. They are secret variables of the
tester-e environment (Test Environments in the app).
