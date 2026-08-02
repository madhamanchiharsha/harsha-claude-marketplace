# resume-tailor Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract `jh-resume-tailor` into a standalone `resume-tailor` plugin with ATS scoring, portfolio website caching, and iterative improvement (max 5 rounds, no degradation).

**Architecture:** A new `resume-tailor/` subdirectory becomes its own plugin with one public skill (`/resume-tailor`) and one agent (`rt-resume-tailor.md`). The agent inlines the scoring logic from `jh-fit-scorer` so it needs no pre-computed `fit.json` when called standalone, but accepts one when called from `linkedin-job-hunt`. The `linkedin-job-hunt` skill's R4 step is updated to delegate to the new agent, and `agents/jh-resume-tailor.md` is deleted.

**Tech Stack:** Markdown agent files (Claude Code plugin format), LaTeX/tectonic, YAML, JSON, WebFetch tool, Bash (cp, mkdir, diff, shasum, tectonic)

---

### Task 1: Scaffold `resume-tailor` plugin structure

**Files:**
- Create: `resume-tailor/.claude-plugin/plugin.json`
- Create: `resume-tailor/skills/resume-tailor/` (directory)
- Create: `resume-tailor/agents/` (directory)

- [ ] **Step 1: Create the plugin manifest**

Create `resume-tailor/.claude-plugin/plugin.json` with this exact content:

```json
{
  "name": "resume-tailor",
  "version": "0.1.0",
  "description": "Standalone LaTeX resume tailoring with ATS scoring. Tailors base.tex against any job description, fetches portfolio website for supplemental context, iterates up to 5 times without degrading the ATS score, and writes actionable suggestions for manual improvements.",
  "author": {
    "name": "harshavardhanmadhamanchi"
  },
  "skills": "./skills/"
}
```

- [ ] **Step 2: Create the directory skeleton**

```bash
mkdir -p resume-tailor/.claude-plugin
mkdir -p resume-tailor/skills/resume-tailor
mkdir -p resume-tailor/agents
```

Verify:

```bash
ls resume-tailor/.claude-plugin/
ls resume-tailor/skills/
ls resume-tailor/agents/
```

Expected: `plugin.json` in `.claude-plugin/`, `resume-tailor/` in `skills/`, empty `agents/`.

- [ ] **Step 3: Update `marketplace.json` to register the new plugin**

Read `.claude-plugin/marketplace.json`. It currently contains:

```json
{
  "name": "harsha-claude-marketplace",
  "owner": { "name": "harshavardhanmadhamanchi" },
  "description": "Personal Claude Code plugin marketplace by harshavardhanmadhamanchi",
  "plugins": [
    {
      "name": "linkedin-job-hunt",
      "description": "Multi-agent LinkedIn job hunt with LaTeX resume tailoring. Searches target companies, scores fit asymmetrically against your resume, generates tailored per-job PDFs via tectonic, and saves approved jobs to LinkedIn.",
      "source": "./",
      "version": "0.1.0"
    }
  ]
}
```

Replace with:

```json
{
  "name": "harsha-claude-marketplace",
  "owner": { "name": "harshavardhanmadhamanchi" },
  "description": "Personal Claude Code plugin marketplace by harshavardhanmadhamanchi",
  "plugins": [
    {
      "name": "linkedin-job-hunt",
      "description": "Multi-agent LinkedIn job hunt with LaTeX resume tailoring. Searches target companies, scores fit asymmetrically against your resume, generates tailored per-job PDFs via tectonic, and saves approved jobs to LinkedIn.",
      "source": "./",
      "version": "0.1.0"
    },
    {
      "name": "resume-tailor",
      "description": "Standalone LaTeX resume tailoring with ATS scoring. Tailors base.tex against any job description, fetches portfolio website for supplemental context, iterates up to 5 times without degrading ATS score, and writes actionable suggestions.",
      "source": "./resume-tailor/",
      "version": "0.1.0"
    }
  ]
}
```

