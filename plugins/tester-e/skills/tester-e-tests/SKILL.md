---
name: tester-e-tests
description: How to write WEB and API tests for tester-e and keep them in this repository under .tester-e/. Use when the user asks to write, update, organise, sync or run tester-e tests, end-to-end tests, regression tests or API tests on tester-e.
---

# Writing tests for tester-e

tester-e runs tests from **structured steps**, not from test code. You write the
steps as YAML files in this repository; `/tester-e:sync` registers them in a
tester-e project, and `/tester-e:regress` runs them. tester-e never executes
code you write, so a Playwright or pytest file is not something it can run.
Translate the scenario into steps.

## Ground rules

1. **Read the schema before writing.** Call the MCP tool `describe_step_schema`
   with `automation_type: WEB` or `API`. Use only the actions, assertion types
   and operators it lists. Its lists are what the runner accepts today.
2. **Validate before saving.** `validate_test_cases` is free and writes nothing.
   `upsert_test_cases` refuses anything it rejects. A step the runner does not
   understand would otherwise do nothing and pass.
3. **Every test checks something.** End a WEB test with `expect_visible`,
   `expect_text`, `expect_url` or `assert`. Give every API step at least a
   `status_code` assertion.
4. **No secrets in files.** Use `{{VARIABLE}}` placeholders. Values live in the
   tester-e environment; secret ones are set by the user in tester-e →
   Test Environments, or passed to `ensure_environment` as `secrets` only when
   the user gives them to you in this session. Never write a password, token or
   API key into `.tester-e/`.
5. **Prefer stable locators**, in this order: `testid=<id>` (a `data-testid`)
   → `role=<role>[name="<name>"]` → `text=<exact text>` →
   `placeholder=<text>` → CSS. Read the app's source for the real `data-testid`,
   roles and labels. If an element you need has no stable hook, say so and
   suggest adding a `data-testid` rather than writing a brittle CSS path.
6. **Spending.** Creating a test case costs its step count in credits, exactly
   like in the tester-e UI. Re-syncing unchanged tests costs nothing. A run costs
   its executed steps. Say roughly what an action will cost before a large sync
   or run.

## Files

```
.tester-e/
  config.yaml
  tests/
    <suite>/            # one directory = one suite
      <name>.yaml       # one file = one test case; key = "<suite>/<name>"
```

`config.yaml`:

```yaml
project_id: 3f1c…            # tester-e project (list_projects)
repo: acme/shop-frontend     # stable name; tests, suites and plans are keyed by it
environment: staging         # default for /tester-e:regress
environments:
  staging:
    scope: all               # all | web | api
    variables:
      BASE_URL: https://staging.acme.com
    secret_names: [TEST_USER_PASSWORD]
suites:                      # optional: names/descriptions for test directories
  checkout: { name: "Checkout" }
plans:
  regression:
    name: "Regression"
    suites: [auth, checkout] # directory names
```

A WEB test (`.tester-e/tests/auth/login.yaml`):

```yaml
name: A registered user can sign in
priority: HIGH                 # LOW | MEDIUM | HIGH | CRITICAL
automation_type: WEB
tags: [auth, smoke]
web_config: { baseUrl: "{{BASE_URL}}" }
web_steps:
  - { order: 1, action: navigate, value: "{{BASE_URL}}/login", description: "Open the login page" }
  - { order: 2, action: fill, target: "testid=email", value: "{{TEST_USER}}", description: "Type the email" }
  - { order: 3, action: fill, target: "testid=password", value: "{{TEST_USER_PASSWORD}}", description: "Type the password" }
  - { order: 4, action: click, target: 'role=button[name="Sign in"]', description: "Sign in" }
  - { order: 5, action: expect_url, assertionValue: "/dashboard", description: "Lands on the dashboard" }
```

An API test (`.tester-e/tests/api/get-product.yaml`):

```yaml
name: GET /products/{id} returns the product
automation_type: API
api_config: { base_url: "{{BASE_URL}}" }
api_steps:
  - order: 1
    name: Get product 1
    method: GET
    url: "{{BASE_URL}}/api/products/1"
    assertions:
      - { type: status_code, operator: equals, expected: 200 }
      - { type: json_path, path: "$.id", operator: equals, expected: 1 }
```

`description` on a WEB step matters: it is what self-healing uses to find an
element that moved. Write what the step does, in words.

## Keys

The file path is the test's identity: `auth/login.yaml` is key `auth/login`.
Renaming or moving a file creates a new test and leaves the old one stale. Keep
paths stable, and tell the user when a change will do that.
