# The daily run

[← Back to overview](../README.md)

<img src="../assets/daily-loop.svg" alt="The eight steps of a daily run" width="100%">

The whole run is one shell script, `run.sh`, fired by a systemd timer at 10:00. It's written so the boring, deterministic parts always finish, even when the model parts don't.

## 1. Trigger

```ini
# allernote-dailyops.timer
[Timer]
OnCalendar=*-*-* 10:00:00 <local tz>
Persistent=true
```

It began as a macOS `launchd` job on my laptop and moved to the VPS in August. The Mac version taught me two things. Background jobs on macOS can't read `~/Desktop` or `~/Documents`, and nothing tells you so: the job exits 126 and writes no log. And a sleeping laptop fires the job on wake, on login and at the scheduled time, which is why the run-once guard below exists.

## 2. Guards

Three checks before anything touches the network:

```bash
# kill switch
if [[ -f "$DIR/.disabled" ]]; then
  echo "kill-switch present (.disabled), exiting without action."; exit 0
fi

# exactly once per day, at or after the scheduled hour
if [[ -f "$DONE_MARKER" ]]; then exit 0; fi
if [[ "$NOW_HHMM" < "$SCHED_HHMM" ]]; then exit 0; fi

# wait (up to ~90s) for outbound HTTPS before calling any API
wait_for_network
```

Every time value in the script comes from one configured timezone (`DAILYOPS_TZ`). The first VPS version compared a UTC clock against a local-time schedule, decided 10:00 was always "too early", and skipped every run. See [lessons](lessons.md#the-run-that-vetoed-itself).

## 3. Sense

Eleven collectors run in sequence. Each one writes JSON to `out/<date>/`. A failing collector logs and the loop moves on.

```bash
for col in collect_admin collect_revenue collect_content collect_social collect_posthog \
           collect_funnel collect_gsc collect_sentry collect_lab collect_deploys collect_ga4; do
  "$PY" "$DIR/$col.py" || echo "($col failed, continuing)"
done
```

| Collector | Reads | Why it's there |
|---|---|---|
| `admin` | The product's own admin API (booted locally for the run) | Signups, scans, onboarding answers, outreach state, inbox messages. Uses the admin API, not raw SQL |
| `revenue` | Read-only revenue endpoint | Paid vs gifted plans, new, cancelled, expiring |
| `funnel` | PostHog | The north-star funnel: visitor, scan, signup, paid |
| `posthog` | PostHog | Engagement and tool usage |
| `gsc` | Search Console | Impressions, CTR and position, plus week-over-week regression alerts |
| `ga4` | GA4 | Acquisition by channel |
| `sentry` | Sentry | Top issues with the latest event, stack frames flagged as *our code* or *vendor code* |
| `deploys` | Deploy logs | Did yesterday's deploys succeed |
| `content` | The video engine's folders | What's rendered, what's posted, what's waiting |
| `social` | Social posting history | What went out where |
| `lab` | Lab health endpoint + PostHog | The separate Lab service, which has its own DB |

The Sentry collector does more than count. For each top issue it pulls the latest event and marks each stack frame as in-app or vendor (asyncpg, SQLAlchemy and `node_modules` don't count as our code). That lets the agent sort "real bug" from "browser extension noise" without guessing.

## 4. Baseline report

`build_report.py` turns the JSON into a markdown digest with no model involved. It's the floor. Whatever happens in later steps, this report exists and gets emailed.

## 5. Outreach

A capped batch of B2B emails to clinics, using the same template and send path as the admin app's manual button. It needs two switches on, one in `config.json` and one in `.env`, and it only sends to verified addresses. Sends are timed to land at 9:00 in each lead's own timezone: the admin app polls every 15 minutes and sends to whoever is at that hour.

Instagram cold DMs can't be sent through Meta's API (it only allows replies), so the run drafts that day's queue and attaches it to the report for me to send by hand.

## 6. Agents

The planner, executors and reporter. This is the only step that uses models. See [orchestration](orchestration.md).

## 7. Email

`send_report.py` emails the report, with the agents' section added if they finished and the baseline alone if they didn't. **"Needs you"** comes first.

## 8. You

The admin portal has a Daily Ops page with tabs for the report history, build queue, PRDs, decisions, roadmap, steering, learnings and input status. What I decide there becomes the context for the next run. See [memory](memory.md).

## Afternoon: content

Video generation is a separate timer. It checks every hour from mid-afternoon and runs once the morning run is at least five hours old, so the two never compete for the same model session window. It renders one new reel and stages it. It never posts. The autoposter has its own schedule.
