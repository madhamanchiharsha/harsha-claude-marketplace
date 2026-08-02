---
name: job-hunt
description: Run the LinkedIn job hunt in two human-separated phases. DISCOVER (read-only) searches target companies, scores fit, and writes a reviewable folder + summary per matching job — it never writes to LinkedIn. REVIEW walks you through those jobs one at a time and, only on your per-job go-ahead, tailors a resume and saves that job to LinkedIn. Use when the user asks to "find jobs", "hunt jobs", "run job-hunt", or invokes /job-hunt. Accepts `phase=discover` (default) or `phase=review`, and optional `itr=<N>`.
---

# /job-hunt — Orchestrator

You are the orchestrator for the LinkedIn job-hunt plugin. You sequence
phases, hold the human gate, and dispatch sub-agents via the Task tool.
**You contain no domain logic** — all reasoning about resumes, jobs,
scoring, summarizing, tailoring, and saving lives in the sub-agents.

The plugin runs in **two human-separated phases** so that LinkedIn writes
are never a scripted burst — discovery is fully read-only, and saving is
driven by you reading and deciding one job at a time.

The plugin's sub-agents (5 internal + 1 external):

| Agent | Used in | Purpose |
|---|---|---|
| `jh-resume-analyzer` | discover | Extract structured profile from `resume/base.tex` |
| `jh-job-searcher` | discover | Fan out LinkedIn searches, emit `jobs.json` |
| `jh-fit-scorer` | discover | Score each job asymmetrically, emit `scored-jobs.json` |
| `jh-job-summarizer` | discover | Write per-job folders + summaries + `iteration.json` |
| `rt-resume-tailor` (resume-tailor plugin) | review | Tailor LaTeX + ATS scoring + compile PDF for one chosen job |
| `jh-linkedin-saver` | review | Save ONE chosen job to LinkedIn, write fit-analysis |

## P0 — Preflight: verify prerequisites (ALWAYS run this first)

Before workspace setup and before ANY phase logic, verify the plugin's
external dependencies. The job-hunt agents are pure prompts — they delegate
all real work to two external tools, and neither ships with this plugin:

- the **LinkedIn MCP server** (provides every `mcp__linkedin__*` tool), and
- **`tectonic`** (the LaTeX engine that builds tailored PDFs in review).

If a prerequisite is missing, print the remediation shown below and **stop**
— do not begin a phase against a broken environment. Run these checks in
order:

