---
name: resume-tailor
description: Tailor a LaTeX resume (base.tex) to any job description using ATS-scored iterative rewriting (max 5 rounds). Fetches portfolio website for supplemental context. Outputs resume.pdf, ats-report.md, suggestions.md. Use when the user says "tailor my resume", "rewrite my resume for this job", or invokes /resume-tailor. Accepts optional jd=<path> and output=<path> arguments.
---

# /resume-tailor — Skill Orchestrator

You sequence the preflight, resolve paths, dispatch the agent, and report
results. All tailoring logic lives in `agents/rt-resume-tailor.md`.

## P0 — Preflight

Run these checks in order. Stop on the first failure.

**1. tectonic present:**

```bash
which tectonic
```

If not found, stop:

> ⚠️ `tectonic` is not on your `$PATH`. This skill needs it to compile
> LaTeX resumes to PDF. Install it (`brew install tectonic` on macOS, or
> see <https://tectonic-typesetting.github.io/>), verify with
> `which tectonic`, then re-run `/resume-tailor`.

**2. `resume/base.tex` exists and has sentinels:**

Check that `<workspace>/resume/base.tex` exists and contains all three
sentinel pairs by grepping:

```bash
grep -c "TAILOR-START: SUMMARY\|TAILOR-START: SKILLS\|TAILOR-START: BULLETS" \
  "<workspace>/resume/base.tex"
```

Expected output: `3`. If less, stop:

> ⚠️ `resume/base.tex` is missing one or more required TAILOR sentinel zones
> (SUMMARY, SKILLS, BULLETS). See `templates/resume.base.tex` in the
> resume-tailor plugin for the required format.

**3. JD source resolved:**

If the user's invocation includes `jd=<path>`:
- Read the file at that path. Its full text is `<jd_text>`.

If `jd=` is not provided:
- Ask: "Please paste the job description text (or type `jd=path/to/file.txt`
  to provide a file):"
- Wait for the user's response. Use the provided text as `<jd_text>`.

## Workspace resolution

If `resume/base.tex` exists in the current working directory, use cwd as
`<workspace>`. Otherwise ask:

> "Where is your resume-tailor workspace? (a directory containing
> `resume/base.tex` and `preferences.yaml`)"

Use their answer as `<workspace>` (absolute path).

## Output dir resolution

If the user's invocation includes `output=<path>`:
- Use that path as `<output_dir>` (resolve to absolute).

Otherwise:
- Extract company and role from `<jd_text>` (first pass, best-effort).
  Sanitize: lowercase, spaces to hyphens, keep only `[a-z0-9-]`, max 40 chars.
  Example: "Senior Backend Engineer at Stripe" → `stripe-senior-backend-engineer`.
  If neither company nor role is extractable, use `tailored` as the slug.
- Set `<output_dir> = <workspace>/tailored/<slug>/`.

## Dispatch

Dispatch `agents/rt-resume-tailor.md` (path relative to the plugin root,
i.e., the directory containing `resume-tailor/.claude-plugin/plugin.json`)
with:

> "Run the procedure in `resume-tailor/agents/rt-resume-tailor.md`.
> Workspace: `<workspace>`.
> JD text: `<jd_text>`.
> Output dir: `<output_dir>`."

## Report results

On `OK`:
> ✅ Resume tailored successfully.
>
> **ATS score:** `<score>/100` (`<iterations>` iteration(s))
>
> **Outputs written to `<output_dir>`:**
> - `resume.pdf` — tailored resume (ready to submit)
> - `ats-report.md` — score breakdown and remaining gaps
> - `tailor-diff.md` — diff vs your base.tex
> - `ats-iterations.json` — score trajectory
> - `suggestions.md` — what to add to base.tex for future runs

On `SKIPPED`:
> ℹ️ Already tailored — `<output_dir>/resume.pdf` already exists. Delete it
> and re-run if you want to re-tailor.

On `ERROR`:
> ❌ Tailoring failed: `<error message>`
> Check `<output_dir>/tailor-error.md` for diagnostics.

## Hard constraints

- **No domain logic here.** If you find yourself reasoning about resume
  content, JD keywords, or LaTeX — stop and delegate to the agent.
- **Never call the agent twice** for the same job without the user asking.
- **Bash whitelist for this skill:** `which`, `grep`, `ls`.
