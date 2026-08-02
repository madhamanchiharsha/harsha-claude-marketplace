# harsha-claude-marketplace

A personal Claude Code plugin marketplace. Three plugins for job hunting and resume tailoring.

| Plugin | Skill | What it does |
|---|---|---|
| `linkedin-job-hunt` | `/job-hunt` | LinkedIn-authenticated job discovery, fit scoring, resume tailoring, and saving |
| `resume-tailor` | `/resume-tailor` | Tailor a LaTeX resume to any job description with ATS scoring |
| `job-search` | `/job-search` | Web-only job search from any resume — no LinkedIn auth required |

---

## Plugin 1 — linkedin-job-hunt

Multi-agent LinkedIn job hunt. Reads your LaTeX resume and target-company list, searches LinkedIn, scores fit asymmetrically, tailors a per-job PDF, and saves approved jobs to LinkedIn. You stay in control of every LinkedIn write.

### Prerequisites

- **Claude Code** with the **LinkedIn MCP server** configured and authenticated.
- **`tectonic`** on `$PATH` — install from <https://tectonic-typesetting.github.io/> (`brew install tectonic` on macOS).

### Workspace setup

```bash
mkdir my-job-hunt && cd my-job-hunt

cp <plugin-path>/templates/preferences.yaml      preferences.yaml
cp <plugin-path>/templates/target-companies.yaml target-companies.yaml
mkdir resume
cp <plugin-path>/templates/resume.base.tex       resume/base.tex
```

Edit the three files:
- **`resume/base.tex`** — replace placeholder content with your real resume. Keep the `% <<TAILOR-START: ZONE>>` / `% <<TAILOR-END: ZONE>>` sentinels (required zones: `SUMMARY`, `SKILLS`, `BULLETS`).
- **`preferences.yaml`** — location, remote policy, salary floor, scoring thresholds.
- **`target-companies.yaml`** — companies to search and optional per-company keyword filters.

### Running

```
/job-hunt
```

Runs in two human-separated phases:

**Phase: discover** (default, read-only on LinkedIn)
1. Analyzes `resume/base.tex` → structured skills profile
2. Fans out LinkedIn searches across target companies
3. Scores each job asymmetrically (job needs vs. your strengths)
4. Writes per-job folders + `overview.md` + `iteration.json` under `jobs/itr_N/`
5. Never touches your LinkedIn saved-jobs list

```
/job-hunt phase=discover
```

**Phase: review** (human-driven saving)
1. Walks you through discovered jobs one at a time
2. For each job you approve: tailors your resume (bounded tailor zones), compiles PDF via `tectonic`, saves to LinkedIn
3. Appends skipped close-fits to `missing-skills.md`

```
/job-hunt phase=review
/job-hunt phase=review itr=2   # review a specific iteration
```

### Workspace layout

```
my-job-hunt/
├── resume/base.tex                    # canonical LaTeX resume — never auto-edited
├── preferences.yaml
├── target-companies.yaml
├── .job-hunt/skills-profile.json      # cached profile (regenerated when base.tex changes)
├── jobs/
│   └── itr_1/
│       ├── jobs.json
│       ├── scored-jobs.json
│       ├── iteration.json
│       ├── overview.md
│       └── <company-role>/
│           ├── job.json
│           ├── fit.json
│           ├── overview.md
│           ├── description.md
│           ├── resume.tex             # tailored copy (written in review phase)
│           ├── resume.pdf
│           ├── tailor-diff.md
│           └── fit-analysis.md
└── missing-skills.md                  # skill-gap ledger
```

### How tailor zones work

`resume/base.tex` is never modified. The tailor agent edits only between sentinels:

```latex
% <<TAILOR-START: SUMMARY>>
... your summary text ...
% <<TAILOR-END: SUMMARY>>
```

Required zones: `SUMMARY`, `SKILLS`, `BULLETS`. Content outside these zones (preamble, packages, company names, dates, titles) is off-limits.

### Plugin layout

```
./                                     # linkedin-job-hunt plugin root
├── .claude-plugin/plugin.json
├── agents/
│   ├── jh-resume-analyzer.md
│   ├── jh-job-searcher.md
│   ├── jh-fit-scorer.md
│   ├── jh-job-summarizer.md
│   └── jh-linkedin-saver.md
├── skills/
│   └── job-hunt/SKILL.md
└── templates/
    ├── preferences.yaml
    ├── target-companies.yaml
    └── resume.base.tex
```

### Limitations

- **No Easy Apply.** LinkedIn doesn't expose application submission via API — the plugin saves jobs to your saved-jobs page and you apply manually.
- **Tailor zones are prompt-enforced.** Always review `tailor-diff.md` before submitting an application.
- **LinkedIn rate limits.** Heavy queries can hit caps; check `iteration.json` for partial flags.

