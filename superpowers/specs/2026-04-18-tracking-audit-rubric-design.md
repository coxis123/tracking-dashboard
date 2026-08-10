# Tracking Audit Rubric — Design

**Date:** 2026-04-18
**Scope:** Spec 1 of a 3-spec program. Defines the rubric that every future audit (and every new entry point) is measured against.
**Status:** Design — pending user review.

---

## 1. Problem

We run 5 web properties (terryreal.com, relationallife.com, summit.terryreal.com, quiz.terryreal.com, grid.terryreal.com), 39+ entry points (forms, webinars, checkouts), and ~445 ThriveCart products. Tracking is already implemented — server hooks, client SDK, cross-domain identity, dedup, reconciliation — but we lack a single, repeatable way to *prove* that every entry point is:

1. Covered at all (no silent gaps);
2. Emitting correct events with correct properties;
3. Attributed correctly (UTMs, touchpoints, identity stitched);
4. Reconciled to source-of-truth (ThriveCart / WP / Ontraport);
5. Implemented cleanly — not as a patchwork of one-off scripts that drifts from documentation.

The purpose of this spec is **not** to run the audit. It is to define the rubric against which audits are run.

## 2. Program Shape

This is Spec 1 of 3. Do not expand its scope.

| Spec | Purpose | Output |
|---|---|---|
| **1 (this doc)** | Define the rubric | Dimensions, test protocol, registry schema, runner CLI contract, output format |
| 2 | Apply the rubric to every entry point | Per-entry-point reports + master scorecard + ranked gap list |
| 3 | Remediate | Prioritized backlog of fixes, executed |

Everything below is Spec 1.

## 3. Rubric — Two Layers, 18 Dimensions

Every entry point is scored on every applicable dimension. Grading is **Pass / Fail / N/A** — no amber, no partial credit. Subjectivity is the enemy of a rubric.

### Layer 1 — Data Correctness (12)

| # | Dimension | Question it answers |
|---|---|---|
| 1 | Registered | Is this entry point in the registry, or is it a shadow form we don't know about? |
| 2 | Server-side capture | Does an event fire at the backend, independent of the user's browser? |
| 3 | Client-side capture | Does the SDK also fire, so we get browser-side attribution context? |
| 4 | Payload completeness | Are all required properties present and non-null? |
| 5 | Payload correctness | Do property values match source-of-truth (e.g., product_name ≠ bump name)? |
| 6 | Attribution chain | Do UTMs / Touchpoints link correctly to Lead and Purchase? |
| 7 | Identity stitching | Same person recognized across subdomains and ThriveCart? |
| 8 | Dedup | No duplicate events per order_id / per person+form+window? |
| 9 | Reconciliation | Does PostHog count match ThriveCart / WP / Ontraport at ≥99%? |
| 10 | Resilience | Survives page cache, consent gate, redirect, cross-domain hop, mobile? |
| 11 | Freshness | Event arrives in expected window (client <5s, webhook <60s)? |
| 12 | Monitoring/alerting | If this entry point stops firing, are we notified? |

### Layer 2 — Implementation Quality (6)

| # | Dimension | Question it answers |
|---|---|---|
| 13 | Architectural fit | Is the approach still right, or have we reinvented a native platform feature? |
| 14 | Simplicity / YAGNI | Are we capturing or persisting things nothing reads? |
| 15 | Consistency | Same pattern, property names, event names across all 5 sites? |
| 16 | Code quality & DRY | Duplication, dead deploy scripts, stale mu-plugins, case-by-case patches? |
| 17 | Testability | Is there an automated test/monitor that would have caught the last 3 regressions? |
| 18 | Documentation truth | Does FUNNELS.md / TODO.md / memory match reality? |

## 4. Test Protocol

Each dimension has a concrete, operational test. Wherever possible: automated. Manual reserved for judgement calls.

### 4.1 Layer 1 Tests