- [ ] **Step 4: Commit**

```bash
git add resume-tailor/.claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "feat(resume-tailor): scaffold plugin structure and register in marketplace"
```

---

### Task 2: Write the `rt-resume-tailor` agent — steps 0–4 (profile + JD extraction)

**Files:**
- Create: `resume-tailor/agents/rt-resume-tailor.md`

This task writes the agent frontmatter and Steps 0–4 only (idempotency check, base.tex profiling, website cache, JD extraction, match computation). The iterative loop (Step 5) and output writing (Step 6) are added in Task 3 to keep each task reviewable.

- [ ] **Step 1: Create `resume-tailor/agents/rt-resume-tailor.md` with the frontmatter and steps 0–4**

```markdown
---
name: rt-resume-tailor
description: Rewrites base.tex TAILOR zones against a job description, scores ATS fitness iteratively (max 5 rounds, no degradation), and writes resume.pdf + ats-report.md + suggestions.md. Callable standalone (pass jd_text) or from linkedin-job-hunt (pass scored_entry to skip inline scoring).
tools: Read, Write, Edit, Bash, WebFetch
---

# rt-resume-tailor

You produce one tailored resume for one job description. You may be dispatched
by the `/resume-tailor` skill (standalone) or by the `linkedin-job-hunt`
orchestrator (review phase). Behave identically in both cases.

## Inputs

The caller passes:

| Field | Required | Description |
|---|---|---|
| `<workspace>` | yes | Absolute path containing `resume/base.tex` and `preferences.yaml` |
| `<jd_text>` | yes | Full job description as inline text |
| `<output_dir>` | yes | Absolute path where all outputs are written |
| `<scored_entry>` | no | Pre-computed fit fields from job-hunt's `fit.json`. If provided, skip Steps 1–4. |

## Step 0 — Idempotency check

If `<output_dir>/resume.pdf` already exists, print `SKIPPED: already tailored`
and stop. Do not re-tailor.

## Step 1 — Profile `base.tex`

Read `<workspace>/resume/base.tex` as plain text. Extract inline:

- **`core_skills`** (4–8 items): candidate's top strengths. These appear in the
  summary, the skills section header, and the majority of bullets in the most
  recent role. Keep lowercase, short (`"go"`, `"distributed systems"`).

- **`secondary_skills`** (4–10 items): supporting tools from older roles or
  secondary lines in the skills section. Anything not `core_skills`.

- **`tech_stack`** (deduplicated array): every named technology in the file,
  union of core + secondary + anything in bullets.

- **`current_role`** (string): the most-recent job title (first `\textit{...}`
  after the most-recent `\textbf{Company}` block).

Do NOT write a cache file — this is inline only.

Skip this step if `<scored_entry>` was provided.

## Step 2 — Fetch and cache portfolio website

Read `portfolio_url` from `<workspace>/preferences.yaml`. If the field is
absent or empty, skip this step entirely.

Ensure cache dir:

```bash
mkdir -p "<workspace>/.resume-tailor"
```

Read `<workspace>/.resume-tailor/website-cache.json` if it exists.

Fetch the website using `WebFetch` with a prompt of
`"Extract: author summary/bio, list of projects with descriptions, list of skills/technologies, and any notable links. Return as structured text."`.

Compute sha256 of the raw fetched content:

```bash
echo -n "<fetched_content>" | shasum -a 256 | awk '{print "sha256:"$1}'
```

Compare against `content_hash` in the cache:

- **Match** → use `extracted` from cache as-is, skip re-parse.
- **Mismatch or no cache** → extract structured fields from the fetched content
  (summary, projects array, skills array, notable_links array). Write:

```json
{
  "url": "<portfolio_url>",
  "content_hash": "sha256:<hash>",
  "extracted": {
    "summary": "...",
    "projects": ["..."],
    "skills": ["..."],
    "notable_links": ["..."]
  }
}
```

to `<workspace>/.resume-tailor/website-cache.json`.

Merge `extracted.skills` into `secondary_skills` (additive, no duplicates).
Merge `extracted.projects` as context — note them but they do not override
anything in `base.tex`.

Skip this step if `<scored_entry>` was provided.

## Step 3 — Extract JD signals

From `<jd_text>`, identify:

- **`jd_required`**: skills/keywords the JD treats as required. Signals:
  "must", "required", "X+ years of", "strong proficiency in", "deep
  experience with", "we expect you to have". Normalize: `"Golang"` → `"go"`,
  `"K8s"`/`"Kubernetes"` → `"k8s"`, `"Postgres"`/`"PostgreSQL"` → `"postgres"`.
  Lowercase, short forms.

- **`jd_preferred`**: nice-to-haves. Signals: "plus", "bonus", "nice to have",
  "optional", "preferred", "ideal candidate would also have".

- **`company`**: company name, if extractable (used for output dir slug).
- **`role`**: job title, if extractable (used for output dir slug).

Skip this step if `<scored_entry>` was provided (use its `jd_required` and
`jd_preferred` directly).

## Step 4 — Compute match

Skip if `<scored_entry>` provided — use its fields directly.

Compute (liberal semantic matching — `"event-driven systems"` matches
`"event-driven"`, `"distributed systems"` matches `"distributed"`):

- `core_required_match` = `jd_required` ∩ `core_skills`
- `secondary_required_match` = (`jd_required` ∩ `secondary_skills`) − `core_required_match`
- `unmet_required` = `jd_required` − (`core_skills` ∪ `secondary_skills` ∪ website extracted skills)
```