---

## Plugin 2 — resume-tailor

Standalone LaTeX resume tailoring. Takes any job description, rewrites the bounded tailor zones in `resume/base.tex`, iterates up to 5 rounds to maximize ATS score without degrading it, and compiles a PDF.

### Prerequisites

- **`tectonic`** on `$PATH`.
- A workspace with `resume/base.tex` containing `SUMMARY`, `SKILLS`, and `BULLETS` tailor zones.

### Running

```
/resume-tailor
```

Accepts optional arguments:
```
/resume-tailor jd=~/Downloads/stripe-swe.txt
/resume-tailor jd=~/Downloads/stripe-swe.txt output=~/tailored/
```

Outputs written to the job folder (or `output` path if specified):
- `resume.tex` — tailored LaTeX
- `resume.pdf` — compiled PDF
- `ats-report.md` — ATS score breakdown per iteration
- `ats-iterations.json` — score history
- `tailor-diff.md` — unified diff of changes from base
- `suggestions.md` — manual improvements outside the tailor zones

### Plugin layout

```
resume-tailor/
├── .claude-plugin/plugin.json
├── agents/
│   └── rt-resume-tailor.md
└── skills/
    └── resume-tailor/SKILL.md
```

---

## Plugin 3 — job-search

Web-only job search — no LinkedIn auth, no workspace setup, works with any PDF or DOCX resume. Reads your resume, asks a few preference questions, fans out parallel discovery workers across company buckets, verifies comp/links/routes per company, and renders two ranked tables.

### Prerequisites

None. Works out of the box with any Claude Code session.

### Running

```
/job-search
```

Or point it at your resume directly:
```
/job-search ~/Downloads/my-resume.pdf
```

### What it does

1. **Reads your resume** — extracts stack, domain, YoE, location; confirms with you.
2. **Clarifies filters** — role focus, company types, location/remote, comp floor + currency, timeline.
3. **Fans out parallel discovery workers** — one sub-agent per company bucket (big product, funded startups, GCCs, tier-1 services) plus a live job boards sweep.
4. **Verifies each company** — comp realistic at your YoE? Apply link real and live? Direct apply or get a referral first?
5. **Renders two ranked tables:**
   - **✅ Apply Directly** — self-serve ATS boards, lower-volume companies
   - **🤝 Get a Referral First** — high-volume companies where cold apps get buried
6. **Saves results to disk** (on request) — per-company folders under `$HOME/job-search-results/<date>/`, with `overview.md` for every company and `referral-pitch.md` (one-line pitch + cold DM + email template) for every referral-track company.

### Output folder structure (Step 8, on request)

```
~/job-search-results/
  2026-08-02/
    stripe/
      overview.md          # role, comp, fit, open status, apply link
      referral-pitch.md    # one-line pitch + cold DM + email template
    thoughtworks/
      overview.md          # apply-directly company — overview only
```

### Plugin layout

```
job-search/
├── .claude-plugin/plugin.json
├── agents/
│   ├── job-discover.md    # discovery worker (bucket fan-out via WebSearch)
│   └── job-verify.md      # verification worker (comp, link, route per company)
├── skills/
│   └── job-search/SKILL.md
└── reference/
    └── methodology.md     # referral-vs-direct routing heuristics
```

### Limitations

- **Authenticated portals are invisible.** LinkedIn/Naukri/Instahyre behind login can't be read; works from public careers pages and job boards only.
- **Comp is estimated** from levels.fyi / AmbitionBox / Glassdoor — a sourced range, never an offer.
- **Open status is point-in-time.** Reqs churn weekly — re-run for a fresh view.
- **Read-only.** Never modifies your resume, never applies on your behalf.

---

## Marketplace layout

```
harsha-claude-marketplace/
├── .claude-plugin/
│   ├── plugin.json         # linkedin-job-hunt plugin manifest
│   └── marketplace.json    # registry of all three plugins
├── agents/                 # linkedin-job-hunt agents (jh-*)
├── skills/                 # linkedin-job-hunt skill (/job-hunt)
├── templates/              # workspace bootstrap templates
├── resume-tailor/          # resume-tailor plugin subtree
│   ├── .claude-plugin/
│   ├── agents/             # rt-resume-tailor
│   └── skills/             # /resume-tailor
├── job-search/             # job-search plugin subtree
│   ├── .claude-plugin/
│   ├── agents/             # job-discover, job-verify
│   ├── skills/             # /job-search
│   └── reference/          # methodology.md
└── docs/
    └── superpowers/
        ├── specs/          # design specs
        └── plans/          # implementation plans
```