| # | Dimension | Source of truth | Method | Pass threshold | Auto? |
|---|---|---|---|---|---|
| 1 | Registered | `FUNNELS.md` + `tracking-contract.json` | Enumerate forms on each site (HTML scrape) and product SKUs in ThriveCart; diff against registry | Zero unknowns | Auto (nightly) |
| 2 | Server-side capture | WP form submissions table / ThriveCart transactions API | HogQL: count events with `tracking_layer='server'` vs source-of-truth count over same window | ≥99% of source count | Auto |
| 3 | Client-side capture | Server-side count (as baseline) | HogQL: count events with `tracking_layer='client'` OR `sdk_version` present | ≥75% of server count | Auto |
| 4 | Payload completeness | Event schema in `tracking-contract.json` | HogQL: count events with required properties NULL or empty | 0 missing on required fields | Auto |
| 5 | Payload correctness | ThriveCart / Ontraport / WP raw records | Sample N=20 random events per entry; diff properties vs source-of-truth | 100% match on sampled rows | Semi-auto |
| 6 | Attribution chain | UTMs on landing URL + Touchpoint events | For each Purchase, assert ≥1 Touchpoint exists for that distinct_id with source≠'direct' within 90d | ≥90% of purchases have a non-direct touchpoint | Auto |
| 7 | Identity stitching | Multi-site persons | HogQL: count distinct_ids with events on ≥2 sites; spot-check a known email | Grows over time; 0 identity fragments per known email | Auto |
| 8 | Dedup | One event per order_id; one Lead per (person+form+5min) | HogQL: `GROUP BY order_id HAVING count()>1` | 0 duplicates in rolling 7d | Auto |
| 9 | Reconciliation | ThriveCart API / WP form log / Ontraport API | Existing `monitoring/reconciliation.py` | ≥99% match in 24h rolling | Auto |
| 10 | Resilience | Playwright synthetic paths | Scripted: cache hit, consent denied, mobile UA, cross-domain hop — each fires event | All paths pass | Auto |
| 11 | Freshness | Event `timestamp` vs `ingested_at` (or receipt log) | HogQL: p95 latency per event type | client <5s, webhook <60s | Auto |
| 12 | Monitoring/alerting | Slack alert channel | Quarterly fire drill: temporarily disable a hook; alert within SLA | Alert fires within 15 min | Manual (quarterly) |

### 4.2 Layer 2 Tests

| # | Dimension | Source of truth | Method | Pass threshold | Auto? |
|---|---|---|---|---|---|
| 13 | Architectural fit | PostHog feature set vs our custom code | Checklist review per subsystem: "could this be replaced by a native platform feature?" | Documented decision for each subsystem | Manual |
| 14 | Simplicity / YAGNI | PostHog property usage stats | HogQL: for each captured property, count references in insights/dashboards/exports over 90d | 0 "captured but never read" properties (or explicit keep-reason) | Semi-auto |
| 15 | Consistency | Per-site event samples | Diff property names + event names across 5 sites for same concept | Identical taxonomy | Auto |
| 16 | Code quality & DRY | Repo tree + git log | Files untouched 90d + grep for obvious dupes + unused deploy scripts | 0 dead/duplicate files flagged | Manual (semi-auto scan) |
| 17 | Testability | `tests/` folder | One smoke/reconciliation test per entry-point category exists and is green | Green on main | Auto (CI) |
| 18 | Documentation truth | `FUNNELS.md` vs reality | Generated-from-reality report diffed against committed doc | 0 drift | Auto (nightly regen) |

### 4.3 Threshold Rationale

