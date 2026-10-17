---
name: regress
description: Run a tester-e plan or suite from this repository as a regression and report what failed.
disable-model-invocation: true
argument-hint: "[plan or suite key] [environment]"
---

Run a regression on tester-e. Follow the `tester-e-tests` skill's rules.

1. Read `.tester-e/config.yaml`. Target: `$0` if given (a plan key, otherwise a
   suite/directory key), else the first plan. Environment: `$1`, else
   `config.environment`.
2. Find the target's id: run `ensure_catalog` with the config's suites/plans if
   needed (it returns ids), or `list_test_plans` / `list_test_suites`.
3. Call `run_regression` with `project_id`, `test_plan_id` or `test_suite_id`,
   `environment` and `wait_seconds: 480`. Keep `healing` off: a regression should
   fail on a broken selector, not repair it. If the user asks for healing,
   explain that and then pass `healing: true`. If the run looks large (more than
   ~200 steps), pass `max_steps` and confirm the cost first.
4. If the status is `STILL_RUNNING`, poll `get_execution_results` (failed_only)
   for each execution id every ~30 s until done.
5. Report: passed/failed totals, then for each failed case its key/name, the
   first failing step (number, description, error) and what that most likely
   means. Read the relevant source to tell an app regression from a test that
   needs updating. Offer to fix the test or the code, but **do not** change the
   test to make it pass without saying that is what you are doing.