- [ ] **Step 2: Verify the file was created**

```bash
head -5 resume-tailor/agents/rt-resume-tailor.md
```

Expected: the `---` frontmatter opening line.

- [ ] **Step 3: Commit**

```bash
git add resume-tailor/agents/rt-resume-tailor.md
git commit -m "feat(resume-tailor): add agent steps 0-4 (profile, website cache, JD extraction, match)"
```

---

### Task 3: Write `rt-resume-tailor` agent — Step 5 (iterative tailor loop)

**Files:**
- Modify: `resume-tailor/agents/rt-resume-tailor.md` (append Step 5)

- [ ] **Step 1: Append Step 5 to `resume-tailor/agents/rt-resume-tailor.md`**

Append the following content to the end of the file:

```markdown

## Step 5 — Iterative tailor loop (max 5 iterations)

Ensure output dir:

```bash
mkdir -p "<output_dir>"
```

Initialize state:
- `best_tex` = contents of `<workspace>/resume/base.tex`
- `best_score` = 0
- `scores` = []
- `stop_reason` = `"max_iterations"`
- `best_iteration` = 0
- `last_gap_report` = [] (empty on first iteration)

Run the loop up to 5 times. On each iteration `i` (1-indexed):

### 5a — Copy and edit

```bash
cp "<workspace>/resume/base.tex" "<output_dir>/resume.tex"
```

Edit ONLY inside the three TAILOR zone markers using the Edit tool. Never
touch anything outside `% <<TAILOR-START: ZONE>>` / `% <<TAILOR-END: ZONE>>`
pairs — not the preamble, not section headers, not company names, dates,
or education. Never rename, move, or add sentinel lines.

**SUMMARY zone** (priority 1):
Rewrite to lead with `core_required_match` skills and JD domain vocabulary.
Keep all facts (years of experience, role names) identical to `base.tex`.
Use phrasing from `jd_required`. ≤4 sentences total.
If `i > 1`, additionally work in keywords from `last_gap_report` that semantically
fit the summary.

**SKILLS zone** (priority 2):
Reorder skill lines so `jd_required` + `jd_preferred` skills the candidate
has appear earliest in their respective category lines. You may reword a
category label (e.g., `"Infrastructure:"` → `"Distributed Systems:"`) to
match JD vocabulary if that category's skills are in `jd_required`.
Never add skills not present in `base.tex` or `website-cache extracted.skills`.
If `i > 1`, surface remaining gap keywords from `last_gap_report` earlier
in the skills lines if they are present anywhere in `base.tex`.

**BULLETS zone** (priority 3):
Lightly reword 1–2 bullets to use JD vocabulary. Permitted changes:
- Prepend JD term: `"async pipelines"` → `"event-driven async pipelines"` (if JD says event-driven)
- Append context: `"50M jobs/day"` → `"high-throughput pipeline (50M jobs/day)"` (if JD says high-throughput)
- Synonym substitution if the JD term and current term mean the same thing

Never change: numbers/quantitative claims, company names, dates, technologies
not already in `base.tex`, job titles.

**Edit budget:** total line-delta across all three zones must stay under 10%
of the file's line count. Eyeball estimate is sufficient. If you find
yourself rewriting more than a few lines per zone, pull back — prefer
minimal, targeted rewordings.

### 5b — Compile

```bash
cd "<output_dir>" && tectonic --keep-logs resume.tex 2>tectonic.stderr
```

Verify:

```bash
[ -s "<output_dir>/resume.pdf" ] && echo OK || echo FAIL
```

On `FAIL` or non-zero exit:
1. Read `tectonic.stderr`.
2. Revert: `cp "<workspace>/resume/base.tex" "<output_dir>/resume.tex"`
3. Retry once editing SUMMARY zone only (narrowest possible scope).
4. Compile again.
5. If still failing: write `<output_dir>/tailor-error.md` with the tectonic
   error output. Set `stop_reason = "compile_failed"`. Break the loop.
   Exit non-zero with `ERROR: tectonic compile failed twice; see tailor-error.md`.

### 5c — ATS score

Read `<output_dir>/resume.tex` as plain text. Compute a score in 0–100:

**Keyword match (40 points max):**
Count how many items from `jd_required` appear (as substrings, case-insensitive)
in the resume text. Score = `(matched / total_jd_required) * 40`.
If `jd_required` is empty, award full 40 points.

**Preferred match (20 points max):**
Same method for `jd_preferred`. Score = `(matched / total_jd_preferred) * 20`.
If `jd_preferred` is empty, award full 20 points.

**Section presence (15 points max):**
Award 3.75 points each for the presence of these LaTeX section markers in
`resume.tex`: `\section*{Summary}`, `\section*{Skills}`,
`\section*{Experience}`, `\section*{Education}`. (4 × 3.75 = 15)

**Quantification (15 points max):**
Count `\item` lines in the resume. Count how many contain at least one digit.
Score = `(quantified_bullets / total_bullets) * 15`.
If no bullets, award 0.

**Format safety (10 points max):**
Award 10 points if `resume.tex` contains none of: `\begin{tabular}`,
`\begin{multicols}`, `\includegraphics`. Award 0 if any are found.

`total_score = sum of all five components`, rounded to integer, clamped to [0, 100].

Compute `gap_report`: list of items from `jd_required` + `jd_preferred` that
were NOT found (case-insensitive substring) in `resume.tex`. This becomes
`last_gap_report` for the next iteration.

### 5d — Accept or revert

If `total_score > best_score`:
- Set `best_score = total_score`
- Set `best_tex` = current contents of `<output_dir>/resume.tex`
- Set `best_iteration = i`
- Append `total_score` to `scores`
- Continue to iteration `i+1` (unless `i == 5`, then stop)

If `total_score <= best_score`:
- Do NOT update `best_score` or `best_tex`
- Append `best_score` to `scores` (no improvement marker)
- Set `stop_reason = "no_improvement"`
- Break the loop
```

