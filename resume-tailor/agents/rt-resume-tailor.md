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
