---
name: bucket-ask
description: Structured project kickoff — interview the user to find the real goal and core decision before any code or writing happens, then break the work into small agile "buckets" built one at a time with a plan and review checkpoint per bucket. Use whenever the user invokes /bucket-ask with a project description, or says things like "interview me first", "let's scope this before building", or describes a new project they want to approach incrementally with checkpoints.
argument-hint: <describe your project>
---

# Bucket-Ask: Interview → Buckets → Checkpoints

The user is starting a project and wants to avoid the classic failure mode: jumping into code before the actual goal is clear, then drifting from the original intent halfway through. Your job is to slow down at the start so the build phase is fast and on-target.

The text after `/bucket-ask` is the project description. If it's empty or too vague to interview against, ask for a one-paragraph description first.

## Phase 1 — Interview (no code, no writing yet)

Do NOT start building. Interview the user to uncover:

1. **The actual goal** — not the feature list, but what success looks like. Why this project, why now?
2. **The core decision this project drives** — what will the user (or their users) decide or do differently because this exists? If the project doesn't drive a decision or action, surface that — it may be a vanity build.
3. **Constraints** — timeline, budget, stack preferences, what already exists.
4. **The smallest version that would still be useful** — this anchors bucket #1.

Ask 3–5 sharp questions, batched together (use AskUserQuestion if available, otherwise plain conversation). Don't drip-feed one question per turn. When answers are vague, push back once with a concrete example of what a specific answer looks like.

After the interview, play back your understanding in 2–3 sentences: the goal, the core decision, and the success criterion. **Get explicit confirmation before moving on.** This playback becomes the project's "intent anchor" — you will check against it later.

## Phase 2 — Bucket the work

Break the project into small, agile buckets:

- Each bucket is independently shippable or reviewable — it produces something the user can look at and judge, not invisible plumbing.
- Bucket #1 is the smallest thing that proves the core idea works.
- 3–7 buckets is the sweet spot. More than that means the buckets are too thin; fewer means they're too fat.
- Order by risk: put the bucket most likely to invalidate the idea earliest.

Present the bucket list with a one-line scope per bucket and what "done" looks like for each. Let the user reorder, merge, or cut before you start.

## Phase 3 — Build one bucket at a time

For each bucket, in order:

1. **Plan** — present a short plan for this bucket only (approach, files touched, anything you'll need from the user). Keep it skimmable, not a design doc.
2. **Build** — execute the plan (see "Conductor vs. executor" below).
3. **Checkpoint** — show the output and stop. Summarize what was built, how it maps to the bucket's "done" criteria, and anything you deviated on and why. Wait for the user's review before opening the next bucket.

Never silently roll two buckets together "because they were related" — the checkpoint between them is the point.

### Conductor vs. executor (use subagents for the Build step)

Stay the **conductor**: you hold the intent anchor, run the interview, present buckets, and run every checkpoint and drift check. Those are user-facing and must stay in the main session — never delegate them. But the **Build step is independent, well-scoped implementation work** — exactly what a fresh-context subagent does best, and keeping it out of the main session keeps your intent anchor and checkpoint reasoning uncluttered.

If your harness supports subagents (e.g. Claude Code's Agent tool), dispatch **one executor subagent per bucket** to do the build:
- **Pass it the intent anchor.** A fresh subagent never saw the interview. Its prompt must carry the goal + core decision + this bucket's scope and "done" criteria + the approved plan — or it will drift exactly the way Phase 1 was meant to prevent.
- **One subagent per bucket, never one spanning two.** The subagent boundary IS the bucket boundary; the checkpoint barrier between them still holds.
- **It returns a tight summary only** — what it built, how it maps to "done", deviations — not a transcript. You relay that into the checkpoint.
- **Model:** use a capable implementation model for the executor and keep planning/judgement (bucketing, checkpoints, drift) in the main session.

If subagents aren't available, build inline — but still keep the Build mechanical and the judgement (plan, checkpoint, drift) deliberate.

## Drift guard — verify key decisions explicitly

Throughout the build, when a decision arises that touches the intent anchor (scope expansion, a tradeoff that changes the user experience, a tech choice that constrains the future), call it out explicitly: state the decision, the options, your recommendation, and how it relates to the original goal. Don't bury decisions inside a wall of implementation detail.

At each checkpoint, do a one-line drift check: does what we just built still serve the goal and core decision from Phase 1? If something has drifted, say so plainly — it's cheaper to course-correct at a checkpoint than at the end.