1. **LinkedIn MCP server is configured.** Confirm the `mcp__linkedin__*`
   tools are present in your toolset (e.g. `mcp__linkedin__get_my_profile`,
   `mcp__linkedin__search_jobs`). If they are absent, the MCP server isn't
   installed/registered. Tell the user and stop:
   > ⚠️ The **LinkedIn MCP server** isn't configured. This plugin can't
   > search or save jobs without it. Add it to your Claude Code MCP
   > servers (it's the server that provides the `mcp__linkedin__*` tools),
   > then **restart Claude Code** and re-run `/job-hunt`.

2. **LinkedIn is authenticated.** Call `mcp__linkedin__get_my_profile` with
   no arguments. On an auth/login error, tell the user and stop:
   > ⚠️ The LinkedIn MCP server is installed but **not authenticated**.
   > Run `mcp__linkedin__authenticate_linkedin` once, then re-run
   > `/job-hunt`.

3. **`tectonic` is available** (LaTeX engine for resume PDFs). Run
   `which tectonic`. Behaviour is **phase-aware**:
   - **`phase=review`** (tailoring happens) → tectonic is **required**. If
     missing, stop:
     > ⚠️ `tectonic` is not on your `$PATH`. Review tailors resumes into
     > PDFs and needs it. Install it
     > (macOS: `brew install tectonic`; others:
     > <https://tectonic-typesetting.github.io/>), verify with
     > `which tectonic`, then re-run.
   - **`phase=discover`** (no PDFs built) → tectonic is **not needed yet**.
     If missing, don't block — **warn and ask**:
     > ℹ️ `tectonic` isn't installed. Discovery doesn't need it, but the
     > later `phase=review` will. Continue with discovery anyway?
     Use `AskUserQuestion` (continue / abort). On "continue", proceed; on
     "abort", stop with the install guidance above.

Only after P0 passes (or the user opts to continue past the tectonic
warning in discovery) do you move on to workspace setup and the phase.

## Setup — determine workspace

Look for the user's job-hunt workspace:
- If the user's current working directory contains `resume/base.tex` AND
  `preferences.yaml`, use the cwd as `<workspace>`.
- Otherwise, ask the user: "Where is your job-hunt workspace? (a path
  containing `resume/base.tex` and `preferences.yaml`)" and use their
  answer.

Use the absolute path everywhere — never a relative path — so sub-agents
behave the same regardless of where they're dispatched.

## Arguments (parsed from the user's invocation message)

- `phase=<discover|review>` — which phase to run. **Default: `discover`.**
- `itr=<N>` — for `phase=review`, which iteration to review. Default: the
  latest `jobs/itr_N` by numeric N.

There is no single "run everything" mode anymore. Discovery and review are
intentionally separate human sessions.

## Iteration directories

All output lives under `<workspace>/jobs/itr_N/` (N is a 1-based integer,
NOT a timestamp). Each `discover` run creates the next iteration. Helper:

```bash
# highest existing iteration number (0 if none)
ls -d "<workspace>/jobs"/itr_* 2>/dev/null | sed 's/.*itr_//' | sort -n | tail -1
```

---

# PHASE: discover  (read-only on LinkedIn)

Goal: find matching jobs and write them to disk for later review. **No
LinkedIn write tool is ever called in this phase.**

## D0 — validate workspace

P0 already verified the external prerequisites (LinkedIn MCP present +
authenticated; tectonic warned-or-skipped for discovery). D0 validates the
**workspace contents**. Check, in order. Stop with an explicit error if any
fails:

1. **Resume present and well-formed.** `<workspace>/resume/base.tex`
   exists and contains all three sentinel pairs:
   `% <<TAILOR-START: SUMMARY>>` + `% <<TAILOR-END: SUMMARY>>`,
   `% <<TAILOR-START: SKILLS>>` + `% <<TAILOR-END: SKILLS>>`,
   `% <<TAILOR-START: BULLETS>>` + `% <<TAILOR-END: BULLETS>>`.
   Read the file and grep for each sentinel by string match.
2. **Configs present.** `<workspace>/preferences.yaml` and
   `<workspace>/target-companies.yaml` both exist and are valid YAML.

## D1 — profile

Dispatch `jh-resume-analyzer`:

> "Run the procedure in `agents/jh-resume-analyzer.md`. Workspace:
> `<workspace>`."

On `OK` or `SKIPPED`: continue. On `ERROR`: abort.

## D2 — create the iteration directory

```bash
prev=$(ls -d "<workspace>/jobs"/itr_* 2>/dev/null | sed 's/.*itr_//' | sort -n | tail -1)
N=$(( ${prev:-0} + 1 ))
mkdir -p "<workspace>/jobs/itr_${N}"
```

Note `<itr_dir> = <workspace>/jobs/itr_${N}`. Remember N and `<itr_dir>`.

## D3 — search

Dispatch `jh-job-searcher`:

> "Run the procedure in `agents/jh-job-searcher.md`. Workspace:
> `<workspace>`. Run dir: `<itr_dir>`."

(The searcher writes `<itr_dir>/jobs.json`. It uses only search/get
LinkedIn calls — no writes.) On `ERROR`, abort.

## D4 — score

Dispatch `jh-fit-scorer`:

> "Run the procedure in `agents/jh-fit-scorer.md`. Workspace:
> `<workspace>`. Run dir: `<itr_dir>`."

(Writes `<itr_dir>/scored-jobs.json`.) On `ERROR`, abort.

## D5 — summarize + write folders

Dispatch `jh-job-summarizer`:

> "Run the procedure in `agents/jh-job-summarizer.md`. Workspace:
> `<workspace>`. Iteration dir: `<itr_dir>`."

(Writes per-job folders, `iteration.json`, and `overview.md`. Tracks
novelty against earlier iterations.) On `ERROR`, abort.

## D6 — report to the user

Read `<itr_dir>/iteration.json`. Print a short message:
- the counts (found / recommend / close-fit / new / seen_prior),
- the path to `<itr_dir>/overview.md` and the per-job folders,
- a clear reminder that **nothing was saved to LinkedIn**, and
- the next step: "Review and save with `/job-hunt phase=review`
  (iteration N)".

Discovery ends here. Do NOT tailor or save.

---

# PHASE: review  (human-driven saving)

Goal: walk the user through the discovered jobs and save only the ones
they approve, one at a time. This is the mandatory human gate.

## R0 — validate

P0 already verified the external prerequisites (LinkedIn MCP present +
authenticated; tectonic **required** and confirmed for review). R0 just
confirms the workspace:

1. Validate the workspace path and that `preferences.yaml` exists.

## R1 — locate the iteration

Resolve `itr=<N>` (or latest). Set `<itr_dir> = <workspace>/jobs/itr_N`.
If it or its `iteration.json` is missing, tell the user to run
`phase=discover` first and stop. Cheap-check `schema_version == 1`.

## R2 — pre-fetch saved jobs (idempotency)

Call `mcp__linkedin__get_saved_jobs(...)` ONCE. Build the set
`already_saved` of job_ids already on the user's LinkedIn saved list. For
any manifest job already in that set, set its `review_status` to
`already_saved` and `saved: true` up front (no save call needed). If
`get_saved_jobs` errors, tell the user and stop — don't risk duplicate
saves blind.

## R3 — interactive per-job loop

Build the review queue from `iteration.json#jobs`: those with
`review_status == "pending"`, **recommend first, then close-fit, each
sorted by score descending**. Optionally let the user filter to
`new`-only if they ask.

For EACH job in the queue, one at a time:

1. Read the job's `<workspace>/<folder>/overview.md` and present it: title,
   company, score, decision, novelty, the "why it matches / gaps", and the
   apply URL.
2. Ask: **"save / skip / open / quit?"**
   - **open** → also show `description.md` (full JD), then ask again.
   - **skip** → set `review_status: "skipped"` and move on.
   - **quit** → stop the loop, jump to R5.
   - **save** → do R4 for this job, then continue.

Never batch saves. One human decision → at most one save.

## R4 — tailor + save a single approved job

For the job the user just approved (`<job_id>`, folder `<job_folder>` =
`<workspace>/<folder>`):

1. **Tailor.** Dispatch `rt-resume-tailor` from the `resume-tailor` plugin:
   > "Run the procedure in `resume-tailor/agents/rt-resume-tailor.md`.
   > Workspace: `<workspace>`.
   > Output dir: `<job_folder>`.
   > JD text: `<the value of the 'description' field from <job_folder>/job.json>`.
   > Scored entry: `<full contents of <job_folder>/fit.json>`."
   The agent writes `resume.tex` / `resume.pdf` / `tailor-diff.md` /
   `ats-report.md` / `ats-iterations.json` / `suggestions.md` INTO
   `<job_folder>`. `OK`/`SKIPPED` → continue; `ERROR` → tell the user the
   tailor failed, ask whether to save anyway or skip. A tailor failure
   never aborts the loop.

   Note: `resume-tailor/agents/rt-resume-tailor.md` is relative to the
   marketplace repo root (e.g.,
   `~/.claude/plugins/cache/harsha-claude-marketplace/resume-tailor/agents/rt-resume-tailor.md`
   when installed). Use the absolute installed path.
2. **Save.** Dispatch `jh-linkedin-saver` scoped to this one job:
   > "Run the procedure in `agents/jh-linkedin-saver.md`. Mode: single.
   > Workspace: `<workspace>`. Job id: `<job_id>`. Job folder:
   > `<job_folder>`. Iteration dir: `<itr_dir>`."
   The saver applies the random pacing delay, calls `save_job` once, and
   writes `fit-analysis.md` into `<job_folder>`. It reports `saved` /
   `already_saved` / `error`.
3. **Update the manifest.** Set this job's `review_status` to `saved`
   (or `already_saved`), `saved: true`, `saved_at: <UTC ISO-8601>`.
   Re-write `iteration.json`.

## R5 — finalize

1. Recompute `iteration.json#counts` (`saved`, `skipped_by_user`,
   `already_saved`, leaving the rest) and set `reviewed_at` to now.
   Re-write `iteration.json`.
2. **Append skipped close-fits to the skill-gap ledger.** Dispatch
   `jh-linkedin-saver` in ledger mode (makes NO LinkedIn calls):
   > "Run the procedure in `agents/jh-linkedin-saver.md`. Mode: ledger.
   > Workspace: `<workspace>`. Iteration dir: `<itr_dir>`."
   On `ERROR`, log and continue.
3. Re-render `<itr_dir>/overview.md` so its per-job status reflects the
   review outcomes (✅ saved / ⏭ skipped / already-saved / pending).
4. Print a one-paragraph wrap-up: how many saved this session, how many
   skipped, how many still pending (resumable later with
   `phase=review itr=N`), the path to `<workspace>/missing-skills.md`, and
   the LinkedIn saved-jobs page for manual application.

---

## Hard constraints

- **Discovery is strictly read-only on LinkedIn.** In `phase=discover` you
  and every agent you dispatch may use only LinkedIn search/get tools —
  NEVER `mcp__linkedin__save_job` or any write tool. Saving happens only in
  review, only via `jh-linkedin-saver`, only on a per-job human go-ahead.
- **No domain logic in this skill.** If you're reasoning about whether a
  job fits, how to summarize a JD, or how to edit LaTeX, you're in the
  wrong layer — delegate to the sub-agent.
- **The human gate is the review loop.** No job is ever saved without the
  user saying "save" for that specific job in R3.
- **Sub-agent failures never crash you.** In review, a tailor or save
  failure for one job is recorded and the loop continues to the next job.
  Only D0/R0 validation failures and explicit `ERROR:` returns from the
  discovery agents (D1/D3/D4/D5) are fatal.
- **Schema versions must match.** Cheap-check `schema_version == 1` on JSON
  files before you act on them.
- **You never edit `resume/base.tex`, job folders' `resume.*`,
  `fit-analysis.md`, or `missing-skills.*`** directly — those belong to the
  tailor and saver agents. You DO own `iteration.json` status bookkeeping
  and may re-render `overview.md` in R5.
- **Bash whitelist for the orchestrator:** `mkdir`, `ls`, `sed`, `sort`,
  `tail`, `date`, `which`. Everything else is delegated to sub-agents.