- [ ] **Step 2: Verify the append**

```bash
grep -n "Step 5" resume-tailor/agents/rt-resume-tailor.md
```

Expected: a line like `## Step 5 — Iterative tailor loop (max 5 iterations)`

- [ ] **Step 3: Commit**

```bash
git add resume-tailor/agents/rt-resume-tailor.md
git commit -m "feat(resume-tailor): add agent step 5 - iterative tailor loop with ATS scoring"
```

---

### Task 4: Write `rt-resume-tailor` agent — Step 6 (outputs) and hard constraints

**Files:**
- Modify: `resume-tailor/agents/rt-resume-tailor.md` (append Step 6 + constraints + output declaration)

- [ ] **Step 1: Append Step 6, hard constraints, and output section**

Append the following to `resume-tailor/agents/rt-resume-tailor.md`:

```markdown

## Step 6 — Write final outputs

Write `best_tex` back to `<output_dir>/resume.tex`:
Use the Write tool to write `best_tex` content to `<output_dir>/resume.tex`.

Compile the final PDF:

```bash
cd "<output_dir>" && tectonic --keep-logs resume.tex 2>tectonic.stderr
```

Generate the diff:

```bash
diff -u "<workspace>/resume/base.tex" "<output_dir>/resume.tex" > /tmp/rt-diff.$$
```

Read `/tmp/rt-diff.$$` for embedding.

### `tailor-diff.md`

Write `<output_dir>/tailor-diff.md`:

```
# Tailoring diff

