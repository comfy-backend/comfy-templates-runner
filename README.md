# comfy-templates-runner — compute-only GHA runner for comfy-templates

This repository is a **compute-only runner**: it hosts the GitHub
Actions workflow that executes the weekly data refresh of the private
**[trinitylivy/comfy-templates](https://github.com/trinitylivy/comfy-templates)**
repo (a ComfyUI workflow-template explorer: Next.js web app + Python
data pipeline, deployed on Vercel). It intentionally contains *nothing
but the workflow definition and a tiny `state/` dir* — no source code,
no data, no credentials. (Public repos get unlimited Actions minutes,
which is why the workload moved here: the trinitylivy account's Actions
runners are blocked account-wide — every workflow there, even a
bone-standard echo, dies in `startup_failure` with zero jobs, while
this org's runners are healthy.)

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `refresh.yml` | Monday 16:00 UTC + Tuesday 17:07 UTC backstop / dispatch | The weekly upstream data refresh: checks out the private repo via the `GH_PAT` secret, runs the 8-step pipeline (`01–07 + verify`, 100% Python stdlib — no install step), runs the 7-check audit, then race-safely commits the refreshed data (`work/comfy-templates/data` + `public/data`) back to the private repo's `main` as `trinitylivy` — which the connected Vercel project auto-deploys. Keeps `state/last-run.json` fresh (public observability + resets GHA's 60-day schedule-inactivity timer). |

## Why the odd cadence (Mon + Tue backstop)

GitHub's scheduler DROPS a large fraction of scheduled runs under
platform load (observed 2026-09-06 on the sibling hourly runner: ~2/3
dropped). The Tuesday backstop self-heals a dropped Monday; when Monday
already landed, Tuesday finds "no data changes" and exits clean.

## Safety properties

- All credentials (the GitHub PAT used for private checkouts/pushes)
  live as **encrypted GitHub Actions secrets** of this repo — never in
  any file here.
- This repo is public, so **run logs are public**: registered secret
  values are masked automatically by GitHub's log redaction wherever
  they appear. The workflow additionally never interpolates secrets
  into echoed commands.
- **No `pull_request` triggers** exist, so fork PRs can never access
  secrets. Only org admins can push here.
- Every private-repo access authenticates with the `GH_PAT` secret;
  this repo's own default `GITHUB_TOKEN` cannot read or write anything
  private (workflows declare `permissions: {}`).
- The data commit is scoped strictly to pipeline outputs — never
  `git add -A` — and the push is plain (never force; rebase-retry ×3).
- Failure gating: a pipeline step failure withholds the stamp, so the
  commit step never runs — partial data can never land on main.
- Commit identity `trinitylivy <trinitylivy@gmail.com>` (the Vercel
  git integration has skipped deploys for other authors before).

## Triaging a failed run

- **refresh failures** — the pipeline step's log names the failing
  step (01–07 or verify); a failing upstream scrape (comfy.org gallery)
  is usually transient — the next weekly run re-syncs. A 401 on
  checkout/push means the `GH_PAT` secret expired (rotate: set a fresh
  PAT with `repo` scope on this repo's settings). A rejected push means
  a parallel agent session pushed to the private repo's main mid-run —
  the run rebase-retries and fails loudly by design; the next run
  re-syncs.
- **startup_failure with 0 jobs here** would mean the org-level Actions
  allocation broke (never observed; this org's runners are healthy).

## Relationship to the in-repo workflow

The private repo carries its own dormant copy
(`.github/workflows/weekly-refresh.yml`) for the day the account-level
block lifts — then both run the same canonical Monday slot; the
concurrency groups live in different repos, so the practical guard
against double-refresh is the "no data changes" early exit (the refresh
is deterministic; the second run finds nothing to commit). The sandbox
daemon remains a Thursday-09:00-PT local fallback that never commits.
