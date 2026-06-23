# linkedin-job-hunt

A Claude Code plugin that automates LinkedIn job discovery, fit analysis,
and LaTeX resume tailoring. You stay in control of LinkedIn writes and
final applications.

## What it does

1. Reads your LaTeX resume (`resume/base.tex`) and target-company list.
2. Searches LinkedIn for jobs matching your preferences (via the LinkedIn MCP).
3. Scores each job **asymmetrically** — does the job's needs match your
   strengths? (Resume skills not in the JD aren't penalized.)
4. Presents a ranked shortlist; **you approve which jobs to pursue.**
5. For each approved job, tailors your resume (within bounded `<<TAILOR>>`
   regions) and compiles a per-job PDF via `tectonic`.
6. Saves approved jobs to LinkedIn (so they appear on your saved-jobs page
   for manual application).
7. Logs close-but-rejected jobs to `missing-skills.md` so you can see which
   skills are holding you back.

## Why it's structured this way

- **Five focused sub-agents**, one verb each (analyze, search, score, tailor,
  save). Files are the only contract between them — no agent-to-agent
  prompt passing.
- **One approval gate** in the middle of the pipeline. Nothing writes to
  your LinkedIn account until you say which jobs to pursue.
- **Tailor zones** in the LaTeX source (`<<TAILOR-START: ZONE>> ... <<TAILOR-END: ZONE>>`)
  bound where the tailor agent is allowed to edit. Preamble, packages,
  factual content (companies, dates, titles) are off-limits.
- **No code, no tests** — this is a pure-prompt plugin. The five agents
  are LLM-driven; deterministic operations (LaTeX compile, LinkedIn API
  calls) are delegated to `tectonic` and the LinkedIn MCP.

## Prerequisites

- **Claude Code** with the **LinkedIn MCP server** configured and
  authenticated. Run `mcp__linkedin__authenticate_linkedin` once before
  first use.
- **`tectonic`** on `$PATH` — install from
  <https://tectonic-typesetting.github.io/>.
  - macOS: `brew install tectonic`
  - Linux: see project install guide
  - Verify: `which tectonic`

## Workspace setup

Pick a directory to be your job-hunt workspace (separate from this plugin
repo), then bootstrap it:

```bash
mkdir my-job-hunt && cd my-job-hunt

# copy the templates from this plugin
cp <plugin-path>/templates/preferences.yaml      preferences.yaml
cp <plugin-path>/templates/target-companies.yaml target-companies.yaml
mkdir resume
cp <plugin-path>/templates/resume.base.tex       resume/base.tex
```

Now edit those three files:
- **`resume/base.tex`** — replace placeholder content with your real
  resume. **Keep the `% <<TAILOR-START: ...>>` and `% <<TAILOR-END: ...>>`
  comment markers** — they're how the tailor agent knows which regions
  it's allowed to edit. Required zones: `SUMMARY`, `SKILLS`, `BULLETS`.
- **`preferences.yaml`** — set your locations, remote policy, salary min,
  must-avoid keywords, scoring thresholds.
- **`target-companies.yaml`** — list the companies you want crawled and
  optional per-company keyword filters.

## Running

In Claude Code, with your workspace as the cwd:

```
/job-hunt
```

The orchestrator runs Phases 0–7:

1. **Validate** — checks resume, configs, tectonic, LinkedIn auth
2. **Profile** — extracts a structured skills profile from your resume (cached)
3. **Search** — LinkedIn MCP queries across companies + core skills
4. **Score** — asymmetric fit analysis per job
5. **GATE** — shows you a ranked table; you reply with row numbers, `all`, or `none`
6. **Tailor** — generates per-job tailored LaTeX + PDF for approved jobs (parallel)
7. **Save** — saves approved jobs to LinkedIn; appends rejected close-fits to `missing-skills.md`

Outputs land in:
- `runs/<timestamp>/` — per-invocation history (jobs, scores, shortlist, log)
- `opportunity/<job_id>/` — per-job folders (`job.json`, `fit-analysis.md`,
  `resume.tex`, `resume.pdf`, `tailor-diff.md`)
- `missing-skills.md` — accumulated skill-gap ledger

### Re-running a single phase

If you tune `preferences.yaml` and want to re-score without re-searching:

```
/job-hunt phase=score
```

If you want to re-tailor a previous run's shortlist (e.g., after editing
your `base.tex`):

```
/job-hunt phase=tailor run=2026-05-28T14-00-00Z
```

## Workspace layout (after a few runs)

```
my-job-hunt/
├── resume/base.tex                # your canonical LaTeX resume — never auto-edited
├── preferences.yaml
├── target-companies.yaml
├── .job-hunt/skills-profile.json  # cache (regenerated when base.tex changes)
├── runs/
│   └── 2026-05-28T14-00-00Z/
│       ├── jobs.json
│       ├── scored-jobs.json
│       ├── shortlist.json
│       └── log.md
├── opportunity/
│   └── 3812345678/
│       ├── job.json
│       ├── fit-analysis.md
│       ├── resume.tex             # tailored copy
│       ├── resume.pdf             # tectonic output
│       └── tailor-diff.md         # human-readable diff for review
└── missing-skills.md              # skill-gap ledger
```

## How the tailor zones work

Your `resume/base.tex` is the source of truth and is never modified by
the plugin. The tailor agent is allowed to edit ONLY between sentinel
comments:

```latex
\section*{Summary}
% <<TAILOR-START: SUMMARY>>
... your summary text ...
% <<TAILOR-END: SUMMARY>>
```

For each shortlisted job, it:
1. Copies `resume/base.tex` to `opportunity/<job_id>/resume.tex`
2. Edits inside the `SUMMARY`, `SKILLS`, and `BULLETS` zones to lead with
   the JD's required skills (using vocabulary from the JD where the
   candidate genuinely has the skill — never inventing experience)
3. Compiles via `tectonic resume.tex`
4. Writes `tailor-diff.md` with a unified diff so you can verify edits
   stayed in-bounds

If you find an edit went outside the zones, that's a bug — file a report
showing the offending diff.

## Limitations

- **No Easy Apply.** LinkedIn does not expose application submission via
  API, so the plugin saves jobs to your saved-jobs page and you apply
  manually. This is by design; auto-apply violates LinkedIn ToS.
- **Tailor zone discipline is prompt-enforced.** The tailor agent is
  instructed to stay inside sentinels but there's no runtime guard.
  Always review `tailor-diff.md` before applying.
- **LinkedIn API limits.** Heavy queries can hit rate limits. The
  searcher backs off exponentially but won't paper over a hard cap; see
  `runs/<ts>/log.md` for `partial: true` flags.

## Plugin layout (this repo)

```
linkedin-job-hunt/
├── .claude-plugin/plugin.json
├── agents/
│   ├── jh-resume-analyzer.md
│   ├── jh-job-searcher.md
│   ├── jh-fit-scorer.md
│   ├── jh-resume-tailor.md
│   └── jh-linkedin-saver.md
├── skills/
│   └── job-hunt/SKILL.md
├── templates/
│   ├── preferences.yaml
│   ├── target-companies.yaml
│   └── resume.base.tex
└── README.md
```

Pure markdown — no Python, no tests, no build step. The agents are
LLM-driven prompts; the skill is the orchestrator.