**Zones touched:** SUMMARY, SKILLS, BULLETS
**Iterations run:** <len(scores)> (best at iteration <best_iteration>)
**Final ATS score:** <best_score>/100
**Tectonic compile:** OK

## Diff

```diff
<contents of /tmp/rt-diff.$$>
```
```

### `ats-report.md`

Compute the five category sub-scores for `best_tex` (same formulas as Step 5c).
Write `<output_dir>/ats-report.md`:

```
# ATS Score Report

**Score:** <best_score>/100
**Iterations:** <scores> (<stop_reason description>)

## Category Breakdown

| Category | Score | Max |
|---|---|---|
| Keyword match | <kw_score>/40 | 40 |
| Preferred match | <pref_score>/20 | 20 |
| Section presence | <sec_score>/15 | 15 |
| Quantification | <quant_score>/15 | 15 |
| Format safety | <fmt_score>/10 | 10 |

## Remaining Gaps

**Required keywords not matched:** <comma-separated list, or "none">
**Preferred keywords not matched:** <comma-separated list, or "none">
```

Stop reason descriptions:
- `"max_iterations"` → "completed 5 iterations"
- `"no_improvement"` → "stopped early — no improvement at iteration <i>"
- `"compile_failed"` → "stopped early — tectonic compile failed"

### `ats-iterations.json`

Write `<output_dir>/ats-iterations.json`:

```json
{
  "schema_version": 1,
  "scores": <scores array>,
  "best_iteration": <best_iteration>,
  "final_score": <best_score>,
  "stop_reason": "<stop_reason>"
}
```

### `suggestions.md`

Produce suggestions for content the user could add to `base.tex` to improve
future runs. Write `<output_dir>/suggestions.md`:

```
# Suggestions for improving future runs

These are things you could add to `resume/base.tex` to raise your ATS score.
This agent never writes these into your resume — they are yours to consider.

## Missing required skills (in JD but not in your resume)
<for each item in unmet_required:>
- **<skill>**: Listed as required. <one sentence on where in base.tex to add it and what to write, grounded in the actual JD phrasing>

## Missing preferred skills
<for each item in jd_preferred not found in best_tex:>
- **<skill>**: Nice-to-have. <one sentence on how to add it if the user has any relevant experience>

## Portfolio-derived suggestions
<if website cache extracted projects not represented in base.tex:>
- Consider adding a **Projects** section. Your portfolio at <portfolio_url> lists
  projects (<project names>) that aren't in your resume — these could strengthen
  the application.
<if no portfolio_url or all projects already represented:>
(none)

