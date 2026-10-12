# Guardrails

[← Back to overview](../README.md)

<img src="../assets/merge-gate.svg" alt="Decision path for auto-merging an agent change" width="100%">

The agents can change six repos. The guardrails decide what reaches production without me. One principle sits under all of them: **a prompt is a request, a check is a guarantee.** Anything that matters is enforced in code against the real diff, not written into a prompt and hoped for.

## Switches

| Control | Where | Effect |
|---|---|---|
| Kill switch | `touch scripts/dailyops/.disabled` | The run does nothing |
| `--dry-run` | flag | No email, no outreach, no branches, no memory writes |
| Outreach | `config.json` **and** `.env` | Both must be on to send anything |
| Code changes | `config.json code_changes.enabled` | Off means the agents can plan but not stage |
| Auto-merge | `config.json code_changes.auto_merge.enabled` | Off means every change waits for review |
| New features | An accepted roadmap item or decision | Required before an agent builds anything new |
| Daily caps | `config.json` | Max emails, max tasks, max DM drafts per run |

## Never touch

Four keywords, matched against every changed path, plus migrations and dependency manifests:

```python
DEPENDENCY_MANIFESTS = (
    "package.json", "package-lock.json", "pnpm-lock.yaml", "yarn.lock",
    "requirements.txt", "pyproject.toml", "poetry.lock", "Dockerfile",
)

def blocked_paths(files: list[str], never_touch: list[str]) -> list[str]:
    """Paths that must never reach main unreviewed.

    Substring matching is deliberately over-eager here: a false positive costs one
    review, a false negative ships an unreviewed change to payments.
    """
    keywords = [k.lower() for k in never_touch if k.strip()]
    hits = []
    for path in files:
        low = path.lower()
        if any(keyword in low for keyword in keywords):
            hits.append(path)
        elif Path(path).name in DEPENDENCY_MANIFESTS:
            hits.append(path)
        elif "/migrations/" in low or low.startswith("migrations/"):
            hits.append(path)
    return hits
```

```json
"never_touch": ["billing", "auth", "schema", "payments"]
```

## Tests are run, not reported

Nothing in these repos runs CI, and the executor's report of its own tests is text the model wrote. Neither counts as evidence. The orchestrator runs each repo's configured verify commands in the worktree and reads the exit codes:

```python
def run_checks(worktree: Path, commands: list[str], timeout: int = 900) -> list[CheckResult]:
    """Run the repo's verify commands for real and report real exit codes."""
    results = []
    for command in commands:
        proc = subprocess.run(command, cwd=worktree, shell=True,
                              capture_output=True, text=True, timeout=timeout)
        results.append(CheckResult(command, proc.returncode == 0, proc.returncode, ...))
        if not results[-1].ok:
            break  # a failed check makes the rest irrelevant and costs minutes
    return results
```

```json
"verify": {
  "frontend": ["npm run typecheck", "node scripts/ui-audit.mjs --changed --strict"],
  "backend":  ["venv/bin/python -m pytest test -q"],
  "admin":    ["venv/bin/python -m pytest -q"],
  "infra":    ["python3 -m pytest scripts/dailyops -q"]
}
```

A repo with no verify command can never auto-merge.

`ui-audit.mjs` is a mechanical check for two design mistakes that reached production once: CSS that uses design tokens that don't exist (the fallback wins quietly, so the page renders in random greys), and the coloured left-border-on-a-tinted-card pattern that marks UI as AI-made. It only judges files the branch touched. The whole app has older findings, and gating on those would fail every task.

## Tiers

| Tier | What | Where it goes |
|---|---|---|
| 1 | `content`, `seo`, `copy`, `data`, `instrumentation`; ≤ 25 files, ≤ 600 lines; gate passes | Merges to `main`, which deploys |
| PRD slice | A slice of a PRD I approved; gate passes | Merges to `staging`, a separate deployment with its own DB and analytics off |
| Everything else | `bugfix`, `feature`, `config`, anything unclassified | Pushes a branch. I open the PR |
| Blocked | never-touch paths, migrations, manifests | Never automated |

Instrumentation is in Tier 1 for a specific reason. A missing event has been this project's most expensive repeated failure: five separate times a metric read zero because nothing was recording it. Adding an event doesn't change anything a user sees. Renaming or removing a live event does break history, though, so that counts as a `data` change and needs review.

## The watchdog

Merging to `main` deploys, so Tier 1 needs a safety net. A separate timer runs every 5 minutes and checks two layers:

1. **Containers:** state and Docker healthcheck.
2. **Endpoints:** real HTTPS requests to the public URLs, through Caddy.

A container that's down gets **one** restart. If it's still broken on the next run, the watchdog stops and emails me, because a restart loop turns a bad deploy into a crash loop that hides the original error.

If a public endpoint starts failing, the watchdog reverts any agent merge from the previous 45 minutes. Each merge is reverted at most once. The revert is a push to `main`, so the deploy webhook rebuilds.

It sends mail through plain `smtplib`, not the backend's email service. A monitor that depends on the thing it monitors is how you hear about an outage from a customer. Alerts are edge-triggered: one email when something breaks and one when it recovers.

## What agents never do

- Merge or deploy anything outside the tiers above
- Push to `main` outside Tier 1 (one founder-approved exception: tested commits to the infra repo)
- Touch billing, auth, schema or payments
- Add a new external integration
- Email anyone outside the capped B2B template
- Resolve a decision, accept a roadmap item or approve a PRD
