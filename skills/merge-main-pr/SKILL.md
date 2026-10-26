---
name: merge-main-pr
description: Merge main into the current feature branch, resolve the conflicts, verify the CI gates locally, and write the PR body. The body explains what main brought in, how each conflict was resolved, and what the change affects in the AI flows (prompts, model config, pipeline and MCQ fingerprints, worker, GPU budget, limitations). Use when the user asks to pull or merge main, resolve conflicts, sync the branch with main, prepare or write a PR description, or says "pull main", "resolve conflict", "viết PR", "request merge vào main", even if they mention only one of these.
---

# Merge main into the branch and write the PR body

The reader of the PR body is a reviewer who did not watch the work. They need to know what
came from main, which conflicts were decided and why, and what the change does to the AI
features. Every claim must be backed by something you ran or read in this session.

## Guardrails

- **Merge, never rebase.** The branch is pushed and has review history.
- **No blanket resolution:** no `-X ours`, `-X theirs`, `git checkout --theirs .`.
- **Modules 08–13** (`m08`..`m13`): if they conflict, take main's side without inspecting
  (CLAUDE.md). Do not report on them.
- **Do not push, force-push, or edit the PR** unless the user asks. Commit locally, then
  offer.
- **Python:** use `projects/core-ai/.venv`. The root `.venv` lacks `vtc_core_ai`, and the
  preparation subprocess then fails with `No module named 'vtc_core_ai'`.
- A conflict you cannot decide from the code and the incoming PR's intent: stop and ask
  (`AskUserQuestion`) with both options spelled out. Do not guess.

## 1. Preflight

```bash
git status --short                      # must be clean; if not, ask whether to commit or stash
git rev-parse --abbrev-ref HEAD         # refuse main
git fetch origin main
PRE=$(git rev-parse HEAD)               # keep: every later diff compares against it
git rev-list --count HEAD..origin/main  # commits main has that the branch lacks
```

If the count is 0 there is nothing to merge: say so, skip steps 2–6, and still do 7–9.

## 2. Learn what main brought, before touching a conflict

```bash
git log --no-merges --format='%h %an %s' $PRE..origin/main
git diff --stat=200 $(git merge-base HEAD origin/main) origin/main
```

- Take PR numbers from the `(#N)` in subjects; `gh pr view N --json title,body,author` for
  each large one. Read any `docs/plans/*` and ADR the incoming work added: they hold the
  intent you need to resolve conflicts correctly.
- Note, per incoming PR: area, what it changed, new migrations, new `LIM-` ids, config keys,
  new routes. This becomes the *What came from main* table.

## 3. Merge and resolve

```bash
git merge origin/main        # conflicts expected; stop at the first surprise
git diff --name-only --diff-filter=U
```

For each conflicted file read both sides and the incoming PR's intent, then:

| File | Policy |
|---|---|
| `projects/core-ai/config/limitations/seed.yaml` | **Union.** Both sides append blocks at the end, and taking one side silently drops the other's `# 2026-…` notes (it happened twice: `dcda0e8`, `fe2bed9`). Keep every entry and note. If both claim the same `LIM-NNN`, main keeps its number; renumber the branch's, and leave a comment saying so. Never delete an entry; supersede in place with a bumped `entry_version`. |
| `docs/architect/openapi.json` | Never merge by hand. Take either side, then `cd projects/core-ai && make openapi`. |
| `migrations/versions/*` | Never edit a migration that is on main. Two heads (`alembic heads`): add a merge revision with no DDL (as `d4e1c8a0f12b`), or re-point the branch's `down_revision`. |
| `config/grading/pipeline.yaml`, `config/prompts/**`, other `config/**` | Union the keys. Both sides' values must stay; where one key differs, read why each side changed it. These are AI behaviour: record every one in the AI impact section. |
| `pyproject.toml` | Union dependencies. |
| `README.md`, `docs/**` | Keep both sides' facts; re-read the result for contradictions. |
| Source code | Combine intent, not lines. If both sides changed one function for different reasons, write the combination and add or adjust a test that covers both. |

Afterwards: `git diff --check`, `grep -rn '<<<<<<<\|>>>>>>>' --include=* .` outside `.venv`
and `node_modules` must be empty. `git add` the resolved files.

## 4. Look for damage that git did not flag

A clean merge can still be wrong.

```bash
git diff $PRE HEAD -- projects/core-ai/config/limitations/seed.yaml | grep '^-' | grep -v '^---'
```

- Removed lines in `seed.yaml` must be only the ones you meant to change. Restore any lost
  note from `$PRE`.
- `alembic heads` must print exactly one head.
- Duplicate imports or definitions introduced by the merge: `ruff check` finds them.
- A function or config field that main renamed and the branch still uses: `mypy` and the
  tests find them.
- `make openapi` run last, then `git diff --stat docs/architect/openapi.json`.

## 5. Verify like CI (`.github/workflows/ci.yml`)

From `projects/core-ai` with its venv active:

```bash
ruff check src test scripts && ruff format --check src test scripts
mypy
python scripts/check_structural_constraints.py
python -m vtc_core_ai.tools.export_openapi | cmp - ../../docs/architect/openapi.json
```

