---
name: tester-e-tests
user-invocable: false
description: How to write WEB, API, PERFORMANCE and MOBILE tests for tester-e and keep them in this repository under .tester-e/. Use when the user asks to write, update, organise, sync or run tester-e tests, end-to-end tests, regression tests or API tests on tester-e.
---

# Writing tests for tester-e

tester-e runs tests from **structured steps**, not from test code. You write the
steps as YAML files in this repository; `/tester-e:sync` registers them in a
tester-e project, and `/tester-e:execute` runs them. tester-e never executes
code you write, so a Playwright or pytest file is not something it can run.
Translate the scenario into steps.

## Ground rules

1. **Read the schema before writing.** Call the MCP tool `describe_step_schema`
   with `automation_type: WEB`, `API`, `PERFORMANCE` or `MOBILE`. Use only the actions, assertion types
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
environment: staging         # default for /tester-e:execute
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

A PERFORMANCE test has no steps — its `performance_config` is the test. Two kinds:
**Lighthouse** page performance (`test_type: page_performance`) and a **k6 load test**
(`test_type: smoke | load | stress | spike | soak | breakpoint` + `traffic_test`).

```yaml
# .tester-e/tests/perf/home-page.yaml
name: Home page stays fast
automation_type: PERFORMANCE
performance_config:
  test_type: page_performance
  page_performance:
    urls: [{ url: "{{BASE_URL}}/", name: Home }]   # objects, not strings
    run_count: 1
    network_throttling: fast_3g
    thresholds:
      - { metric: lcp, max_value: 2500 }               # ms, lower is better
      - { metric: performance_score, max_value: 80 }   # 0–100, a MINIMUM
```

```yaml
# .tester-e/tests/perf/products-api-load.yaml
name: Products API holds 20 users for 2 minutes
automation_type: PERFORMANCE
performance_config:
  test_type: load
  traffic_test:
    target_url: "{{BASE_URL}}/api/products"
    method: GET
    stages: [{ duration: 30s, target: 20 }, { duration: 2m, target: 20 }, { duration: 30s, target: 0 }]
    thresholds:
      - { metric: http_req_duration, condition: "p(95)<500" }
      - { metric: http_req_failed, condition: "rate<0.01" }
```

Performance rules:
- **Always give thresholds.** Without them a Lighthouse run never fails and a load
  test passes whatever the server answers.
- FID and INP cannot be measured in a lab run — use `tbt`/`lcp` instead.
- Targets must be public URLs. **Never load-test production** or anything the user
  does not own without asking; keep stages short (the total must stay under 28 min).
- A performance run costs **10 credits per URL** (Lighthouse) or **10 per load test**
  — say so before running a suite with many of them.

A MOBILE test (`.tester-e/tests/mobile/login.yaml`):

```yaml
name: A user can sign in on Android
automation_type: MOBILE
mobile_steps:
  - { order: 1, action: launch_app, description: "Open the app" }
  - { order: 2, action: tap, target: login_button, target_strategy: accessibility_id, description: "Open sign in" }
  - { order: 3, action: type_text, target: email, target_strategy: resource_id, value: "{{TEST_USER}}", description: "Type the email" }
  - { order: 4, action: tap, target: Sign in, target_strategy: text, description: "Sign in" }
  - { order: 5, action: assert_visible, target: Welcome, target_strategy: text, description: "Home screen shows" }
```

Mobile rules:
- Prefer `accessibility_id` (Android content-desc / iOS accessibilityIdentifier), then
  `resource_id`, then `text` — those three work on every device. Read them from the
  app's source; xpath and the other strategies work on a device farm only.
- Coordinates and reference images are recorded, not written — never use
  `target_strategy: coordinates`.
- **Running needs a device.** The device and the app come from the run, not the test:
  a device farm set up in tester-e → Settings → Integrations → Mobile Devices, and
  either the app uploaded in tester-e (its `appPath`) or the package / bundle id of an
  app already on the device. Keep them under `mobile:` in config.yaml:

```yaml
mobile:
  platform: android
  deviceSource: device_farm
  deviceFarmProvider: browserstack
  deviceName: Google Pixel 8
  osVersion: "14.0"
  appPackage: com.example.shop      # or appPath of an uploaded build, or bundleId on iOS
```

## Keys

The file path is the test's identity: `auth/login.yaml` is key `auth/login`.
Renaming or moving a file creates a new test and leaves the old one stale. Keep
paths stable, and tell the user when a change will do that.
