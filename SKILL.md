---
name: bucket-ask
description: Autonomous, subagent-driven project kickoff. Use whenever the user invokes /bucket-ask with a project description, or says things like "interview me first", "scope this before we build", or describes a new project to build incrementally without babysitting it. It runs ONE batched interview to lock the real goal and the core decision as an intent anchor, decomposes the work into a dependency graph of small reviewable "buckets", takes a SINGLE up-front approval, then builds every bucket end-to-end — in parallel where safe, via subagents — without pausing between buckets, stopping only on genuine drift, an unresolvable decision, or an unrecoverable failure.
argument-hint: <describe your project>
---

# Bucket-Ask: Interview → Plan → One Gate → Autonomous Build

The most expensive bug is building the wrong thing. It happens in the first ten minutes — the user describes an idea, the build starts, and hours later the core question was never answered. Bucket-ask kills that by front-loading the intent conversation. **Then it gets out of the way and runs to completion on a single approval.**

The text after `/bucket-ask` is the project description. If it's empty or too vague to interview against, ask for a one-paragraph description first.

## The shape

1. **Interview** — find the real goal (no code, no writing yet).
2. **Plan** — lock the intent anchor; decompose into a dependency graph of buckets.
3. **One gate** — confirm anchor + plan + autonomy contract in a single approval. *The only mandatory stop.*
4. **Autonomous build** — build every bucket to completion, parallel where safe, **without asking between buckets or waves.**
5. **Report** — one consolidated review at the end.

## Speed contract — read this first

Fast means: **one** batched interview round, **one** combined approval, then an autonomous build that stops only on a real halt condition. Fast does **NOT** mean skipping the interview or auto-confirming the anchor — those are the cheap insurance that makes the autonomous build safe. You buy speed by collapsing N human round-trips into one gate, never by deleting the safeguard.

## Phase 1 — Interview (no code, no writing yet)

The interview is the skill. It may be brief; it is never skipped. Uncover, in **one batched turn**:

1. **The actual goal** — not the feature list. What does success look like, and why this, why now?
2. **The core decision this drives** — what will the user (or their users) decide or do differently because this exists? If it drives no decision, surface that — it may be a vanity build.
3. **Constraints** — timeline, budget, stack, what already exists.
4. **The smallest version that would still be useful** — this anchors the idea-proving bucket.

Ask 3–5 sharp questions **batched together** in a single turn (use `AskUserQuestion` if available, else plain prose). Never drip-feed. **Skip any dimension the opening description already answers** — don't pad to hit five. Push back for a second round **only** if a *core* answer (the goal or the core decision) is too vague to anchor against — never for constraints or nice-to-haves; otherwise proceed with a best-effort anchor the user confirms at the gate.

Then draft (do not yet confirm — confirmation happens at the gate) the **intent anchor**: goal + core decision + success criterion, in 2–3 sentences. This becomes the verbatim contract every later step is checked against.

## Phase 2 — Plan: buckets as a dependency graph

Decompose into **3–7 small buckets**, each independently **reviewable** (it produces something judgeable, not invisible plumbing) with explicit **`done`** criteria.

- One bucket is the **idea-prover**: the smallest thing that proves the core idea works — the riskiest, most idea-invalidating piece. It is special (see Phase 4).
- For every *other* bucket declare an explicit **`depends-on`** (the bucket ids it needs; empty = independent). **Every non-idea-proving bucket also implicitly depends on the idea-prover** — treat the idea-prover as a dependency of the whole graph. No cycles: if one appears, merge or split to break it.
- The idea-prover is **wave 0** — always run solo. Compute topological **waves** over the *remaining* buckets only: wave 1 = remaining buckets with no unmet dependency, wave 2 = those whose deps are now met, etc. Independent buckets in the same wave run **in parallel**; dependent buckets pipeline across waves.

Present a table — `# | bucket | depends-on | done-looks-like` — then the computed plan in plain language, e.g. *"Wave 0: B1 (idea-prover, solo gate) · Wave 1 (parallel): B2, B3 · Wave 2: B4 (needs B2, B3)."*

