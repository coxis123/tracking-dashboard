# Spec 2a: First Honest Scorecard — Design

**Date:** 2026-04-19
**Status:** Design (pre-implementation plan)
**Predecessor:** Spec 1 (Tracking Audit Rubric) — shipped on `plastic-mood`, 27 tasks, 75 tests passing
**Successors:** Spec 2b (scheduling + Command Centre tab), Spec 2c (remediation of findings)

---

## 1. Goal + Invariants

**Goal:** Produce the first trustworthy scorecard for the tracking system. It must (a) score the top 10 highest-volume entry points on all 18 audit dimensions against real PostHog data, and (b) prove we know about every entry point that exists, with new ones picked up automatically.

**What "trustworthy" means here:**
- Numbers come from live PostHog / ThriveCart / Ontraport / MySQL — no placeholders, no mocks.
- Every entry point is either fully audited (Tier 1) or enumerated and listed as "registered, not yet audited" (Tier 2) — no silent gaps.
- Drift between what scanners find and what the registry lists is surfaced loudly.

**Invariants (hard rules for this spec):**

1. **Zero customer impact.** Read-only adapters. No edits to SDK, mu-plugins, Elementor, forms, or anything in the request path of a live user. Scripts run only on manual invocation.
2. **No production fixes.** Findings become a TODO for Spec 2c. This spec stops at "we have a credible picture."
3. **Real data only.** No mocks outside unit tests. Integration/smoke must hit PostHog.
4. **Completeness is non-negotiable.** Every entry point either appears in the registry or appears in drift output. There is no third category.

---

## 2. Registry + Drift — Two-Tier + Enumerators

### 2.1 Two-tier registry

`tracking-contract.json` grows a `tier` field on each entry:

- **Tier 1 — Fully audited (top 10 highest-volume).** Hand-curated. All 18 dimensions run. PASS / FAIL / N/A per dim in scorecard.
- **Tier 2 — Enumerated but not yet audited.** Every other known entry point. Appears in scorecard under "registered, not yet audited" with its type, site, and locator. Not scored.

Tier 1 top 10 (selection criteria: volume × business value from `FUNNELS.md`):

| # | Entry | Site | Type | Rationale |
|---|---|---|---|---|
| 1 | Certification Guide | relationallife.com | Ontraport SmartForm | ~900 leads/month, largest RLI source |
| 2 | Summit → VIP | summit.terryreal.com | Ontraport SmartForm | ~500/week |
| 3 | 5 Common Mistakes Webinar | relationallife.com | EverWebinar + Elementor | ~200/week |
| 4 | Quiz → Staying in Love | quiz.terryreal.com | React (Grid) | ~750/week |
| 5 | Betrayal Workshop | terryreal.com | Elementor | ~80/week, flagship TR |
| 6 | Couple Therapists Workshop | terryreal.com | Elementor | ~120/week |
| 7 | Getting a Second Session | relationallife.com | Ontraport SmartForm | Upsell funnel |
| 8 | ThriveCart Purchase (all products) | (cross-site) | Webhook | 100% of revenue |
| 9 | Direct Professional Applications | relationallife.com | Fluent Forms | High-value, low-volume |
| 10 | Unstuck Workshop | terryreal.com | Elementor | ~recurring |

### 2.2 Per-type enumerators

Each scanner produces a canonical list of entry points for its type. They run on every `audit run` before dimensions execute.

| Type | Scanner module | Source | Status |
|---|---|---|---|
| Elementor forms | `audit/scanners/html_forms.py` | HTTP GET live sites, parse `<form>` tags | Exists (Spec 1) |
| Fluent Forms | `audit/scanners/fluent_forms.py` | MySQL `wp_fluentform_forms` via SSH | **New** |
| Ontraport SmartForms | `audit/scanners/ontraport_forms.py` | Ontraport API `/Forms` | **New** |
| EverWebinar | `audit/scanners/everwebinar.py` | Grep `rli_ew_register` across mu-plugins | **New** |
| ThriveCart products | `audit/scanners/thrivecart_products.py` | ThriveCart API `/products` | **New** |
| Webhooks | `audit/scanners/webhooks.py` | Grep `add_action` hook registrations | **New** |

All scanners follow the shape established in Spec 1: return `list[DiscoveredEntry]` with `type`, `site`, `locator`, `source` fields.

### 2.3 Drift workflow

`python3 -m audit drift-check` (exists from Spec 1, extended here):

1. Run every scanner. Aggregate into `audit/out/discovered.json`.
2. Diff `discovered.json` vs `tracking-contract.json`:
   - **New entries** (in scanners, not in registry) → emit a suggested JSON patch block to stdout AND append them to registry as Tier 2 with these defaults:
     - `tier: 2`
     - `expected_events` inferred from type (`elementor_form` → `["FormSubmit"]`; `thrivecart_webhook` → `["Purchase"]`; `ontraport_smartform` → `["FormSubmit","Lead"]`; `fluent_form` → `["FormSubmit"]`; `everwebinar_registration` → `["FormSubmit"]`)
     - `required_properties: {}` (empty — dimensions skip with N/A when no props declared)
     - `applies: []`, `skip: []`
     - `source_of_truth` set from scanner type
     - `reconciliation_query: ""`

     Controlled by `--auto-register` flag; default is dry-run (print patch, exit 1).
   - **Removed entries** (in registry, not in scanners) → flag as "disappeared." Could be a genuine removal or a scan failure — human decides.
