# job-search Step 8 — Referral Messages + File Output Design

**Date:** 2026-08-02  
**Status:** Approved

## Summary

Add Step 8 to the `job-search` skill that (a) writes structured per-company output folders for all verified results and (b) drafts referral messages (one-line pitch + cold DM + email template) for every company in Table 2 ("Get a Referral First"). Triggered on user request via the Step 7 closing offer.

## What changes

**One file modified:** `job-search/skills/job-search/SKILL.md`

1. **Update Step 7 closing offer** — make option (b) explicit about what it produces.
2. **Add Step 8** — file-write loop + referral message drafting.

## Output folder structure

Root: `~/job-search-results/<YYYY-MM-DD>/`

The date subfolder keeps runs separate so re-runs don't overwrite prior results.

```
~/job-search-results/
  2026-08-02/
    stripe/
      overview.md          # role, comp, fit, open status, apply link, route
      referral-pitch.md    # one-line pitch + cold DM + email template
    google/
      overview.md
      referral-pitch.md
    thoughtworks/          # Table 1 company — overview.md only
      overview.md
```

- **Table 2 companies** (referral track): get both `overview.md` and `referral-pitch.md`
- **Table 1 companies** (apply directly): get `overview.md` only
- Folder name: company display name lowercased, spaces/special chars replaced with `-` (e.g. `google-deepmind`, `jp-morgan`)

## File contents

### `overview.md`

```markdown
# <Company> — <Role Title>

- **Route:** Apply directly | Get a referral first
- **Role:** <roleTitle>
- **Comp:** <compVerdict range + source>
- **Fit:** <matchStrength>
- **Open:** <openStatus>
- **Apply:** <bestLink or "none verified">
```

### `referral-pitch.md` (Table 2 companies only)

```markdown
# Referral Messages — <Company>

## One-line pitch
"<YoE> YoE <roleType> engineer (<core stack>) with <domain> experience,
targeting the <roleTitle> role at <Company> — happy to connect if you
know anyone on the hiring side."

## Cold DM (LinkedIn / Slack)
Hi [Name], I came across <Company>'s <roleTitle> opening and think my
background in <domain> (<YoE> YoE, <stack>, <notable signal>) is a strong
fit. Would you be open to referring me or passing my resume to the team?
Happy to share more context — no pressure at all.

## Email template
Subject: Referral request — <roleTitle>, <Company>

Hi [Name],
I hope you're well. I'm currently exploring new opportunities and noticed
<Company> is hiring for a <roleTitle> role. Given my <YoE> years building
<domain> systems in <stack>, I believe I'd be a strong fit.

Would you be comfortable referring me or forwarding my resume to the
hiring team? I've attached a one-pager for context.

Thanks so much — I really appreciate it.
[Your name]
```

All `<placeholders>` are filled from the verified company record + candidate profile already in context. `[Name]` stays as-is — the user fills in their actual contact.

## Step 7 closing offer update

**Before:**
> "Want me to (a) verify the remaining companies, (b) draft tailored referral-request messages / a one-line pitch for the referral-track companies, or (c) re-run with different filters?"

**After:**
> "Want me to (a) verify the remaining companies, (b) save all results to `~/job-search-results/` and draft referral messages (one-line pitch + cold DM + email template) for every company in Table 2, or (c) re-run with different filters?"

## Step 8 logic

Triggered when user selects option (b) from the Step 7 closing offer.

1. Resolve the output directory: `~/job-search-results/<YYYY-MM-DD>/` using today's date from `date +%Y-%m-%d`.
2. `mkdir -p` the date directory.
3. For each company in **Table 1** (apply directly):
   - Slugify the company name.
   - `mkdir -p <slug>/`
   - Write `<slug>/overview.md` (role, comp, fit, open, link, route).
4. For each company in **Table 2** (referral track):
   - Slugify the company name.
   - `mkdir -p <slug>/`
   - Write `<slug>/overview.md`.
   - Write `<slug>/referral-pitch.md` (one-line pitch + cold DM + email template, all pre-filled from candidate profile + verified record).
5. Print a summary: folder path, count of companies written, count with referral messages.
6. Offer: "Want me to (a) verify remaining unverified companies or (b) re-run with different filters?"

## Slugify rule

```
slug = company_name.lower()
       .replace(" ", "-")
       .replace(".", "")
       .replace(",", "")
       .replace("/", "-")
       .replace("&", "and")
```

Examples: `Stripe` → `stripe`, `Google DeepMind` → `google-deepmind`, `J.P. Morgan` → `jp-morgan`, `AT&T` → `atat` → `at-and-t`... simplify: just lowercase + replace non-alphanumeric runs with `-`.

## What is NOT changing

- No new agents or files — Step 8 runs inline in the orchestrator.
- No changes to agent files (`job-discover.md`, `job-verify.md`).
- No changes to `reference/methodology.md`.
- No changes to `plugin.json` or `marketplace.json`.
- The Bash tool is already in `allowed-tools` (via the existing `date +%s` timing step) — `mkdir` and the Write tool cover the file operations.
