# resume-tailor Plugin — Design Spec

**Date:** 2026-08-02
**Status:** Approved

---

## Goal

Extract the resume tailoring capability from `linkedin-job-hunt` into a standalone, publicly invokable `resume-tailor` plugin that:

1. Can be invoked independently via `/resume-tailor` with any job description text or file
2. Is reusable by `linkedin-job-hunt` and any future plugin
3. Fetches and caches a portfolio website to supplement `base.tex`
4. Iteratively improves the tailored resume against an ATS score (up to 5 iterations, never degrading)
5. Produces actionable suggestions for content the user could add to `base.tex`

---

## Repository Structure

```
harsha-claude-marketplace/
├── resume-tailor/                        ← NEW plugin
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── skills/
│   │   └── resume-tailor/
│   │       └── SKILL.md                  ← public skill: /resume-tailor
│   └── agents/
│       └── rt-resume-tailor.md           ← rewriter + scorer agent
│
├── linkedin-job-hunt/
│   ├── agents/
│   │   ├── jh-resume-tailor.md           ← DELETED (replaced by rt-resume-tailor)
│   │   └── ...
│   └── skills/
│       └── job-hunt/
│           └── SKILL.md                  ← R4 updated to delegate to rt-resume-tailor
│
├── .claude-plugin/
│   └── marketplace.json                  ← gains second plugin entry for resume-tailor
└── docs/
    └── superpowers/specs/
        └── 2026-08-02-resume-tailor-plugin-design.md
```

---

## Plugin Manifest (`resume-tailor/.claude-plugin/plugin.json`)

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

---

## Skill: `/resume-tailor` (`SKILL.md`)

### Invocation

```
/resume-tailor                                 # prompts for JD, uses cwd as workspace
/resume-tailor jd=path/to/jd.txt              # reads JD from file
/resume-tailor jd=path/to/jd.txt output=path/ # custom output dir
```

### P0 — Preflight

1. **tectonic present** — `which tectonic`. If missing, stop:
   > ⚠️ `tectonic` is not on your `$PATH`. Install it (`brew install tectonic`), then re-run.

2. **`resume/base.tex` exists** — `<workspace>/resume/base.tex` present and contains all three sentinel pairs:
   `% <<TAILOR-START: SUMMARY>>` / `% <<TAILOR-END: SUMMARY>>`,
   `% <<TAILOR-START: SKILLS>>` / `% <<TAILOR-END: SKILLS>>`,
   `% <<TAILOR-START: BULLETS>>` / `% <<TAILOR-END: BULLETS>>`.
   If missing or malformed, stop with an explicit error.

3. **JD source resolved** — if `jd=` not provided, prompt the user to paste the JD text or provide a file path. Wait for their input before continuing.

### Workspace resolution

- If cwd contains `resume/base.tex`, use cwd as `<workspace>`.
- Otherwise ask the user for the workspace path.

### Output dir resolution

- If `output=` provided, use that path.
- Otherwise default to `<workspace>/tailored/<slug>/` where `<slug>` is a short sanitized name derived from the company/role in the JD (e.g., `stripe-backend-engineer`). If no company/role can be extracted, use `tailored-<YYYYMMDD-HHMMSS>`.

### Dispatch

Dispatch `agents/rt-resume-tailor.md` with:
- `<workspace>`
- `<jd_text>` — the full JD as inline text
- `<output_dir>` — resolved absolute path

Report the result to the user:
- On `OK`: path to `resume.pdf`, `ats-report.md`, `suggestions.md`
- On `ERROR`: the error message and path to any diagnostic files written

---

## Agent: `rt-resume-tailor.md`

### Inputs

Passed by the skill or by the job-hunt orchestrator:

