---
name: init
description: Set up tester-e for this repository — pick the project, create .tester-e/config.yaml and the test environment.
disable-model-invocation: true
argument-hint: "[project name]"
---

Set this repository up for tester-e. Follow the `tester-e-tests` skill's rules.

1. Call `list_projects`. If `$ARGUMENTS` names a project, use it. Otherwise show
   the list and ask which one. Do not create a project.
2. Choose `repo`: the `origin` remote's `owner/name` (run `git remote get-url
   origin`), or the directory name if there is no remote. Tell the user. It must
   stay stable.
3. Find out where the app under test runs (staging/preview URL). Look in the
   repo (README, `.env.example`, deploy config) and **ask** the user to confirm.
   Never assume production.
4. Call `list_environments`. Reuse a matching one, or call `ensure_environment`
   with `scope: all`, `BASE_URL` and any non-secret variables. Put every
   credential the tests will need in `secret_names`, then tell the user to set
   those values in tester-e → Test Environments (the tool reports them as
   `missing_secrets`).
5. Write `.tester-e/config.yaml` (format in `tester-e-tests`) and an empty
   `.tester-e/tests/`. Show the file. Nothing in it may be secret.
6. Close with what to do next, in two lines:
   - tests already in tester-e → `/tester-e:pull` brings them into the repo;
   - new tests → just ask, e.g. "write tests for the sign-in and checkout flows", then
     `/tester-e:sync` and `/tester-e:execute`.
   Offer a first set of tests from the app's main flows; write them only when asked.
