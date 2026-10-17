---
name: pull
description: Bring tests that already exist in tester-e into this repository as .tester-e/ files, so Claude can maintain them and /tester-e:sync keeps both sides in step.
disable-model-invocation: true
argument-hint: "[plan or suite name/id]"
---

Pull existing tester-e tests into this repository. Follow the `tester-e-tests` skill's rules.

1. Read `.tester-e/config.yaml`; if it is missing, run the `/tester-e:init` steps first.
2. Choose what to pull: `$ARGUMENTS` names a plan or a suite; otherwise list them
   (`list_test_plans`, `list_test_suites`) and ask. For a plan, its suites come from
   `list_test_suites` with `test_plan_id`.
3. For each suite, `list_test_cases` with `test_suite_id`, then `get_test_case` for every
   automated case (WEB, API, PERFORMANCE, MOBILE). Skip MANUAL cases and list them in the
   report — they have no runnable steps.
4. Write one file per case: `.tester-e/tests/<suite dir>/<name>.yaml`, where the directory
   is the suite name and the file name the case name, both lower-case-hyphenated. Make
   keys unique. Keep only the authorable fields: `name`, `description`, `priority`,
   `tags`, `automation_type`, `execution_mode` (WEB), `web_config`/`web_steps`,
   `api_config`/`api_steps`, `performance_config`, `mobile_steps`. Drop ids, owners,
   timestamps and `extra_metadata`. `get_test_case` answers with camelCase top-level keys
   (`webSteps`, `apiConfig`, `performanceConfig`, `automationType`…) — write them in the
   file in snake_case (`web_steps`, `api_config`, …); keep the contents of each step and
   config exactly as they are. Never copy a secret value into a file — replace it
   with a `{{VARIABLE}}` and tell the user.
5. Run `validate_test_cases` on what you wrote. Do not silently "fix" steps: list every
   problem with its file (steps the runner cannot perform, missing targets…) and ask
   before changing a test's behaviour.
6. Call `adopt_catalog` with `repo` from the config and the mapping: each case
   (`test_case_id` → key), each suite (`test_suite_id` → directory name) and the plan
   (`test_plan_id` → a plan key). Report anything it refuses — e.g. a case already
   attached to another repository — and leave those files out.
7. Add the suites and the plan to `config.yaml` (`suites:` names, `plans:` with their
   suites).
8. Tell the user: the repository is now the source of truth for these tests. The next
   `/tester-e:sync` rewrites each pulled test once from its file (free) and changes
   nothing after that; an edit made later in the tester-e UI is overwritten by the next
   sync unless it is pulled again first.