| Field | Required | Description |
|---|---|---|
| `<workspace>` | yes | Absolute path to project root (contains `resume/base.tex`, `preferences.yaml`) |
| `<jd_text>` | yes | Full job description as inline text |
| `<output_dir>` | yes | Absolute path where all outputs are written |
| `<scored_entry>` | no | Pre-computed fit.json fields from job-hunt (skips inline scoring if provided) |

### Step 0 — Idempotency check

If `<output_dir>/resume.pdf` already exists, print `SKIPPED: already tailored` and exit 0.

### Step 1 — Read and profile `base.tex`

Read `<workspace>/resume/base.tex`. Extract inline:
- `core_skills` (4–8): top strengths (appear in summary, skills header, most bullets)
- `secondary_skills` (4–10): supporting tools from older roles or secondary tools
- `tech_stack` (deduplicated): every named technology
- `current_role`: most-recent job title

This mirrors `jh-resume-analyzer` logic but runs inline — no separate agent call, no cache written here.

### Step 2 — Fetch and cache portfolio website

Read `portfolio_url` from `<workspace>/preferences.yaml` (field: `portfolio_url`). If absent, skip this step.

**Cache check:**

```bash
mkdir -p "<workspace>/.resume-tailor"
```

Read `<workspace>/.resume-tailor/website-cache.json` if it exists. It stores:

```json
{
  "url": "https://...",
  "content_hash": "sha256:<hash>",
  "extracted": {
    "summary": "...",
    "projects": ["..."],
    "skills": ["..."],
    "notable_links": ["..."]
  }
}
```

Fetch the website HTML using the `WebFetch` tool. Compute `sha256` of the raw HTML body. Compare against `content_hash`:

- **Match** → use cached `extracted` content, skip re-parse
- **Mismatch or no cache** → extract structured content from the HTML (summary, projects, skills, notable links), write updated `website-cache.json`

Merge `extracted.projects` and `extracted.skills` into the candidate profile from Step 1 (additive — website content supplements, never overrides `base.tex`).

### Step 3 — Extract JD signals

From `<jd_text>` extract (same logic as `jh-fit-scorer`):
- `jd_required`: skills/keywords marked required ("must", "required", "X+ years", "strong proficiency")
- `jd_preferred`: nice-to-haves ("plus", "bonus", "preferred", "nice to have")
- `company`: company name if extractable
- `role`: job title if extractable

If `<scored_entry>` was provided (job-hunt path), use its `jd_required`, `jd_preferred`, `core_required_match`, `secondary_required_match`, `unmet_required` directly — skip Steps 1–3 profile extraction.

### Step 4 — Compute match (inline scoring)

Skip if `<scored_entry>` provided.

- `core_required_match` = `jd_required` ∩ `core_skills` (liberal semantic match)
- `secondary_required_match` = (`jd_required` ∩ `secondary_skills`) − `core_required_match`
- `unmet_required` = `jd_required` − (`core_skills` ∪ `secondary_skills` ∪ website skills)

### Step 5 — Iterative tailor loop (max 5 iterations)

Ensure output dir exists:

```bash
mkdir -p "<output_dir>"
```

Maintain:
- `best_tex` — content of the best `resume.tex` so far (starts as `base.tex` content)
- `best_score` — ATS score of the best version (starts at 0)
- `scores` — list of scores per iteration (for `ats-iterations.json`)
- `stop_reason` — set when loop exits early (`no_improvement`, `compile_failed`)

**Per iteration:**

#### 5a — Copy and edit

```bash
cp "<workspace>/resume/base.tex" "<output_dir>/resume.tex"
```

Edit ONLY inside the three TAILOR zones using the same boundary discipline as `jh-resume-tailor`:

**SUMMARY** (priority 1): Rewrite to lead with `core_required_match` skills and JD domain language. Use vocabulary from `jd_required`. ≤4 sentences. Facts identical.

**SKILLS** (priority 2): Reorder so `jd_required` + `jd_preferred` skills appear earliest. Reword category labels to emphasize JD vocabulary. Never add skills not in `base.tex` or website cache.