## Phrasing (outside TAILOR zones — manual edits only)
<1-3 specific bullets outside TAILOR zones in base.tex that lack quantification
or JD keyword alignment, with concrete rewording suggestions>
```

If all four sections would be empty, write a single line:
`No suggestions — your resume is a strong match for this JD.`

## Hard constraints

- **Never edit outside TAILOR zone markers.** Preamble, `\usepackage`, section
  headers (`\section*{...}`), company names, job titles, dates, education —
  all immutable.
- **Never edit the sentinel comment lines themselves.** Their text is sacred.
  Do not rename zones, add zones, remove zones, or move sentinels.
- **Edit budget:** total line-delta across all zones ≤ 10% of file line count.
- **Never invent.** No skills, technologies, certifications, or quantitative
  claims that are not in `base.tex` or `website-cache extracted.skills/projects`.
- **Never touch `<workspace>/resume/base.tex`.** It is the canonical source.
- **Never write outside `<output_dir>`** (except: reading `base.tex`,
  reading/writing `<workspace>/.resume-tailor/website-cache.json`,
  reading `<workspace>/preferences.yaml`).
- **Bash whitelist:** `cp`, `mkdir`, `rm -f`, `cd`, `tectonic`, `diff`,
  `shasum`, `sha256sum`, `[ -s ... ]`, `echo`. Use the Edit tool (not `sed`
  or `awk`) for all resume edits.
- **A failure is local.** Always exit non-zero on permanent failure so the
  caller can record it. Never crash the caller.

## Output

Print to stdout exactly one of:

- `OK <output_dir>/resume.pdf score=<best_score>/100 iterations=<len(scores)>`
- `SKIPPED: already tailored`
- `ERROR: <reason>` (exit non-zero; `tailor-error.md` has diagnostics)
```

- [ ] **Step 2: Verify the file ends with the output section**

```bash
tail -10 resume-tailor/agents/rt-resume-tailor.md
```

Expected: the last lines include `- `OK ...`` and `- `ERROR: ...``.

- [ ] **Step 3: Commit**

```bash
git add resume-tailor/agents/rt-resume-tailor.md
git commit -m "feat(resume-tailor): add agent step 6 (outputs) and hard constraints"
```

---

### Task 5: Write the `/resume-tailor` skill

**Files:**
- Create: `resume-tailor/skills/resume-tailor/SKILL.md`

- [ ] **Step 1: Create `resume-tailor/skills/resume-tailor/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Verify the file**

```bash
head -6 resume-tailor/skills/resume-tailor/SKILL.md
```

Expected: the frontmatter with `name: resume-tailor`.

- [ ] **Step 3: Commit**

```bash
git add resume-tailor/skills/resume-tailor/SKILL.md
git commit -m "feat(resume-tailor): add /resume-tailor public skill"
```

---

### Task 6: Update `preferences.yaml` template to include `portfolio_url`

**Files:**
- Modify: `templates/preferences.yaml`

The agent reads `portfolio_url` from the user's `preferences.yaml`. The template must document this field so users know to set it.

- [ ] **Step 1: Add `portfolio_url` to `templates/preferences.yaml`**

Read `templates/preferences.yaml`. Add the following block after the existing content (before the `scoring:` section, as it's a general profile field):

Find this line:

```yaml
visa_sponsorship_required: false
```

Add after it:

```yaml

# Your portfolio/personal website URL. resume-tailor fetches this on each run
# (cached by content hash) to supplement base.tex with projects and skills
# not in your resume. Leave empty to skip.
portfolio_url: ""
```

- [ ] **Step 2: Verify**

```bash
grep -A3 "portfolio_url" templates/preferences.yaml
```

Expected output includes the comment and `portfolio_url: ""`.

- [ ] **Step 3: Commit**

```bash
git add templates/preferences.yaml
git commit -m "feat(resume-tailor): add portfolio_url field to preferences.yaml template"
```

---

### Task 7: Update `linkedin-job-hunt` to delegate to `rt-resume-tailor`

**Files:**
- Modify: `skills/job-hunt/SKILL.md` (R4 step 1)
- Delete: `agents/jh-resume-tailor.md`

- [ ] **Step 1: Read the current R4 section of `skills/job-hunt/SKILL.md`**

Read `skills/job-hunt/SKILL.md` and locate the `## R4 — tailor + save a single approved job` section. The current step 1 reads:

