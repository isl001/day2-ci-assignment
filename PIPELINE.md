# Pipeline Design

## 1. Trigger events

| Event | Why |
|-------|-----|
| `push` to `main` (ignoring `**.md` and `docs/**`) | Every change that lands on `main` gets verified. Docs-only changes can't break the code, so we skip them to save runner minutes. |
| `pull_request` targeting `main` | Catch problems **before** merge and give reviewers a green/red signal. |
| `workflow_dispatch` (inputs `python-version`, `create-package`) | Re-run on demand, e.g. to try another Python version or test without packaging. |
| `schedule` (Mondays 07:00 UTC) | Detect "bit rot": new versions of dependencies or runner images can break the build even when our code didn't change. |

## 2. Jobs and graph

```
            ┌──────────► verify
test ──► package
  │         ├──────────► pr-comment   (pull requests only)
  │         │
  └────┬────┘
       ▼
     report   (always runs)
```

- `test` runs first, alone.
- `package` waits for `test`.
- `verify` and `pr-comment` both wait for `package` and run **in parallel**.
- `report` waits for `test` and `package` and runs whatever their result.

## 3. Artifact

`dist/temperature-0.1.<run_number>.tar.gz` (contains `temperature.py` and `README.md`), uploaded as artifact `package`.
A JUnit test report (`test-report`) is also uploaded, but that is a by-product, not the deliverable.

## 4. CI, Continuous Delivery, or Continuous Deployment?

**CI**: it builds, tests, and packages every change, but it never releases or deploys the package to any environment.

## 5. `package` when tests fail

`package` must be **skipped**. A package built from code that fails its tests is broken and must never be produced: anyone who downloaded it would get a known-bad build. `needs: test` gives us this for free.