3. Drift report appears at top of every scorecard. Non-zero drift = loud banner.
4. Exit code 1 if drift found and `--auto-register` not set → fails CI when scheduled in Spec 2b.

### 2.4 What this guarantees

- Known-unknowns eliminated: every entry is in the registry or in drift output.
- New entry points surface within one audit cycle.
- Zero customer impact (all scanners read-only).

---

## 3. Adapters — PostHog + Source-of-Truth Counters

### 3.1 PostHog adapter

Already exists (`audit/adapters/posthog.py`) from Spec 1. Spec 2a requires:

- Confirm `POSTHOG_PROJECT_ID=127361` and personal API key wired via env var.
- Retry + backoff on 429/5xx (simple: 3 retries, exponential).
- Query results cached in memory per audit run (dimensions issue ~5 queries per entry — don't duplicate).

### 3.2 Source-of-truth counters (per entry type)

These answer "how many real events happened in the window?" — Dimension 2 depends on them. Each is read-only.

| Entry type | Source | Access method | Notes |
|---|---|---|---|
| `elementor_form` | WP DB `wp_e_submissions` | SSH + MySQL | Filter by `form_name` / page URL |
| `fluent_form` | WP DB `wp_fluentform_submissions` | SSH + MySQL | Filter by `form_id` |
| `ontraport_smartform` | Ontraport form submission log | Ontraport API `/FormSubmissions` | **Submissions, not contacts.** Accounts for existing leads opting into new funnels. EST → UTC conversion required. |
| `everwebinar_registration` | WP DB `wp_options` rows `rli_ew_register_*` OR dedicated log if one exists | SSH + MySQL | Implementation plan task must first confirm which source is authoritative on live site |
| `thrivecart_webhook` | ThriveCart transactions API | API | Filter by product + date |
| `react_quiz` (Grid) | Grid's own PostHog project | PostHog API | Cross-project query |

Each counter implements the `SourceOfTruth` Protocol (exists in `audit/core/sources.py`). Registered in `SourceRegistry` by `entry.type`.

### 3.3 Ontraport specifics (called out per user feedback)

Ontraport SmartForms are **opt-in events**, not contact-creation events. Existing leads re-opt-in to new funnels regularly. The counter MUST query the form-submission log, not contact-created events. Timezone conversion (EST log → UTC audit window) done in the adapter, not the caller.

---

## 4. Deliverables

A successful Spec 2a run produces:

1. **`audit/reports/<entry-id>.md`** — per-entry 18-dim breakdown for each Tier 1 entry (10 files).
2. **`audit/scorecards/<date>.md`** — consolidated scorecard with:
   - Drift banner (top)
   - Tier 1 rollup: entries sorted worst-first, dim pass rates, score excludes N/A
   - Tier 2 list: every registered-but-unaudited entry, grouped by site
3. **`audit/scorecards/<date>.json`** — machine-readable version of the above.
4. **`audit/out/discovered.json`** — latest enumerator output (for drift diffing + 2b Command Centre).

CLI entry points (all exist from Spec 1, extended here):

- `python3 -m audit run --window 7d` — full pipeline: scan → drift → adapters → dimensions → report
- `python3 -m audit drift-check [--auto-register]` — scan + drift only
- `python3 -m audit generate-registry` — bootstrap registry from scanners (first-time use)

---

## 5. Out of Scope (explicit — these are 2b / 2c / later)

- **No scheduling.** No GitHub Actions, no cron. Manual invocation only. (→ Spec 2b)
- **No Command Centre tab.** JSON is written but nothing consumes it in the UI yet. (→ Spec 2b)
- **No production fixes.** Bad FAILs become a TODO list, not a PR. (→ Spec 2c)
- **No Slack alerts.** (→ Spec 2b)
- **No long-tail entry expansion beyond top 10 Tier 1.** Tier 2 lists everything, but only 10 are scored. (→ Spec 2b or later)
- **No new dimensions.** All 18 are from Spec 1.
- **No Grid app tracking changes.** Read-only cross-project PostHog query only.

---

## 6. Risks + Mitigations

| Risk | Mitigation |
|---|---|
| Ontraport API rate limits | Cache per-run, batch window queries, exponential backoff |
| MySQL SSH queries slow down production | Use read replicas where available; run off-peak; explicit `--limit` clauses |
| ThriveCart product enumerator returns 445 products, most not entry points | Filter to products with sales in last 90d; register the rest as Tier 2 without auditing |
| Scanner false positives (e.g., newsletter signup scraped as workshop form) | Manual review of first drift-check output; scanners err on the side of "report everything, humans triage" |
| Cross-project PostHog (Grid) requires separate API key | Document in README; skip gracefully with N/A if key missing |

---

## 7. Success Criteria

Spec 2a ships when:

1. `python3 -m audit run` executes end-to-end against real PostHog + real sources of truth (no mocks outside unit tests).
2. First scorecard lists all 10 Tier 1 entries with 18 dimensions each, real numbers, real evidence.
3. First scorecard lists ≥30 Tier 2 entries (matches `FUNNELS.md` ~39 entry points).
4. `drift-check` passes on first clean run; re-runs after adding a test entry correctly flag drift.
5. All new code has unit tests; integration test proves the pipeline with one real entry.
6. Zero customer-facing changes (verified by `git diff` — no changes to `sdk/`, `mu-plugins/`, `tracking-command-centre/`).
