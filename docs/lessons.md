# Lessons

[← Back to overview](../README.md)

Most of the design in this repo exists because something failed first. These are the failures worth writing down, roughly in the order they happened.

## The job that couldn't see its own folder

The first schedule was a macOS `launchd` job. It failed every morning with exit code 126 and no log file. macOS privacy controls stop background jobs from reading `~/Desktop`, `~/Documents` and `~/Downloads`, and you can't grant access to a single folder. The options were Full Disk Access for bash or moving the project, so I moved the project. The job then ran cleanly. Later the whole thing moved to a VPS, partly so it no longer depended on a laptop being awake.

**Takeaway:** when a scheduled job fails silently, suspect the environment before the code.

## The run that vetoed itself

On the VPS, the run skipped every day with "before scheduled time". The box clock is UTC, the schedule is in my local time, and the guard compared `date +%H%M` from one against the other. Every 10:00 local run looked like 04:30 and exited.

**Fix:** export one configured timezone at the top of the script and derive every date and time from it.

## A dirty tree that was never dirty

The agents ran inside the production checkouts, with a rule that a dirty working tree means founder work in progress, so stop. But the running services rewrite tracked files (a content cache, a leads file), so the tree was never clean. For five days every frontend and admin task was blocked, and backend, the one repo with a clean tree, was the only one that shipped anything.

**Fix:** executors work in a throwaway `git worktree` from `origin/main`. The production checkout is never touched or inspected, so the question of whether it's dirty no longer comes up.

## The gate nobody could pass

Auto-merge shipped with `npm run typecheck` as the frontend gate. Typecheck had already been failing on `main` for days. So from the day auto-merge went live, no frontend task could merge, and for about two weeks nothing said why.

**Fix:** get `main` green, and make new gates (like the UI audit) judge only the files a branch touched. A gate that fails on old problems teaches you to switch it off.

## The planner that didn't see the data

The context packet had an overall cap. One section, a dump of past decisions, had grown to 53k characters with no budget of its own. It pushed the funnel, Sentry, admin, revenue and learnings sections out of the packet. The planner kept planning without any of today's data, and nothing flagged it.

**Fix:** every section has its own budget, a test asserts the budgets sum below the cap, and truncation is written into the packet where the model can see it.

## A planner that could only find conversion problems

Every input described people who were already on the site: funnel, errors, admin metrics, revenue. So every problem the planner found was a conversion problem, and about half of its first 60 roadmap items were CTA and event tweaks. That was a selection effect, not a strategy.

**Fix:** an acquisition section built from Search Console, the one source that shows demand the product hasn't won yet.

## Zeros that weren't zeros

Five times, a metric read 0% and it looked like nobody used a feature. Each time an event was missing: a component never emitted it, or the funnel's allowlist didn't include it. One of those hid an entire product bet.

**Fix:** instrumentation went into the auto-merge tier. Adding an event doesn't change what a user sees, and a missing one is the most expensive blind spot this project has had.

## Fixes that sat on a branch

Early on, the agent staged good fixes on branches and waited for me. One SEO title fix sat for 9 days while the page it fixed kept losing clicks. The agent's own lesson was "a fix that isn't pushed is worth zero."

**Fix:** a narrow Tier 1 that merges itself after verification, plus the watchdog to undo it if production breaks.

## The wrong Python

A daily data-refresh step began failing with `password authentication failed for user "postgres"`. A venv had been created on the box so the backend tests could run, and the script preferred any venv that existed. But the database URL lives in the container's environment, not the host's, so the venv started with the default URL and failed.

**Fix:** choose the interpreter that can reach the database, not the one that exists. Use the venv only when `DATABASE_URL` is already set, and otherwise run inside the container.

## Product lessons the ops loop surfaced

A few lessons from the product side, found by the same habit of checking real data:

- **A smaller model was more accurate.** For pulling ingredient lists from label photos, an eval on real failed images showed the larger model inventing plausible lists where the smaller one didn't. Production runs the smaller one.
- **The chatbot was mostly crawlers.** About 90% of AllerBot "conversations" were bots following links that fired a chat automatically. I blocked the crawlers and started counting real use separately.
- **A database egress overage came from reference data.** Every product page read the full allergen tables. An in-process cache with a short TTL fixed it.
- **I deleted the vector database.** Ingredient-to-allergen matching had already moved to an LLM call over a structured list. The pgvector tables and embedding code were dead weight, so they went.