- **≥99% server reconciliation (#2, #9):** server-side events have no legitimate reason to drop. <1% allows for network blip / race condition on cutover. If <99%, the hook is broken somewhere.
- **≥75% client-side (#3):** industry baseline for SDK coverage after ad blockers, JS errors, fast-navigation aborts. Below this indicates real SDK problem.
- **≥90% non-direct touchpoint (#6):** some purchases legitimately have no traceable source (bookmark, direct traffic). 10% is the accepted dark-traffic ceiling; below 90% means attribution chain is leaking.
- **0 duplicates (#8):** hard requirement. Any dupes indicate a bug that can compound.

## 5. Registry — `tracking-contract.json` Extended

### 5.1 Unit of Audit

An **entry point** is a triple: `(site, locator, expected_event)`.

Examples:
- `(terryreal.com, /betrayal/ Elementor form, Lead+FormSubmit)`
- `(relationallife.com, /the-certification-guide/confirmed/, Lead)`
- `(terryreal.com, ThriveCart product 477, Purchase)`

### 5.2 Registry Entry Schema

```json
{
  "id": "tr-betrayal-form",
  "site": "terryreal.com",
  "type": "elementor_form",
  "locator": {
    "url_pattern": "^/betrayal(-offer|-1|-workshop-test)?/$",
    "form_id": null
  },
  "expected_events": ["FormSubmit", "Lead"],
  "required_properties": {
    "FormSubmit": ["email", "site", "page", "form_type", "utm_source"],
    "Lead": ["email", "site", "page", "utm_source"]
  },
  "source_of_truth": "elementor_pro/forms/new_record WP hook",
  "reconciliation_query": "SELECT count() FROM wp_posts WHERE post_type='e-submissions' AND ...",
  "applies": [1,2,3,4,5,6,7,8,9,10,11,12,14,15,17,18],
  "skip": [13, 16]
}
```

Field notes:
- `applies` / `skip` — explicit list of dimensions that apply to this entry. Dimensions 13 and 16 are *repo-wide*, not per-entry — they're audited once against the whole codebase.
- `reconciliation_query` — opaque string; each `type` gets its own resolver that interprets it (WP SQL / ThriveCart API / Ontraport API).

### 5.3 Registry Types (initial)

| `type` | Source of truth | Example |
|---|---|---|
| `elementor_form` | WP `elementor_pro/forms/new_record` hook / `_elementor_submissions` | Betrayal landing |
| `fluent_form` | `fluentform/submission_inserted` / Fluent submissions table | RLI applications |
| `ontraport_smartform` | Ontraport Contacts API | Cert Guide optin |
| `everwebinar` | `rli_ew_register` log | 5-Common-Mistakes Webinar |
| `thrivecart_product` | ThriveCart transactions API | Product 477 (Betrayal) |
| `grid_quiz` | Grid's own event log (external) | grid.terryreal.com quiz |

### 5.4 Registry Generation

The registry is **generated then curated**, not hand-written:

1. `audit.py generate-registry` scans each site (HTML scrape for forms; ThriveCart API for products; WP DB for form IDs).
2. Output is merged with existing `tracking-contract.json` — new entries added as `status: "discovered"`, removed entries marked `status: "missing"`.
3. Human curates — confirms new entries, removes false positives, sets `required_properties` and `applies` for new types.
4. Nightly re-run flags drift (dimension #1 — Registered).

## 6. Runner — `audit.py` Contract

### 6.1 CLI

```bash
# Full audit across all entry points, last 7 days
python3 audit.py run --window 7d

# Single entry point
python3 audit.py run --entry tr-betrayal-form --window 24h

# Single dimension across everything (e.g., dedup sweep)
python3 audit.py run --dimension 8 --window 7d

# Regenerate registry from live sites
python3 audit.py generate-registry

# Detect drift (registry vs live) — exits non-zero if drift found
python3 audit.py drift-check
```

### 6.2 Dimension Plugin Interface

Each dimension is a Python module under `audit/dimensions/`:

```python
# audit/dimensions/dim_08_dedup.py
def applies_to(entry: Entry) -> bool:
    return 8 in entry.applies

def run(entry: Entry, window: Window) -> Result:
    """Return Result(status='pass'|'fail'|'na', evidence={...}, remediation_hint=str)."""
    ...
```

This makes adding a 19th dimension a self-contained new file, not a diff through the runner.

### 6.3 Dependencies

Reuse existing: `monitoring/reconciliation.py`, PostHog personal API key at `.posthog-key`, ThriveCart API credentials. No new services.

Language: **Python 3**, matching the existing `monitoring/` and `tracking-dashboard.py` stack.

## 7. Output Format — Three Layers

### 7.1 Per-Entry-Point Report (markdown)

`audit/reports/<entry-id>-YYYY-MM-DD.md`, one file per entry point per run. 18 rows, evidence links, remediation hints.

Example excerpt:

```
ENTRY: tr-betrayal-form
Window: 2026-04-11 → 2026-04-18
─────────────────────────────
 1. Registered          PASS
 2. Server capture      PASS  82 FormSubmit events / 82 WP submissions (100%)
 3. Client capture      PASS  69/82 (84.1%)
 4. Payload complete    FAIL  utm_source NULL on 6/82 (7.3%)
                              → HogQL: <link>
                              → Hint: check UTM cookie read in Elementor tracker snippet
 5. Payload correct     PASS  sampled 10, all correct
 6. Attribution chain   FAIL  73% have non-direct touchpoint (<90% threshold)
                              → HogQL: <link>
 7. Identity stitching  PASS
 8. Dedup               PASS
 9. Reconciliation      PASS  82/82 in WP (100%)
10. Resilience          PASS  all 4 synthetic paths pass
11. Freshness           PASS  p95 client 2.1s, webhook n/a
12. Monitoring          N/A   last fire drill: 2026-01-15 — overdue
13. Architectural fit   (repo-wide — see scorecard)
14. Simplicity          PASS  all required properties read by ≥1 insight
15. Consistency         PASS
16. Code quality        (repo-wide — see scorecard)
17. Testability         FAIL  no smoke test for Elementor form regression
18. Doc truth           PASS
─────────────────────────────
SCORE: 12/15 applicable dimensions passing
```

### 7.2 Master Scorecard (markdown + JSON)

`audit/scorecard-YYYY-MM-DD.md` + `audit/scorecard-YYYY-MM-DD.json`. Aggregate view:

- Entry-points × dimensions matrix (one cell per pair)
- Pass rate per dimension (worst dimensions surfaced)
- Pass rate per site
- Ranked list of worst offenders (entries with most fails)
- Deltas vs previous run (new fails, resolved fails, new entries)

### 7.3 Command Centre Tab

`tracking-command-centre` gets a new **Rubric** tab that renders the latest `scorecard-*.json`. Read-only v1 — re-runs happen via scheduled GitHub Action, not from the browser.

## 8. Non-Goals (for Spec 1)

- Running the audit. That's Spec 2.
- Fixing anything the audit surfaces. That's Spec 3.
- Re-architecting the tracking system. Layer 2 only *identifies* misfits; it doesn't fix them.
- UI design for the Command Centre Rubric tab beyond "reads the JSON."
- New data sources — everything reuses PostHog, ThriveCart, WP, Ontraport, and the existing monitoring scripts.

## 9. Open Decisions (to confirm during writing-plans)

- Exact shape of `tracking-contract.json` after extension — breaking change vs additive? Proposal: additive, new fields are optional; old consumers ignore them.
- Where `audit.py` lives in the repo tree — proposal: top-level `audit/` directory, peer to `monitoring/` and `sdk/`.
- How the scheduled run is triggered — proposal: GitHub Action, same pattern as hourly reconciliation.

## 10. Success Criteria for Spec 1

Spec 1 is done when:

1. This design doc is committed.
2. An implementation plan (Spec 1 → writing-plans) exists that produces:
   - The extended registry schema
   - `audit.py run` executing all 18 dimensions
   - One reference report for one entry point, end-to-end
3. A second person (or future-you in 3 months) can read this doc and understand what's being measured, how, and why — without re-deriving it.
