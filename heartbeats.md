# Workflow Heartbeats (Healthchecks.io)

Sprint 1, task 1.4. Each cron-driven GitHub Actions workflow pings Healthchecks.io after every run. If a ping is missed (cron broke, repo archived, free-tier quota hit, runner down), Healthchecks.io alerts on its own channel — independent of Slack alerts that depend on the workflow itself running.

## Why a separate channel?

The existing `Alert on failure` Slack steps only fire if the workflow itself runs. They cannot detect:

- A cron that silently stopped triggering (e.g. GitHub disables schedules on long-idle repos).
- A runner outage that prevents any step from executing.
- A missing/rotated secret that fails before the alert step.

Healthchecks.io's "missed heartbeat" alert covers exactly that gap.

## Workflows wired up

| Workflow | Cron | Secret name |
|---|---|---|
| `.github/workflows/smoke-test.yml` | `0 */4 * * *` | `HEALTHCHECKS_PING_SMOKE_ID` |
| `.github/workflows/tracking-e2e.yml` (smoke job) | `0 */4 * * *` | `HEALTHCHECKS_PING_E2E_ID` |
| `.github/workflows/tracking-validation.yml` | `30 * * * *` | `HEALTHCHECKS_PING_VALIDATION_ID` |
| `.github/workflows/lead-reconciliation.yml` | `0 * * * *` | `HEALTHCHECKS_PING_RECONCILIATION_ID` |
| `.github/workflows/tracking-monitor.yml` | `0 * * * *` | `HEALTHCHECKS_PING_MONITOR_ID` |

Each step is gated on `env.HEALTHCHECKS_PING_*_ID != ''` and ends with `|| true`, so:

- **Secret unset** → step is skipped silently. Nothing breaks.
- **Curl fails (network blip, hc-ping.com down)** → step still exits 0. The next cron run will ping; missed-heartbeat alert only fires after the configured grace period.
- **Step always runs (`if: always()`)** → heartbeat fires on success AND on test/validation failure, because the goal is "did the workflow run", not "did it pass". Failure detail still flows through the existing Slack alerts.

## One-time setup (free tier covers all 5)

1. Sign up at <https://healthchecks.io/> — free tier allows 20 checks.
2. Click **"Add Check"** five times. Suggested settings per workflow:

   | Field | smoke-test | tracking-e2e | tracking-validation | lead-reconciliation | tracking-monitor |
   |---|---|---|---|---|---|
   | Name | smoke-test | tracking-e2e (smoke) | tracking-validation | lead-reconciliation | tracking-monitor |
   | Period | 4h | 4h | 1h | 1h | 1h |
   | Grace | 30m | 30m | 15m | 15m | 30m |
   | Schedule (cron) | `0 */4 * * *` | `0 */4 * * *` | `30 * * * *` | `0 * * * *` | `0 * * * *` |

   Using cron schedules (instead of period) lets Healthchecks.io align expectations with the GitHub Actions cron exactly.

3. For each check, copy the **UUID** from the "Ping URL" — the bit after `https://hc-ping.com/`.
4. In the GitHub repo, go to **Settings → Secrets and variables → Actions → New repository secret** and add the five secrets listed above. Each value is the UUID for that workflow.
5. Add an integration in Healthchecks.io to send misses somewhere you'll see them (Slack, email, PagerDuty). The free tier supports email and Slack out of the box.

That is the entire setup. Once the secrets are saved, the next cron run pings; if you ever delete the secret, the step skips silently and nothing breaks.

## Verifying it works

After the first cron run with the secret set:

- Healthchecks.io check shows a green "up" dot with a recent ping timestamp.
- The GitHub Actions run log shows a `Heartbeat ping` step that ran and exited 0.

To force a manual ping for testing without waiting for cron:

```bash
curl -fsS https://hc-ping.com/<your-uuid>
```

To simulate a missed ping, pause the workflow (or temporarily change the cron) and wait past the grace period — Healthchecks.io will deliver a "down" alert through whatever integrations you wired up.

## What heartbeats are NOT

- Not a substitute for the existing Slack `Alert on failure` steps. Those carry failure detail (which test, which assertion). Heartbeats only answer "did anything run at all".
- Not a hard gate on workflow success — `|| true` ensures a Healthchecks.io outage cannot fail a workflow run.
- Not in scope for `workflow_dispatch`-only runs (no cron expectation to miss). Manual runs that have a heartbeat step will still ping; that is harmless and resets the timer.
