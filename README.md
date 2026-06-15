# 🪣 bucket-ask

A [Claude Code](https://claude.com/claude-code) skill that stops you (and Claude) from coding before the goal is clear — then builds the whole thing autonomously once it is.

```
/bucket-ask a Chrome extension that summarizes Gmail threads for freelancers
```

**Interview → Intent anchor → Bucket graph → One approval → Autonomous build.**

One interview up front. One approval. Then Claude builds every bucket end-to-end — in parallel where it's safe — and only comes back to you with the finished run. No babysitting, no "shall I continue?" after every step.

## Why

The most expensive bug in any project is building the wrong thing. It usually happens in the first ten minutes — you describe an idea, the AI starts generating code, and three hours later you realize the core question was never answered: *what decision is this project supposed to drive?*

`bucket-ask` front-loads that conversation into a single interview, locks the answer as an **intent anchor**, and then runs autonomously against it. The old version of this skill stopped and waited for your review after *every* bucket — 3–7 forced round-trips per project. This version collapses all of that into **one up-front gate** and then keeps going, stopping mid-run only when something actually goes wrong.

## How it works

### 1. Interview (no code yet)
Claude asks 3–5 sharp questions, **batched in one turn** — not a drip-feed, and it skips anything your description already answered:

- What does success actually look like? Why this project, why now?
- **What decision does this drive?** What will you (or your users) do differently because this exists?
- Constraints: timeline, stack, what already exists.
- What's the smallest version that would still be useful?

The interview can be brief, but it is **never skipped** — it's the whole reason the skill beats "just build it."

### 2. Plan — a bucket *graph*, not a list
The work is broken into 3–7 small, reviewable buckets, each with explicit "done" criteria and a `depends-on` tag. Claude computes execution **waves** from the dependencies:

- The **idea-proving bucket** is a solo **wave 0** — the riskiest piece, run **first and alone** before anything else (every other bucket implicitly depends on it).
- After it passes, independent buckets in the same wave run **in parallel**.
- Dependent buckets pipeline across later waves.

### 3. One gate — the only mandatory stop
Claude presents the **intent anchor + the full bucket graph + an autonomy contract** in a single message and asks for **one** approval. You can reorder, merge, cut, or rewire dependencies here. After you approve, autonomy is licensed — and that approval also covers resuming after any mid-run halt: once you answer the halt, Claude continues under the same approval, it does not re-ask.

### 4. Autonomous build
Claude builds every bucket to completion **without asking between them or between waves**, dispatching one executor subagent per bucket (each carrying the verbatim intent anchor). Independent buckets run in parallel — up to ~3 at a time, each isolated in its own git worktree — and the conductor merges and verifies a wave before opening the next. It posts a one-line progress entry per bucket and keeps moving. (Worktrees apply to multi-file code projects in a git repo; non-repo writing/research/single-file work just builds inline, still in dependency order.)

The idea-proving bucket is gated **automatically**: if it passes its "done" check, the build fans out; if its core premise is *disproven*, that's a halt — no point building on a disproven premise.

### 5. It stops only on a real halt condition
A closed list — the only reasons Claude comes back mid-run:

1. **Drift** — the work no longer serves the approved anchor (or the idea-proving bucket fails).
2. **Unresolvable decision** — a genuine fork the anchor can't settle.
3. **Unrecoverable failure** — a blocker one self-correction can't clear.

A *finished bucket* is not a halt condition. A decision the anchor already answers is not a halt condition. Everything else, Claude decides, logs, and continues.

### 6. One report at the end
A single consolidated review: the anchor restated, a per-bucket log (done / deviations / drift), the decisions Claude made autonomously, an overall "does this serve the original goal?" verdict, and follow-ups. You review and course-correct **once**, with full context — instead of N times mid-build.

### Works everywhere, degrades gracefully
The safety contract (interview, single anchor confirmation, automatic risk gate, halt conditions) holds regardless of the harness. Only the speed mechanism degrades: no worktrees → sequential-but-still-autonomous; no subagents → the conductor builds inline, still in dependency order, still auto-advancing.

## Install

Copy the skill into your personal Claude Code skills directory:

```bash
git clone https://github.com/hasuwini77/bucket-ask.git
mkdir -p ~/.claude/skills/bucket-ask
cp bucket-ask/SKILL.md bucket-ask/LICENSE ~/.claude/skills/bucket-ask/
```

Or for a single project, put it in `.claude/skills/bucket-ask/SKILL.md` inside the repo.

Then in any Claude Code session:

```
/bucket-ask <describe your project>
```

If you invoke it bare, Claude asks for a project description first. It also triggers naturally on phrases like *"interview me before we build"* or *"let's scope this before coding."*

## Example session

```text
> /bucket-ask an app that tracks my freelance invoices and chases late payers

Claude: Before any code — four questions:
  1. Success in 3 months: fewer late payments, or less time spent chasing? Which matters more?
  2. What decision should this drive — when YOU escalate a client, or automating the chasing?
  3. Constraints: just you, or a Micro SaaS from day one? Stack preferences?
  4. Smallest useful version: would a dashboard of "who owes what, how late" already help?

You: (answers)

Claude: Here's the plan — approve once and I'll build it all autonomously.

  Intent anchor: stop losing ~4h/month chasing payments. Core decision this drives:
  which clients get an automated reminder vs. a personal email. Success = chase time
  near zero without damaging client relationships.

  Buckets (graph):
    #  bucket                              depends-on   done
    1  Invoice ledger + overdue detection  —            proves the data model works  ← runs first, alone
    2  Reminder rules engine               1            auto vs. personal escalation decided
    3  Email sending + templates           2            a real reminder goes out
    4  Dashboard UI                        1            "who owes what, how late" visible

  Waves: 0 → B1 solo (idea-prover) · then (B2, B4 in parallel) · then B3
  Contract: on approval I build all 4 end-to-end, stopping only on drift, an
  unresolvable decision, or an unrecoverable failure. Reorder / cut / approve?

You: approved

Claude: ✓ B1 ledger — done, data model proven, on-anchor → fanning out
        ✓ B4 dashboard — done, on-anchor   ✓ B2 rules — done, on-anchor
        ✓ B3 email — done, on-anchor
        [run report] All 4 buckets serve the goal. Decisions log: chose a 7-day
        overdue threshold (anchor: "near-zero chase time"). Follow-ups: none.
```

The whole build runs from a single approval. Claude only interrupts it if the work drifts from the anchor, hits a decision the anchor can't resolve, or fails in a way it can't recover from.

## Credit

The core idea — *interview first, identify the decision the project drives, build in small reviewed increments, verify decisions to prevent drift* — comes from a prompting pattern shared by the community. This skill packages it into a reusable, slash-invocable workflow and adds autonomous, parallel, subagent-driven execution on top.

## License

[MIT](LICENSE)
