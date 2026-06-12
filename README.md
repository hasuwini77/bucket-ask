# 🪣 bucket-ask

A [Claude Code](https://claude.com/claude-code) skill that stops you (and Claude) from coding before the goal is clear.

Instead of jumping straight into implementation, `/bucket-ask` forces a short structured kickoff:

```
/bucket-ask a Chrome extension that summarizes Gmail threads for freelancers
```

**Interview → Intent anchor → Buckets → Build with checkpoints.**

## Why

The most expensive bug in any project is building the wrong thing. It usually happens in the first ten minutes — you describe an idea, the AI starts generating code, and three hours later you realize the core question was never answered: *what decision is this project supposed to drive?*

`bucket-ask` front-loads that conversation, then keeps the build honest with review checkpoints and explicit drift checks.

## How it works

### 1. Interview (no code yet)
Claude asks 3–5 sharp questions, batched — not a drip-feed:

- What does success actually look like? Why this project, why now?
- **What decision does this drive?** What will you (or your users) do differently because this exists?
- Constraints: timeline, stack, what already exists.
- What's the smallest version that would still be useful?

It then plays back the goal, core decision, and success criterion in 2–3 sentences and waits for your explicit confirmation. That playback becomes the **intent anchor** — the reference point for everything that follows.

### 2. Buckets
The work is broken into 3–7 small, agile buckets:

- Each bucket produces something **reviewable** — no invisible plumbing buckets.
- Bucket #1 is the smallest thing that proves the core idea.
- Ordered by risk: the bucket most likely to invalidate the idea comes first.

You can reorder, merge, or cut buckets before anything gets built.

### 3. Build, one bucket at a time
For every bucket:

1. **Plan** — a short, skimmable plan for this bucket only.
2. **Build** — execute.
3. **Checkpoint** — show the output, map it to the bucket's "done" criteria, and *stop*. The next bucket doesn't open until you've reviewed.

Claude is explicitly forbidden from silently rolling two buckets together.

### 4. Drift guard
Any decision that touches the intent anchor — scope expansion, UX tradeoffs, tech choices that constrain the future — gets called out explicitly with options and a recommendation. Every checkpoint ends with a one-line drift check: *does what we just built still serve the original goal?*

## Install

Copy the skill into your personal Claude Code skills directory:

```bash
git clone https://github.com/hasuwini77/bucket-ask.git
mkdir -p ~/.claude/skills/bucket-ask
cp bucket-ask/SKILL.md ~/.claude/skills/bucket-ask/
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
  1. Success in 3 months: fewer late payments, or less time spent chasing? Which one matters more?
  2. What decision should this drive — when YOU escalate a client, or automating the chasing itself?
  3. Constraints: is this for you only, or a Micro SaaS from day one? Stack preferences?
  4. Smallest useful version: would a dashboard showing "who owes what, how late" already help,
     even with zero automation?

You: (answers)

Claude: Intent anchor — you want to stop losing ~4h/month chasing payments. The core
decision this drives: which clients get an automated reminder vs. a personal email.
Success = chase time near zero without damaging client relationships. Confirm?

You: confirmed

Claude: Buckets, riskiest first:
  1. Invoice ledger + overdue detection (proves the data model works)   ← start here
  2. Reminder rules engine (auto vs. personal escalation)
  3. Email sending + templates
  4. Dashboard UI
  Reorder, merge, or cut anything?
```

Each bucket then gets a short plan, the build, and a checkpoint — with a one-line drift check against the intent anchor before the next bucket opens.

## Credit

The core idea — *interview first, identify the decision the project drives, build in small reviewed increments, verify decisions to prevent drift* — comes from a prompting pattern shared by the community. This skill packages it into a reusable, slash-invocable workflow.

## License

[MIT](LICENSE)
