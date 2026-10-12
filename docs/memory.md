# Memory

[← Back to overview](../README.md)

<img src="../assets/learning-loop.svg" alt="Files the founder writes and files the agent writes" width="100%">

The agent's memory is a handful of files in git. I can read every one, edit any of them, and see in `git log` how each changed. No vector store and no hidden state.

## Files I write

**`steering.md`** holds standing orders in plain English. The run reads it first, and the context packet marks it as overriding the agent's defaults. I edit it in a text box in the admin portal. Some real lines:

> Never email anyone outside the B2B clinic outreach flow without asking me first.

> Never propose anything that needs ad spend without flagging it as spend.

> A >10% week-over-week drop in Search Console clicks or impressions on money pages is a regression to fix, not "traffic, not the north star" to wave off. Flag it and propose a fix, same as a broken signup flow.

When the agent keeps getting something wrong, the fix is usually one sentence here, not a code change.

## Files the agent writes

**`learnings.md`** holds lessons the agent distilled from its own runs, pruned as they go stale. Format: date, what was tried, what happened, what to do differently. Examples:

> Staged branches are depreciating fast. A fix that isn't pushed is worth zero. Surface a "days since staged" counter on each branch in the report.

> Discovery source tracking IS working. The earlier "100% None" observation was stale. Remove that hypothesis.

The second one matters as much as the first. The agent has to be able to retract its own lessons, or old wrong conclusions pile up.

**`actions.jsonl`** is an append-only log of every action: sends, branches, merges, reverts, raised decisions. `learnings.md` is the readable summary. This file is the record.

## Decisions

The report's "Needs you" section used to be prose, and prose can't be tracked. Now each item is a decision with a stable key:

```python
def add_decision(key, title, detail="", recommend="", source="other"):
    """Idempotently raise a founder action item, keyed by a stable key
    (e.g. sentry:<issue_id>, content:post-reel:<slug>, flag:outreach-live):
    - key already resolved  -> return None (respect the founder's decision; no re-nag)
    - key open/snoozed      -> refresh its evidence text, keep status
    - new key               -> create an open item
    """
```

The agent can only **raise** decisions. I resolve them in the admin portal: accept, done, dismiss, snooze or reject, with an optional note. A resolved key is never raised again, so the report doesn't nag about something I've already handled.

## Roadmap and PRDs

Two levels of "what to build":

- **`roadmap.md`** holds bets: problem, solution, expected impact, effort, owner. The agent proposes them. I set each to `accepted`, `refine` or `rejected`. The agent builds only accepted, reversible items that it owns.
- **`prds/<slug>.md`** holds specs for work too big for one session. A `prd` task writes a draft and stops there. Nothing in a draft gets built. When I approve it, its slices become normal queued tasks that merge to staging. I answer the spec's open questions in the PRDs tab, and the answers are written back into the file under each question.

## Two rules

1. **Dry runs never write memory.** Testing the pipeline must not teach the agent anything. Only real runs append to actions, learnings or the roadmap.
2. **Founder text outranks agent text.** Steering overrides defaults. My answers in a PRD outrank the agent's prose. A "Founder changes" line in a spec outranks everything else in it.