**BULLETS** (priority 3): Lightly reword 1–2 bullets to use JD vocabulary. Allowed: synonym substitution, prepending JD terms ("event-driven async pipelines"), appending context ("high-throughput pipeline (50M jobs/day)"). Never change numbers, company names, dates, or technologies.

On iteration > 1: use the ATS gap report from the previous iteration to guide edits — target the specific keywords still missing.

#### 5b — Compile

```bash
cd "<output_dir>" && tectonic --keep-logs resume.tex 2>tectonic.stderr
```

Verify: `[ -s "<output_dir>/resume.pdf" ]`

On compile failure:
1. Read `tectonic.stderr`
2. Revert: `cp "<workspace>/resume/base.tex" "<output_dir>/resume.tex"`
3. Retry once with SUMMARY zone only
4. If second failure: write `tailor-error.md`, exit non-zero with `ERROR: tectonic compile failed twice`

#### 5c — ATS score

Compute ATS score (0–100) by evaluating:

| Category | Weight | How scored |
|---|---|---|
| Keyword match | 40% | `(jd_required keywords found in resume.tex) / len(jd_required)` |
| Preferred match | 20% | `(jd_preferred keywords found) / len(jd_preferred)` |
| Section presence | 15% | Summary, Skills, Experience, Education all present |
| Quantification | 15% | Fraction of bullets containing a number |
| Format safety | 10% | No tables, no multi-column layouts, no graphics (LaTeX plain = safe) |

Score = weighted sum × 100, rounded to integer.

Collect gap report: list of `jd_required` + `jd_preferred` keywords still absent from the current `resume.tex`.

#### 5d — Accept or revert

- If `score > best_score`:
  - `best_score = score`
  - `best_tex = current resume.tex content`
  - Append score to `scores`
  - Continue to next iteration (or stop if iteration 5)
- If `score <= best_score`:
  - Do NOT update `best_tex` or `best_score`
  - Append `best_score` to `scores` (no improvement this round)
  - Set `stop_reason = "no_improvement"`
  - **Break the loop**
- After 5 iterations without breaking: set `stop_reason = "max_iterations"`

Step 6 always writes the final output from `best_tex` — no recompile needed in 5d.

### Step 6 — Write final outputs

Write `best_tex` to `<output_dir>/resume.tex` and compile to produce `resume.pdf`.

#### `tailor-diff.md`

```markdown
# Tailoring diff

**Zones touched:** SUMMARY, SKILLS, BULLETS
**Iterations run:** N (best at iteration K)
**Final ATS score:** NN/100
**Tectonic compile:** OK

## Diff

```diff
<diff -u base.tex resume.tex output>
```
```

#### `ats-report.md`

```markdown
# ATS Score Report

**Score:** NN/100
**Iterations:** [62, 71, 78, 78] (stopped early — no improvement at iteration 4)

## Category Breakdown

| Category | Score | Max |
|---|---|---|
| Keyword match | 32/40 | 40 |
| Preferred match | 14/20 | 20 |
| Section presence | 15/15 | 15 |
| Quantification | 12/15 | 15 |
| Format safety | 10/10 | 10 |

## Remaining Gaps

**Required keywords not matched:** kafka, grpc
**Preferred keywords not matched:** rust
```

#### `ats-iterations.json`

```json
{
  "schema_version": 1,
  "scores": [62, 71, 78, 78],
  "best_iteration": 3,
  "final_score": 78,
  "stop_reason": "no_improvement"
}
```

Stop reasons: `max_iterations`, `no_improvement`, `compile_failed`

#### `suggestions.md`

