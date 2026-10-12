# Orchestration

[← Back to overview](../README.md)

<img src="../assets/orchestration.svg" alt="Planner, executors, verify gate and reporter" width="100%">

## Why roles

The first version was a single headless Claude session with a long prompt: read everything, decide what matters, edit whatever repos need it, rewrite the report. It worked until the work got bigger. A session that has just spent its budget editing four repos is bad at writing a clear summary, and a session full of tool output is worse at judgment.

`agent_orchestrator.py` splits it up:

1. **Planner.** One fresh session a day on the strongest model. No tools. It reads a bounded context packet and writes a queue of tasks, each with a size, a repo, a change class, acceptance checks and a model assignment.
2. **Executors.** One session per task, at most a configured number per run. Small tasks go to a cheap fast model and medium tasks to a stronger one. Large tasks are not attempted. They become PRD drafts or sit in the build queue for me.
3. **Reporter.** A cheap session that writes up what shipped, what's blocked and why. If every provider fails, a plain-code fallback writes the section.

Every role has a provider order with automatic fallback. When one provider hits a session limit, the next one is tried:

```json
"planner":        { "provider_order": ["claude", "antigravity"] },
"executor_small": { "provider_order": ["antigravity", "claude"] },
"executor_medium":{ "provider_order": ["antigravity", "claude"] },
"reporter":       { "provider_order": ["antigravity", "claude"] }
```

Executors and the reporter go to Gemini first so the Claude subscription budget is saved for planning.

## The context packet

The planner never sees raw data dumps. It gets a packet built from named sections, each with its own character budget, freshest signal first:

```python
sections = [
    ("TODAY'S BASELINE REPORT", _trimmed(REPORTS / f"{run_date}.md", 9_000), 9_000),
    ("SEO/GSC REGRESSION CHECK", _gsc_alerts(run_date), 1_500),
    ("FUNNEL (North-Star)",      _trimmed(OUT / run_date / "funnel.json", 3_500), 3_500),
    ("SENTRY (24h)",             _trimmed(OUT / run_date / "sentry.json", 2_500), 2_500),
    # ...
    ("FOUNDER STEERING (overrides your defaults)", _trimmed(DOPS / "steering.md", 2_500), 2_500),
    ("ACQUISITION: demand we have not captured yet", _trimmed(OUT / run_date / "gsc.json", 3_000), 3_000),
    ("APPROVED PRD SLICES: build these",           _prd_slices(), 3_000),
    ("OPEN DECISIONS: awaiting the founder, do NOT re-raise", _decision_lines({"open", "snoozed"}, 15), 1_800),
    ("RECENT LEARNINGS", _trimmed(DOPS / "learnings.md", 2_500, tail=True), 2_500),
    # ...
]
```

A test asserts that the budgets add up to less than the overall cap. Before that test existed, one oversized section quietly pushed the funnel, Sentry and revenue data out of the packet, and the planner spent days planning without them.

The acquisition section came from a different blind spot. Every other input describes people who already arrived, so the planner could only ever find conversion problems, and that's all it found: about half of its first 60 roadmap items were CTA and event tweaks. Search Console is the one source that shows demand the product hasn't won yet.

## The queue

The planner's output is written to `out/<date>/agent_queue.json` before any executor starts. If the run crashes, or a provider runs out mid-run, a retry resumes that queue. It doesn't plan again and doesn't redo finished tasks. Tasks that can't run today (repo already used once today, provider exhausted) carry forward to tomorrow, up to a cap.

## Executors work in worktrees

Each executor gets a throwaway `git worktree` built from `origin/main` in a temp directory:

- The production checkout is never touched or inspected. Docker builds from that directory, and the running services rewrite tracked files in it, so it is never clean.
- Dependencies (`node_modules`, `venv`) are symlinked in from production so tests run without a reinstall. That's only valid while manifests are unchanged, which is one reason a diff that touches `package.json` or `requirements.txt` is blocked from auto-merge.
- The executor's only git command is `git commit`. Branch creation, verification, pushing and merging belong to the orchestrator.

When a task is done, the orchestrator hands the worktree to the [gate](guardrails.md).

## Founder questions

A task the planner can't settle without me (for example, a PRD slice that depends on a pricing answer) doesn't guess. It returns `blocked` with the question, and the question shows up in the PRDs tab. When I answer, the answer goes back into the spec and outranks the agent's own wording on the next run. An unanswered question reaches the executor as *do not guess this*.
