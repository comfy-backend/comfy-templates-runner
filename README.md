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
| `refresh.yml` | Wednesday 16:00 UTC + Thursday 17:07 UTC backstop / dispatch (`force_blobs` input) | The WEEKLY BACKUP lane of the data refresh (the PRIMARY lane is the Netlify daily scrape on comfy-backend/comfy-scraper, live 2026-10-09): checks out the private repo via the `GH_PAT` secret, runs the 8-step pipeline (`01–07 + verify`, 100% Python stdlib — no install step), runs the 10-check audit (A–J), then race-safely commits the refreshed data (`work/comfy-templates/data` + `public/data`) back to the private repo's `main` as `trinitylivy` — which the connected Vercel project auto-deploys. Also mirrors to GitLab (fatal) + snapshots to Netlify Blobs (warning-class). Keeps `state/last-run.json` fresh (public observability + resets GHA's 60-day schedule-inactivity timer). |

## Why the Wed + Thu cadence (W18, 2026-10-09)

The Netlify lane (comfy-backend/comfy-scraper) now fires DAILY at
04:00 UTC — it is the primary. This GHA workflow is the weekly backup
lane, moved off Monday so it never races the daily fires. GitHub's
scheduler DROPS a large fraction of scheduled runs under platform load
(observed 2026-09-06 on the sibling hourly runner: ~2/3 dropped), so
the Thursday backstop self-heals a dropped Wednesday. NOTE: a green
backup-lane run does NOT "find no data changes and exit clean" — every
run re-stamps `stats.json` `built_at`, so each green run commits a
small churn commit. That churn is the DESIGNED liveness heartbeat; do
not "fix" it. The standby watchdog (daily 04:30 + 16:30 UTC) alerts
and self-heal-dispatches this workflow if everything goes quiet.

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
block lifts. The practical guard against double-refresh across lanes
is the race-safe push (rebase + retry) — every green run re-stamps
`built_at`, so same-day runs land as small churn commits, newest wins.

## Rollback (last-good corpus)

If a bad corpus lands on prod (a gate regression, a bad pin, upstream
poison), roll the data back — the corpus is pure data commits:

```bash
# 1. find the last good data commit (before the bad one)
git log --oneline -- public/data | head -5
# 2. revert ONLY the data paths (never the whole tree — app code may have
#    moved on since)
git revert --no-commit <bad-sha> && git restore --staged --worktree -- :^public/data ^work/comfy-templates/data && git checkout HEAD -- . ':!public/data' ':!work/comfy-templates/data' 2>/dev/null || true
git commit -m "revert(data): roll back to the last good corpus (<reason>)"
git push origin main   # Vercel auto-deploys the rollback
# 3. while you fix the gate: freeze the Netlify daily fire (delete the
#    build hook on the scraper site, or set the fn schedule far out) and
#    disable the Wednesday cron here — or fix-forward and let the next
#    run re-land fresh data
curl -X PUT -H "Authorization: token $PAT" \
  https://api.github.com/repos/comfy-backend/comfy-templates-runner/actions/workflows/refresh.yml/disable
```

The simplest reliable variant: `git revert <bad-data-commit>` (data
commits touch only public/data + work/comfy-templates/data, so a plain
revert is safe) → push → re-enable after the fix.
