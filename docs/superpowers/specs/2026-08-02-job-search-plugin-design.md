# job-search Plugin Design

**Date:** 2026-08-02  
**Status:** Approved

## Summary

Add the `job-search` folder as a new standalone plugin in the `harsha-claude-marketplace` repo. It is a public-web, zero-auth job search skill — complementary to but independent of `linkedin-job-hunt`.

## Plugin directory layout

```
./job-search/
  .claude-plugin/
    plugin.json              # name: job-search, version: 0.1.0
  agents/
    job-discover.md          # discovery worker (bucket fan-out via WebSearch)
    job-verify.md            # verification worker (per-company comp/link/route check)
  skills/
    job-search/
      SKILL.md               # interactive orchestrator (/job-search)
  reference/
    methodology.md           # referral-vs-direct routing heuristics (read at runtime by SKILL.md)
```

`sample-run-bangalore.md` is not copied — illustrative only, no runtime value.

## Source mapping

| Destination | Source |
|---|---|
| `job-search/.claude-plugin/plugin.json` | New file |
| `job-search/agents/job-discover.md` | `../job-search/agents/job-discover.md` |
| `job-search/agents/job-verify.md` | `../job-search/agents/job-verify.md` |
| `job-search/skills/job-search/SKILL.md` | `../job-search/SKILL.md` |
| `job-search/reference/methodology.md` | `../job-search/reference/methodology.md` |

## plugin.json content

```json
{
  "name": "job-search",
  "version": "0.1.0",
  "description": "Web-only job search skill. Reads any resume (PDF/DOCX), asks a few preference questions, fans out parallel discovery workers across company buckets, verifies comp/links/routes per company, and renders two ranked tables: Apply Directly vs Get a Referral First. No LinkedIn auth or workspace required.",
  "author": {
    "name": "harshavardhanmadhamanchi"
  },
  "skills": "./skills/"
}
```

## marketplace.json update

Add a third entry to the `plugins` array:

```json
{
  "name": "job-search",
  "description": "Web-only job search skill. Reads any resume (PDF/DOCX), asks a few preference questions, fans out parallel discovery workers across company buckets, verifies comp/links/routes per company, and renders two ranked tables: Apply Directly vs Get a Referral First. No LinkedIn auth or workspace required.",
  "source": "./job-search/",
  "version": "0.1.0"
}
```

## Path references inside files

The skill reads `agents/job-discover.md`, `agents/job-verify.md`, and `reference/methodology.md` via relative paths from the skill's own directory. The layout above preserves these relative paths — no edits to file content needed.

## Cross-plugin relationship

The two job-related plugins are independent — no cross-references added. Comparison:

| | `job-search` | `linkedin-job-hunt` |
|---|---|---|
| Auth | None | LinkedIn MCP + login |
| Resume input | Any PDF/DOCX | LaTeX `base.tex` |
| Output | Research tables (in-conversation) | Per-job folders + LinkedIn saves |
| Tailoring | None | Full LaTeX + ATS + PDF |
| Workspace | None | `preferences.yaml` + `target-companies.yaml` |

## What is NOT changing

- No edits to any existing plugin files (`linkedin-job-hunt`, `resume-tailor`)
- No changes to agent/skill content in the source files
- `sample-run-bangalore.md` is excluded