Tests, scoped first, then the full run:

```bash
set -a; . ./.env; set +a      # VTC_DB_DSN, for the worker tests
pytest -q -m "not compliance and not load" --cov     # gate: 80 % lines
```

- `.env` leaks into `Settings`, so `settings_test.py`, `app_test.py` and
  `vtc_source_reader_test.py` fail locally with it. Run those three without it (from a
  directory that has no `.env`, with `--rootdir` and `-c` pointing at the project).
- A failing `models_test` migration round-trip on a dev database with audit rows is
  environmental.
- Any other failure: decide whether it is yours by running it on `origin/main`
  (`git worktree add /tmp/main-baseline origin/main`). Do not call a failure
  "pre-existing" without that run. mypy errors that exist identically on main are
  pre-existing; say which.
- S8: a change under `m0[2-7]/` or `config/{prompts,retrieval,grounding,safety,models}/`
  must also touch `config/limitations/seed.yaml`.

## 6. Commit the merge

Use git's default subject, and put the decisions in the body: one line per conflicted file
saying what was kept and why. A reviewer reads this first.

## 7. Analyse the AI impact

The author owns every AI feature, so this section decides whether the PR is safe to ship.
Compute the files that matter from `git diff --name-only origin/main...HEAD` (what the PR
adds on top of main) and from step 2 (what was integrated). For each area below that
appears, state **what changed, the effect, and what the owner must do**.

| Path | Affects | Check and say |
|---|---|---|
| `config/prompts/**` | model behaviour; `prompt_version`; MCQ config fingerprint | which pass changed, whether outputs are cached by it |
| `config/grading/pipeline.yaml` | models, URLs, token caps, repair attempts, `generation_priority` | each changed key; cost and latency on the single shared card |
| `pipeline/**` (not `__init__.py`, `report.py`) and `utils/hashing.py` | `code_fingerprint` → `pipeline_identity` (`preparation/currency.py`) | **Stored preparations stop being current; teachers must prepare again** (PR #73 said so). Verify in `currency.py`; do not assume. Cached stages recompute. |
| `mcq/*.py` (not tests), `config/grading/mcq_*.yaml` | `mcq_code_fingerprint`, `mcq_config_fingerprint` (`mcq_api/identity.py`) | API and `grading-preparer` must deploy from the same commit, or every MCQ build fails `INTEGRITY`. Name the images to rebuild. |
| `worker/**`, `jobs.py` | `grading-preparer` jobs, claim order, failure codes | which job, whether the loop order changed |
| `mcq_api/**`, `intake/**`, other routers | published contract, `EXAM_NOT_READY` problems, required fields | which request shape breaks old clients |
| `models.py`, `migrations/**` | schema; downgrade data loss; heads | one head? what the downgrade drops |
| `config/limitations/seed.yaml` | what VTC is told | entries added or superseded, with versions |
| `m02`, `m03`, `m04`, `m06`, `m07` | ingestion, retrieval, chatbot, feedback, lesson planner | the path, and which flow a user sees change |
| `docker/**`, `config/gpu/**` | GPU budget | `--gpu-memory-utilization` fractions of `vllm`, embedder, reranker and OCR must sum below 1.0 (default 0.70+0.05+0.04+0.20) |
| `quota/**` | AI quota charging | which operations are metered |

Rules for this section:

- Say **measured** or **not measured**. A figure from a probe is not an evaluation
  reference set (P5, P8). If the PR changes a model, prompt or retrieval config with no
  passing evaluation run, leave that checklist box unticked and say why.
- Compare incoming and branch changes in the same area: when main and the branch both touched
  one pipeline or prompt, that is the interaction to explain.
- If an area is not touched, leave it out. No filler.

## 8. Write the PR body

Write in English (the repo's PRs are), to the scratchpad. Start from
`.github/pull_request_template.md` and keep its headings. Structure:

1. **What changed**: two to four sentences, then numbered subsections by feature.
2. **What came from main**: table of incoming PR number, area, what it brought, and
   whether it conflicted.
3. **Conflicts resolved**: per file, what each side did, what you kept, and why. Include
   silent fixes from step 4.
4. **AI impact**: the step 7 table, filled.
5. **Behaviour changes to know before merging**: migrations and heads, redeploy order,
   re-prepare requirements, breaking request shapes, GPU or quota effects.
6. **What is not measured yet**.
7. **Traceability**: plans and ADRs implemented, `LIM-` ids with versions, constraint ids.
8. **Checklist**: tick only what you ran this session. Leave the evaluation boxes unticked
   when there is no evaluation run.
9. **Tests**: real numbers (passed, skipped, coverage) and the CI results of the head
   commit. If CI has not finished, say that; do not fill in a result.

Style: concrete names (`module/`, config keys, route paths); no commit hashes in prose
unless a reviewer must open that commit; no file-by-file listing.

## 9. Deliver

Reply in the user's language, briefly: the merge result (conflicts and how each was
resolved), the gates and their numbers, the AI areas affected, and the path to the saved PR
body. Offer `git push` and `gh pr edit <n> --body-file <path>`; do neither unprompted.
