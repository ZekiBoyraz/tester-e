---
name: execute
description: Run tester-e tests from this repository — a plan, a suite, or individual test cases — and report what failed.
disable-model-invocation: true
argument-hint: "[plan | suite | test key …] [env=<environment>]"
---

Run tests on tester-e. Follow the `tester-e-tests` skill's rules.

1. Read `.tester-e/config.yaml`. Work out the target from `$ARGUMENTS`:
   - a plan key from `config.plans` → that plan;
   - a test directory name → that suite;
   - one or more test keys (`<suite>/<name>`, i.e. files under `.tester-e/tests/`) → those cases;
   - nothing → the first plan in `config.plans`; if there is none, say so and offer to add one.
   Environment: `env=<name>` if given, else `config.environment`.
   If a test file changed since the last `/tester-e:sync`, say that the run uses the
   synced version and offer to sync first.
2. **Plan or suite:** get its id (`ensure_catalog` with the config's suites/plans
   returns ids, or `list_test_plans` / `list_test_suites`), then call
   `run_regression` with `project_id`, `test_plan_id` or `test_suite_id`,
   `environment` and `wait_seconds: 480` — plus `mobile` (from `config.mobile`) when
   the target has MOBILE tests. Without it mobile tests cannot run; say so and point to
   the `mobile:` block in the `tester-e-tests` skill instead of guessing device values.
   **Test cases:** find them with `find_test_cases` (`metadata_key: external_key`,
   `metadata_value: "<repo>::<key>"`). Group them by `automation_type`; for each group
   call `create_test_execution` with `test_case_ids` (MOBILE cases: run their suite
   through `run_regression` instead — a single-case run cannot carry the device),
   then `start_test_execution`
   with the environment's id (`list_environments`) and `max_healing_attempts: 0`,
   then poll `get_execution_results` (`failed_only`) every ~10 s until it settles.
3. Self-healing stays **off**: a broken selector should fail, not be repaired. If
   the user asks for healing, explain that and then pass `healing: true` (or a
   non-zero `max_healing_attempts`). If the run looks large (more than ~200
   steps), pass `max_steps` and confirm the cost first.
4. If `run_regression` returns `STILL_RUNNING`, poll `get_execution_results`
   (`failed_only`) for each execution id every ~30 s until done.
5. Report: passed/failed totals per execution, then for each failed case its
   key/name, the first failing step (number, description, error) and what that
   most likely means. Read the relevant source to tell an app regression from a
   test that needs updating. Offer to fix the test or the code, but **do not**
   change a test to make it pass without saying that is what you are doing.
