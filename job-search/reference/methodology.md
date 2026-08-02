# Job Search — Methodology

This is how the `/job-search` skill turns a resume + a few preferences into two ranked, verified tables. It documents the pipeline, the heuristics, the data sources, and the limits — so results are reproducible and the reasoning is auditable.

## Pipeline

```
resume ─▶ Step 1  Intake & profile (Read tool; PDF/DOCX)
        ─▶ Step 2  Clarify filters (AskUserQuestion): roles, company types,
                   location+remote, comp floor+currency, YoE, timeline, notes
        ─▶ Step 3  Confirm scope
        ─▶ Step 4  DISCOVER  (parallel workers, 1 per company bucket + 1 sweep)
        ─▶ Step 5  Consolidate & dedupe by company
        ─▶ Step 6  VERIFY    (1 worker per unique company: comp, link, route)
        ─▶ Step 7  Classify → two tables + shortlist + caveats
```

Discovery casts wide; verification is the adversarial check that keeps only what holds up. Splitting them means a weak discovery hit (optimistic comp, stale link) gets caught before it reaches the user.

## Company buckets

Buckets are chosen from the user's "company types" answer and tailored to the candidate's **domain** and **location** — never a fixed global list. Typical buckets:

| Bucket | Who | Comp signal | Default apply route |
|---|---|---|---|
| `big-product` | FAANG-tier & large product MNCs | Highest, transparent (levels.fyi) | Referral (high volume) |
| `funded-startup` | Funded scale-ups in the candidate's domain | Strong for mid-level; ESOP-heavy | Direct (self-serve boards) |
| `gcc` | Global capability centres / captives, domain-adjacent | Good, stable | Mixed; banks lean referral |
| `tier1-services` | Premium consultancies that clear the floor | Near-floor to mid; varies a lot | Direct (portal) |
| `sweep` | Public job boards / aggregators | Whatever's live | Direct (public apply link) |

## Referral-vs-direct heuristic

The core classification. For each verified company:

**Get a referral first** when the application is likely to be buried or filtered before a human sees it:
- High-volume big-product companies (thousands of applicants per req).
- Large banks / GCCs with opaque, volume-gated ATS.
- Any company where the only route is a generic global careers portal with no live, specific posting.

**Apply directly** when a clean application is actually read:
- Self-serve boards: **Lever, Greenhouse, Keka, Workday, Workable, Ashby** — these are the intended channel.
- Lower-volume startups / scale-ups / boutiques.
- Any company with a verified live posting that takes direct applications.

When a company is genuinely `both`, lean by applicant volume: high volume → referral; otherwise → direct. A referral never *hurts* — the table just reflects where it's the deciding factor.

## Comp realism

- Sources, in rough order of trust for tech comp: **levels.fyi** (best for product cos / total comp incl. stock) → **AmbitionBox** / **6figr** / **Glassdoor** (good for India and services) → press/funding news.
- Always benchmark at the candidate's **YoE band**, not the company median.
- Output a **range with a unit and a source**, never a single number, never "an offer."
- Verdicts: `realistic` (clears the floor at this YoE), `borderline` (can dip below; confirm at offer), `unlikely` (typically below the floor — usually excluded or flagged).

## Link durability rubric

Prefer the most durable real URL, in this order:
1. `durable-careers-page` — official `/careers` or ATS board that stays valid as reqs rotate. **Best default.**
2. `live-posting-found` — a specific, currently-open posting (even better, but expires).
3. `stale-or-unverified` — discovery link that couldn't be confirmed; replace if possible.
4. `none` — no real URL; output an empty link and say so. **Never fabricate one.**

## Limitations (always disclose)

- **Authenticated portals are invisible.** Logged-in LinkedIn / Naukri / Instahyre results can't be read. Work is from public careers pages + public boards.
- **Reqs churn weekly.** `confirmed-open` reflects the moment of the search. Re-run for a fresh view.
- **Comp is estimated**, aggregated from public data — not a guarantee and not an offer.
- **Coverage is bounded by the bucket lists.** If a relevant company isn't searched, it won't appear — widen company types or add a sweep to broaden.
- **No auto-apply.** The skill never submits an application or contacts anyone; it only produces the map.