```markdown
# Suggestions for improving future runs

These are things you could add to `resume/base.tex` to raise your ATS score.
The agent never writes these into your resume — they are yours to consider.

## Missing required skills (in JD but not in your resume)
- **kafka**: Listed as required. If you have Kafka experience, add it to the Skills section and a bullet.
- **grpc**: Listed as required. A sentence about RPC frameworks in your summary or a bullet would help.

## Missing preferred skills
- **rust**: Nice-to-have. If you've used it at all, worth adding to secondary skills.

## Structural suggestions
- A **Projects** section would let you highlight open-source work. Your portfolio site
  (https://madhamanchiharsha.github.io/) lists projects that aren't in your resume.

## Phrasing (outside TAILOR zones — manual edits)
- BigCo bullet "Shipped the company's first GraphQL gateway" could reference the scale
  (users, services) to strengthen quantification.
```

### Hard constraints (identical to `jh-resume-tailor`)

- **Never edit outside TAILOR zone markers.** Preamble, headers, company names, dates, education are immutable.
- **Never edit sentinel lines themselves.**
- **Edit budget:** keep total line-delta under 10% of file line count.
- **Never invent.** No skills, technologies, certifications, or achievements that aren't in `base.tex` or website cache.
- **Never touch `<workspace>/resume/base.tex`.**
- **Never write outside `<output_dir>`** (except reading base.tex and website cache).
- **Bash whitelist:** `cp`, `mkdir`, `rm -f`, `cd`, `tectonic`, `diff`, `shasum`, `sha256sum`, `[ -s ... ]`, `echo`. No `sed`/`awk` for editing — use Edit tool.

### Output

Print exactly one of:
- `OK <output_dir>/resume.pdf score=NN/100 iterations=N`
- `SKIPPED: already tailored`
- `ERROR: <reason>`

---

## `linkedin-job-hunt` Integration Change

In `skills/job-hunt/SKILL.md`, R4 step 1 changes from:

> "Run the procedure in `agents/jh-resume-tailor.md`. ..."

To:

> "Run the procedure in `<resume-tailor-plugin-path>/agents/rt-resume-tailor.md`. Workspace: `<workspace>`. Output dir: `<job_folder>`. JD text: `<contents of job_folder/job.json description field>`. Scored entry: `<contents of job_folder/fit.json>`."

Where `<resume-tailor-plugin-path>` is documented in the skill as the installed path of the `resume-tailor` plugin (e.g., `~/.claude/plugins/cache/harsha-claude-marketplace/resume-tailor/`).

`agents/jh-resume-tailor.md` is deleted.

---

## `marketplace.json` Change

```json
{
  "plugins": [
    { "name": "linkedin-job-hunt", ... },
    {
      "name": "resume-tailor",
      "description": "Standalone LaTeX resume tailoring with ATS scoring. Tailors base.tex against any job description, fetches portfolio website for supplemental context, iterates up to 5 times without degrading ATS score, and writes actionable suggestions.",
      "source": "./resume-tailor/",
      "version": "0.1.0"
    }
  ]
}
```

---

## Data Flow Summary

```
Standalone:
  User → /resume-tailor (jd=...) → SKILL.md → rt-resume-tailor.md
                                                    ├── reads base.tex
                                                    ├── fetches + caches portfolio site
                                                    ├── extracts JD signals
                                                    ├── computes match inline
                                                    └── iterates (max 5): tailor → ATS score → accept/revert
                                                         └── writes resume.pdf, ats-report.md, suggestions.md

Via job-hunt:
  job-hunt R4 → rt-resume-tailor.md (scored_entry=fit.json)
                    ├── reads base.tex
                    ├── fetches + caches portfolio site
                    ├── skips inline scoring (uses pre-scored fit.json)
                    └── iterates (max 5): tailor → ATS score → accept/revert
```

---

## Open Questions (resolved)

- **Website fetching:** live fetch with content-hash cache in `.resume-tailor/website-cache.json`
- **ATS scoring:** local, no external API dependency
- **Max iterations:** 5, stops early on no-improvement
- **Fabrication guard:** hard no — suggestions go to `suggestions.md` only
- **tectonic requirement:** required for both standalone and job-hunt paths (same as today)