## Phase 3 — The single gate (the ONLY mandatory human stop)

In **one message**, present together:

- **(a)** the intent anchor (goal + core decision + success criterion),
- **(b)** the full bucket graph + wave plan, and
- **(c)** the **autonomy contract**, stated verbatim:
  > *"On your approval I will build all N buckets end-to-end without further check-ins. I will stop only if (1) the work drifts from this anchor, (2) a decision arises this anchor cannot resolve, or (3) I hit a failure I cannot recover from. Otherwise the next thing you hear from me is the finished run report."*

Take **one** approval (`AskUserQuestion`: Approve / Edit / Reorder). The user may reorder, merge, cut, or rewire dependencies here. **Never confirm your own anchor.** Once approved, autonomy is licensed — do not seek approval again. **This approval persists across a halt:** once the user resolves a halt, resume the remaining buckets autonomously under the same approval — do not re-gate.

## Phase 4 — Autonomous build (do NOT ask between buckets or waves)

> **You already have standing approval from the gate. Building the next bucket — or opening the next wave — is not a new decision; it is the continuation of approved work. Asking "shall I continue?" is a defect. The next time you address the user is the end-of-run report, unless one of the three halt conditions below fires.**

**Wave 0 — the risk gate (automatic, never a human stop).** Build the idea-proving bucket **first and alone**, before any other bucket and before wave computation, even if other buckets are dependency-free. Self-verify it: run its detected test/build command if one exists, **and** check each `done` criterion against the actual artifact by inspection or execution; if nothing can confirm `done`, that is halt #1 (the idea isn't provable as built). **Pass → continue immediately** and fan out the remaining waves — no stop. **Fail = halt #1, but only when the core premise is DISPROVEN** (the thing it was meant to demonstrate cannot work or be built as conceived). A merely rough, incomplete, or improvable idea-prover is a **PASS** — keep going. This pays the risk gate automatically so the happy path never stops.

**Waves 1+.** For each wave, in order:

1. **Isolate** — *code in a git repo only*: if parallel executors would write overlapping files, give each its own worktree (`git worktree add ../.bucket-ask/<id> -b bucket/<id>`). Buckets touching disjoint files need no worktree. For non-repo work (writing, research, a single-file script) skip worktrees and use the inline rung.
2. **Dispatch** — issue up to **3** executor `Agent` calls per message (the concurrency cap); they run concurrently. If a wave has more than 3 buckets, send them in batches of ~3 and let one batch return before dispatching the next batch of the same wave. Each executor plans-then-builds its one bucket in a single shot — **no mid-build plan approval.**
3. **Integrate** — the conductor is the single writer to the shared branch. Record a checkpoint commit, then merge each returned branch **in dependency order**. A merge conflict gets **one** conductor resolution attempt; an unresolvable conflict is halt #3.
4. **Verify before advancing** — run the detected test/build command on the merged result (or, if none exists, confirm each bucket's `done` by inspection). Green → continue. Failure → **one** conductor self-correction attempt (a targeted fix executor, or fix inline); still failing → halt #3. Do **not** dispatch the next wave until this wave is green.
5. **Log and continue** — append a one-line progress entry and immediately open the next wave. Logging is non-blocking; a wave boundary is **not** a checkpoint — between-wave pauses are forbidden exactly like between-bucket pauses.

**If a halt fires mid-wave:** dispatch no further waves; let the current wave's in-flight executors finish (or stop them if your harness allows); merge **none** of that wave's branches; leave their worktrees intact for inspection (list the paths in the report); jump to Phase 5.

**Drift check (every bucket, one line).** `DRIFTING` means a **material** departure from the anchor — specifically: the bucket adds a capability the anchor does not name, changes *who decides what*, or makes the success criterion unmeasurable. A reasonable implementation choice, a renamed field, or rough-but-on-target work is **NOT** drift — log it `on-anchor` and continue. Only material drift halts (#1).

**Decision resolution — bias hard toward deciding.** When a decision touches the anchor, resolve it *from* the anchor + interview answers, **make the call, log it with its rationale, and continue — do not ask.** The interview exists precisely so most decisions are pre-answered. Only a decision the anchor genuinely cannot disambiguate is halt #2.

**Progress-log entry (per bucket, non-blocking):**
`### B<n> <name> — done: yes/partial/no · built: <one line> · deviation: none / <what+why> · drift: on-anchor / DRIFTING`

## Halt conditions — the ONLY reasons to stop mid-run

A closed allow-list. If none fires, **keep going automatically.**

1. **DRIFT** — material departure from the anchor (per the Phase 4 drift check), the idea-prover's premise is disproven, or an executor returns a drift flag.
2. **UNRESOLVABLE DECISION** — a genuine fork where both branches are defensible *and* the anchor + interview answers don't disambiguate.
3. **UNRECOVERABLE FAILURE** — a blocker that survives **one** conductor self-correction attempt: a missing secret/credential, an external dependency down, a contradictory requirement, an unresolvable merge conflict, or a verifier/test still red after one fix.

**NOT halt conditions:** a finished bucket · a finished wave · a decision the anchor already settles · a stylistic/implementation choice · simply "wanting to check in."

## Phase 5 — End-of-run report (the single review surface)

When the run completes (or a halt fires), present **one** message: the intent anchor restated; the full progress log (every bucket → done-mapping, deviations, drift verdict); the **decisions log** (every anchor-touching call the conductor made autonomously, with rationale); any halt and the open question it raises (with the paths of any worktrees left for inspection); an overall verdict — *does the delivered whole serve the original goal and core decision?*; and a short list of follow-ups / out-of-scope items. The user reviews and course-corrects here, **once**, with full context. On clean completion, `git worktree remove` each merged worktree before reporting; on a halt, leave unmerged/in-flight worktrees in place (removing a dirty worktree needs `--force` and discards work).

## Conductor vs. executor

Stay the **conductor** in the main session. Never delegate: the interview, the intent anchor, the bucket graph, the single gate, drift adjudication, branch **integration/merge** (you are the single writer to the shared branch), and the final report. Delegate only the **mechanical build** of each bucket.

Every executor subagent — serial or parallel — gets a prompt carrying:
- the **verbatim** intent anchor (frozen at the gate, embedded unchanged),
- this bucket's scope + `done` criteria + its place in the graph,
- its concrete working location: *"Work only in worktree `../.bucket-ask/<id>` on branch `bucket/<id>` (or, no-worktree rung: edit only these files: …). Commit all your changes on that branch before returning — do not leave the tree dirty; the conductor merges your branch."*
- and this boundary: *"Build ONLY this bucket. Do not implement, refactor, or anticipate any other bucket. Do not make decisions that touch the intent anchor — if your work forces a departure from the anchor, or you discover a dependency not yet present, do NOT improvise; STOP and return a drift flag naming the conflict."*

It returns a **tight summary** (built / maps-to-done / deviations / drift flag), not a transcript. Use a capable implementation model for executors; keep judgement in the main session. Any returned drift flag is halt #1.

## Degradation ladder (capability-aware)

The safety contract is **harness-independent** — the interview, the single anchor confirmation, the automatic risk gate, and the halt conditions hold on every rung. Only the *speed mechanism* degrades. Announce the rung up front.

- **Full** — subagents + worktrees + a detected verify command → parallel, isolated, verified waves exactly as Phase 4 describes.
- **Sequential-autonomous** — subagents but no worktrees → executors write to the shared branch one at a time; no merge step (so the merge-conflict halt cannot fire); verify after each bucket; still auto-advance on green.
- **Inline-autonomous** — no subagents, *or any non-code / non-repo project* (writing, research, single file) → the conductor builds each bucket itself, serially, in wave/dependency order; no worktrees, no merge, no parallelism; verify/confirm `done` after each bucket; still auto-advances, still honors the halt conditions. You lose parallelism, not autonomy.
- **No `AskUserQuestion`** → run the interview and the single gate in plain prose.