```
1. **Tailor.** Dispatch `jh-resume-tailor`:
   > "Run the procedure in `agents/jh-resume-tailor.md`. Workspace:
   > `<workspace>`. Job id: `<job_id>`. Job folder: `<job_folder>`.
   > Scored entry: `<contents of <job_folder>/fit.json>`."
   The tailor writes `resume.tex` / `resume.pdf` / `tailor-diff.md` INTO
   `<job_folder>`. `OK`/`SKIPPED` → continue; `ERROR` → tell the user the
   tailor failed, ask whether to save anyway or skip. A tailor failure
   never aborts the loop.
```

- [ ] **Step 2: Replace R4 step 1 with the delegation to `rt-resume-tailor`**

Replace the exact text found in Step 1 above with:

```
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
```

- [ ] **Step 3: Verify the edit**

```bash
grep -A 15 "R4 — tailor" skills/job-hunt/SKILL.md | head -20
```

Expected: the new delegation text referencing `resume-tailor/agents/rt-resume-tailor.md`.

- [ ] **Step 4: Delete `agents/jh-resume-tailor.md`**

```bash
git rm agents/jh-resume-tailor.md
```

- [ ] **Step 5: Verify deletion**

```bash
ls agents/
```

Expected: `jh-resume-tailor.md` is absent. All other agent files remain.

- [ ] **Step 6: Commit**

```bash
git add skills/job-hunt/SKILL.md
git commit -m "feat(linkedin-job-hunt): delegate R4 tailoring to resume-tailor plugin, remove jh-resume-tailor"
```

---

### Task 8: Smoke test the plugin structure

**Files:**
- No new files — validation only.

- [ ] **Step 1: Verify the complete `resume-tailor` plugin file tree**

```bash
find resume-tailor -type f | sort
```

Expected output:

```
resume-tailor/.claude-plugin/plugin.json
resume-tailor/agents/rt-resume-tailor.md
resume-tailor/skills/resume-tailor/SKILL.md
```

- [ ] **Step 2: Verify `marketplace.json` has two plugins**

```bash
cat .claude-plugin/marketplace.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d['plugins']), 'plugins:', [p['name'] for p in d['plugins']])"
```

Expected: `2 plugins: ['linkedin-job-hunt', 'resume-tailor']`

- [ ] **Step 3: Verify `jh-resume-tailor.md` is gone**

```bash
ls agents/ | grep resume-tailor
```

Expected: no output (file deleted).

- [ ] **Step 4: Verify `rt-resume-tailor.md` has all 6 steps**

```bash
grep "^## Step" resume-tailor/agents/rt-resume-tailor.md
```

Expected:

```
## Step 0 — Idempotency check
## Step 1 — Profile `base.tex`
## Step 2 — Fetch and cache portfolio website
## Step 3 — Extract JD signals
## Step 4 — Compute match
## Step 5 — Iterative tailor loop (max 5 iterations)
## Step 6 — Write final outputs
```

- [ ] **Step 5: Verify SKILL.md has the P0 preflight**

```bash
grep "P0" resume-tailor/skills/resume-tailor/SKILL.md
```

Expected: `## P0 — Preflight`

- [ ] **Step 6: Verify `preferences.yaml` template has `portfolio_url`**

```bash
grep "portfolio_url" templates/preferences.yaml
```

Expected: the field definition line.

- [ ] **Step 7: Verify job-hunt skill no longer references `jh-resume-tailor`**

```bash
grep "jh-resume-tailor" skills/job-hunt/SKILL.md
```

Expected: no output.

- [ ] **Step 8: Final commit**

```bash
git add .
git status
```

Confirm there are no untracked or modified files. If clean:

```bash
git log --oneline -6
```

Expected: 6 commits from Tasks 1–7 visible in the log.
