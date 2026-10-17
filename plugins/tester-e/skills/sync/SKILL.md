---
name: sync
description: Register this repository's .tester-e tests in tester-e as test cases, suites and plans — idempotently.
disable-model-invocation: true
argument-hint: "[--prune]"
---

Sync `.tester-e/` into tester-e. Follow the `tester-e-tests` skill's rules.

1. Read `.tester-e/config.yaml` (`project_id`, `repo`). If it is missing,
   stop and suggest `/tester-e:init`.
2. Read every `.tester-e/tests/<suite>/<name>.yaml`. Each becomes a case with
   `key: "<suite>/<name>"` plus the file's fields.
3. Call `validate_test_cases` with all cases. If anything is invalid, show the
   errors by file, fix them if the fix is obvious (ask otherwise), and validate
   again. Do not continue while errors remain. Mention warnings (e.g. a test with
   no assertion).
4. Call `upsert_test_cases` with `project_id`, `repo` and the cases, at most 200
   per call. Use `prune: "mark"` **only** if `$ARGUMENTS` contains `--prune`,
   and only on the last call, which must carry the complete list. Otherwise
   report the cases that are no longer in the repo as "would be marked stale"
   and ask.
5. Call `ensure_catalog`:
   - one suite per test directory: `key` = directory name, `name` from
     `config.suites.<dir>.name` (default: the directory name, title-cased),
     `case_keys` = that directory's keys;
   - one plan per `config.plans` entry: `key`, `name`, `suite_keys` = its `suites`.
6. Report briefly: created / updated / unchanged / stale counts with display ids
   (TC-…, TS-…, TP-…), the credits creation used (created cases × their steps),
   and anything that failed. Never call a test "passing". A sync does not run anything.
