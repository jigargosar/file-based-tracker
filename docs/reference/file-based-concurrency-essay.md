# File-Based Systems Under Concurrent Writers: Problems and Resolutions

## 1. The Starting Point

The system in question is a file-based issue tracker / kanban board, versioned in a single Git repository on a single `main` branch. Each task is a YAML file. The intended set of writers is heterogeneous: multiple independent Claude CLI agents, a TUI application, and a web application — all of which may read, and more importantly _write_, to the same issue files, sometimes concurrently, sometimes to the very same file.

The starting question was simple to state and hard to fully resolve: **if multiple independent processes update the same file, what actually happens, and what is the simplest way to avoid losing data — period?**

What follows is the full path this investigation took, including several corrections made along the way, because the corrections themselves are as instructive as the conclusions.

---

## 2. What Actually Goes Wrong Without Any Safeguard

Before reaching for solutions, it's worth being precise about the failure modes, because they are not all the same problem:

1. **Lost updates ("last write wins").** Two processes each read a file, compute a change in memory, and write back. Whichever write lands second silently overwrites the first. No error, no warning — the first change simply ceases to exist.
2. **Corruption from interleaved writes.** If a write is not atomic (editing a file in place rather than replacing it wholesale), two near-simultaneous writes can interleave at the byte level, leaving a truncated or malformed file.
3. **Git does not save you by default.** Git only becomes relevant at commit time. If two writers share a working directory and neither has committed, this is a plain filesystem race that Git never sees. Git only catches a conflict if each writer commits independently (e.g., separate branches or worktrees) — otherwise it is simply absent from the problem.

---

## 3. The First (Incomplete) Framing: Two Problems

An early framing proposed that there were exactly two problems to solve:

1. **No corrupted data.**
2. **No stale data.**

This framing was directionally useful but imprecise. On closer inspection, "stale data" was hiding two structurally different problems that deserve separate treatment:

- **Lost updates (write-write conflict):** two writers race to write the same file; one silently clobbers the other.
- **Stale reads (read-then-decide staleness):** a client holds a snapshot of the data in memory; time passes; the underlying file changes; the client's snapshot no longer reflects reality. This only becomes dangerous if the client _acts_ on the outdated view.

The key distinction is architectural: lost updates are a **write-time coordination problem**, entirely internal to the system — every writer can, in principle, be forced through a single choke point. Stale reads are a **read-time problem**, and reads happen everywhere, constantly, outside the system's control. You cannot force a human staring at a screen, or an agent mid-reasoning, to always be looking at the current state.

### 3.1 The Better Split

A cleaner and more defensible split emerged from this discussion, organized by _layer_ rather than by _symptom_:

1. **Storage-layer guarantees: no corruption, no lost updates.** Both are fully and deterministically solvable by mechanism alone — atomic writes and serialized writes — and neither requires any cooperation or good judgment from callers. This is a guarantee you can make absolutely, once, in the storage layer.
2. **Consumer-layer problem: no stale decisions.** This is never fully solved, only bounded and made _safe_ rather than _silent_. The file itself is never corrupted; the danger is that a reader's mental model of the world has drifted from reality, and it may act on that drift.

The place these two layers touch is instructive: a version-check at write time (a storage-layer mechanism) is what _catches_ a stale-basis write before it lands — so the storage layer's mechanism is also what converts a consumer-layer problem from "silent data loss" into "a visible, safe rejection."

---

## 4. What a Single-Writer Queue Solves — and What It Doesn't

A natural proposed solution: put a single serialized write-service in front of the repository. Every writer — agents, TUI, webapp — sends a change _request_ rather than performing a raw file write. The service processes requests strictly one at a time.

This is a strong mechanism, but it is important to be precise about its actual boundary of guarantee, because it was initially oversold.

**What the queue genuinely solves, by mechanism alone:**

- No two writes can physically interleave, because only one is ever being processed at a time.
- Combined with atomic writes (temp file + rename), no file is ever left corrupted or half-written.

**What the queue does _not_ solve on its own:**

1. **Stale-basis writes.** The queue guarantees writes are applied safely, one at a time — it says nothing about whether the _content_ of a given write request is still correct. An agent that read stale data and computed a decision from it will have that decision applied cleanly and destructively if nothing else checks its premise.
2. **Whole-file overwrites still lose data, even serialized.** If the write operation is "replace the entire file" rather than "patch this field," two writers editing _different_ fields will still clobber each other, because the queue faithfully applies exactly what it's told — it prevents _interleaved_ writes, not _blind, complete_ ones.
3. **Cross-file transactions are not atomic just because each file's write is.** An operation spanning multiple files (moving a card between column files, updating a parent epic's rollup count) can leave files internally consistent but mutually disagreeing if it's interrupted partway.
4. **The queue itself can lose an update if it acknowledges before persisting.** If the queue accepts a request, crashes before durably recording it, and tells the client "done" anyway — that's a new failure mode the queue introduces if built carelessly.
5. **Serialization gives an order, not a correct resolution.** Two writers with genuinely opposed intents (one sets `done`, one sets `blocked`, nearly simultaneously) will both be applied cleanly, in some order — with no indication a real conflict of intent occurred. Nothing is technically lost, but the outcome may reflect neither party's actual intention.
6. **Any writer that bypasses the queue voids every guarantee instantly.** A human running `vim` directly on a file, a stray script, a `git checkout` — none of this is protected, because the guarantee is only as strong as the discipline ensuring every writer genuinely goes through the queue with no side doors.

The conclusion: the queue's real, solid guarantee is narrow — no interleaving, no corruption. Everything else requires an additional, separate mechanism.

---

## 5. Two Competing Strategies for Closing the Gap

Two different, valid strategies emerged for handling the remaining problem — silent last-write-wins on conflicting intent.

### 5.1 Pessimistic Locking (the "vim swap file" model)

A lock file is created atomically before an edit begins (using an atomic create-if-not-exists primitive, never "check then create," which is itself racy) and deleted when the edit is done. Critically, the lock must be held across the **entire** read → decide → write cycle, not just around the final write — otherwise the original race simply reopens, because both readers would have already computed a full new file from stale content before either lock is taken.

**What this fully solves:** whole-file replace becomes safe without needing patch-level operations, because true time-overlap between writers is prevented outright rather than detected-and-rejected afterward.

**What it does not solve:**

- It is advisory, not enforced — it only works if every writer honors it. One writer that skips the check, or a human editing the file directly, breaks the guarantee completely.
- Stale locks from crashes — if a session dies mid-edit without releasing its lock, every future editor is blocked forever unless staleness detection (PID + timestamp + timeout) is added.

**Tradeoff:** coarser concurrency — a lock blocks _all_ edits to that issue for its duration, including edits to unrelated fields, and for an LLM agent, "decide" can mean seconds to minutes of reasoning time held under lock.

### 5.2 Optimistic Concurrency (version stamp + queue)

Each file carries a `version` field. A writer reads at some version, computes its change, and submits the change along with the version it was based on. The queue, at the moment of processing, compares the submitted `based_on_version` against the file's current version:

- **Match** → apply the change, bump the version, atomic rename.
- **Mismatch** → reject, or attempt a tiered resolution (see Section 6).

**What this catches that locking cannot:** a lock only knows "is someone else writing right now" — it has no concept of "has the world moved since you formed your intent." A version check specifically catches the case where the writer's decision was formed on data that has since changed, which is the actual mechanism that prevents silent last-write-wins.

**Tradeoff versus locking:** finer-grained concurrency (readers and thinkers never block each other; only the instant of commit is checked) at the cost of needing patch-shaped operations and a defined resolution policy for genuine conflicts.

---

## 6. Tiered Conflict Resolution, and Retry vs. Re-Decide

A version mismatch should not simply mean "hard fail" nor "blindly retry the same payload" — both are wrong in different ways. Blind retry of the same payload reproduces the original problem, because the payload's premise is stale; a bare hard fail is unnecessarily rigid when many conflicts are actually resolvable.

A tiered model was proposed:

1. **Tier 1 — auto-mergeable.** If the operation is field-scoped (e.g., "append a comment," "set priority") and the field(s) touched by the pending write don't overlap with whatever changed in the meantime, the write can be safely rebased onto the current version and applied automatically. This requires operations to be shaped as patches (named fields), not whole-file replacements.
2. **Tier 2 — same-field conflict.** Two writers touched the _same_ field. This cannot be silently auto-merged, but it need not be a hard failure either — a defined, deterministic policy (first-writer-wins, or a fixed precedence rule) can be applied automatically _while logging the loser's payload_, rather than simply rejecting.
3. **Tier 3 — no safe policy exists.** Only here does the write genuinely fail, returning current state and the caller's own payload so a fresh decision can be made.

The distinction between "retry" and "re-decide" matters here: resubmitting the same payload after a conflict is close to meaningless once the world has moved on. What's actually needed is a _new_ decision formed from the _current_ state — not a mechanical resend of the old one.

---

## 7. Synchronous vs. Asynchronous Transport, and Why It Simplifies Things

A late but important correction to the design: the client-server write API is **synchronous** — a write call returns its result (applied / auto-merged / conflict) in the same call, not via a separate notification channel checked later.

This single fact substantially simplifies the design in two ways:

1. **It removes the need for a separate escalation/notification system.** Conflict resolution isn't a deferred async event; it's just the return value of the call the caller already made and is already waiting on.
2. **It dissolves the "lock held during LLM reasoning" problem.** Because there is no lock spanning the full read-decide-write cycle in this model, an agent reads and thinks entirely on its own time, and only makes one quick synchronous call at the end to commit. Nothing is held open during reasoning, so autonomous agents don't introduce a new stuck-lock risk the way they would under the pessimistic-locking model.

### 7.1 The "Same Turn" Confusion, Resolved

A significant amount of discussion revolved around whether a conflict response needs to be "handled in the same turn." The resolution, after several corrections, was:

- The conflict information only exists at the moment the call returns — if nothing is done with it right then, it's gone, because nothing re-delivers it later.
- But "doing something with it" can be as minimal as writing it to a durable log and moving on. It does not require fully resolving the conflict in that instant.
- For a **Claude agent specifically**, this entire concern is largely moot: a tool call's result is simply inserted into the agent's conversation context, exactly like any other tool result. The agent doesn't need bespoke conflict-handling code bolted on, because reacting sensibly to a tool result is what an agent already does, every turn, for every tool, by default. A conflict response is just information the agent reasons about naturally.
- The genuine remaining requirement is narrower than first stated: it applies specifically to a fully unattended, non-agentic caller (a bare script with no reasoning loop at all) — there, and only there, does someone need to have pre-written an explicit catch-and-log branch, because there's no reasoning process present to notice and react to the response.

### 7.2 Do We Need Shared/Broadcast Conflict Information?

A tempting but ultimately unjustified idea was that some shared, cross-agent notification system might be required — so that one agent's conflict is visible to others. On systematic examination of candidate scenarios (a stale-looking TUI, two agents starting the same work before either has written anything, cross-file rollups), none of them actually required shared state for **correctness**. Each reduces either to "call the write API again right before acting" (already covered by the synchronous per-write check) or is a display-freshness concern rather than a data-integrity one. The conclusion: no shared/broadcast system is required for the core "never lose data" guarantee — any live-refresh mechanism is a UX enhancement, not a correctness dependency.

---

## 8. The Real, Separate Problem: Real-Time UX and Board Reflow

A distinct and legitimate concern surfaced: even if the underlying data is never corrupted or silently lost, a live-updating UI (TUI or webapp) that reflows in response to incoming changes can cause **user error** — e.g., a card shifting position as the user is about to click "archive," resulting in the wrong card being archived. This is a well-known problem for any real-time interactive system, not a defect specific to this design.

Patterns that address it, drawn from how established tools (Trello, Linear, Notion, Figma) handle this:

1. **Don't auto-apply reordering/position changes — badge them instead.** Hold incoming positional changes in a pending buffer with a visible "updates available" indicator; the visible board stays frozen until the user chooses to pull in updates.
2. **Separate "content changed" from "position changed."** Non-positional field updates (a title edit, a comment) can safely patch in place live; only reordering/column moves are dangerous enough to warrant holding back.
3. **Freeze zone around active interaction.** No update touches the DOM near an open context menu, an in-progress drag, a focused input, or within a short window after the last pointer interaction — directly preventing the "card moved right as I clicked" failure.
4. **Animate reorders (FLIP-style), never jump-cut**, when a queued reorder is eventually applied, to preserve the user's visual tracking.
5. **Make the destructive action itself recoverable regardless of rendering care** — archive should be a reversible, undoable soft-state change, not an immediate permanent action. This ties back naturally to the write-server design: "archive" can simply be another versioned, atomic write, not a destructive delete.

---

## 9. What Real Products Actually Do

A deliberate check against real-world tools (Trello, GitHub Issues/Projects) confirmed that the "obsessive" end of this design space is not, in fact, what shipped products do:

- Both appear to use **field-level last-write-wins** for structured metadata (status, labels, custom fields, position), with **no conflict UI at all**. A true conflict only arises when two writers touch the exact same field at the exact same instant — treated as rare enough not to warrant a resolution flow.
- **Comments are handled differently** — they're append-only events, not a shared mutable field, so concurrent commenting simply produces two comments rather than any collision.
- For genuinely free-text fields (titles, descriptions), the same last-write-wins-on-save applies, with client-side draft preservation (e.g., session storage restoring an unsent edit on refresh) as the only real safeguard against a user's own in-progress typing being lost locally.
- True real-time collaborative merging (OT/CRDTs) is reserved for tools where that specific quality bar matters, like Google Docs — it is not the default even among mainstream project-management tools.

This validated a specific personal observation: two tabs open on the same item (as tested against Todoist), where tab 1 writes A→B and tab 2, still looking at stale data, writes A→C — C wins, B is invisible, and no broadcast or conflict resolution intervenes. This is not a bug particular to Todoist; it is the industry-standard behavior for this class of field.

---

## 10. Journals, Version History, and What They Actually Fix

A separate mechanism was raised: an append-only journal or activity log (as in Todoist's activity log, or Google Docs' named version history). This solves a genuinely different problem than anything above.

- Locks, version-checks, and serialization are about **preventing** a conflict, or catching it at the moment it happens.
- A journal doesn't prevent anything — it guarantees that no value that ever existed is ever **destroyed**, even if it stops being the "current" value.

Applied to the earlier Todoist example: tab 2's overwritten value (B) is not currently displayed, but if every write is also appended to an immutable log, B is recoverable from history — "lost" and "no longer the current display value" become different things.

**What a journal does _not_ fix:**

- It doesn't prevent the live-view flicker/confusion the user experienced in the moment — it provides recovery, not prevention.
- It doesn't resolve whose intent should win; both values are preserved, but nothing in the log indicates which one _should_ be current.
- Its guarantee depends entirely on the log write being durable at (or before) the same moment as the state write — if that ordering is violated, the same class of silent loss reappears one layer up.

**The key practical insight:** this mechanism does not need to be built separately. Git _is_ this pattern already. If every accepted write is followed by a commit, you get Google-Docs-style version history for free, using infrastructure the system already depends on — nothing a commit once recorded is ever destroyed by a later commit; it remains reachable via `git log`, `git show`, and `git blame` even after the current file no longer reflects it. Named versions map naturally onto annotated tags or commit-message conventions layered on the same history.

---

## 11. Autocommit Is Not a Substitute for Serialization

A specific and important correction arose here. The proposal was: for a solo user with no concern about commit-history noise, why not just autocommit the kanban folder on every change, instantly?

This does not solve the concurrent-write race, and the reason is precise: autocommit only takes a snapshot of _whatever is currently on disk_ at the moment it runs. It has no visibility into what happened _before_ that moment. If two processes both read a file, both compute a change, and both write to disk in close succession, the second write physically overwrites the first at the OS level — the first version never persisted long enough to exist as a state anyone could commit. By the time autocommit fires, there is only ever one file on disk to snapshot; the losing write vanished before Git ever had a chance to see it.

Autocommit protects against a _different_ failure mode entirely — forgetting to save, or a machine crash losing uncommitted work. It gives you **recoverability of every committed state**, but nothing about **preventing two writers from racing each other**, because that race resolves at the filesystem level, in microseconds, entirely between commits — no matter how frequently commits occur.

The corrected, unified position: git history and a single-writer queue are not alternatives, they operate at different layers, and only the queue touches the actual concurrent-write problem.

---

## 12. Folding Commit Into the Write Path

Rather than treating autocommit as a separate process observing the filesystem after the fact, the cleaner design folds it directly into the write server's own handler: for every request, the sequence becomes _check version → apply change → atomic write (temp + rename) → git commit → return response_ — all within the same serialized operation.

**Why this matters beyond tidiness:** if commit is part of the same serialized, version-checked operation as the write, every commit in the history corresponds to exactly one accepted, validated change. There is a clean 1:1 mapping between accepted writes and commits — never a half-applied state, never two racing writes landing in the same commit, never a commit that happened before a version check passed.

**But this has a real cost, which must be named rather than assumed away:**

1. **Latency per write.** A git commit is not microseconds like an atomic rename — typically single-digit-to-tens of milliseconds, more under adverse conditions (hooks, antivirus/indexer interference, cold disk). Folded into a serialized handler, every write now waits on git's internals, not just a file rename.
2. **Throughput ceiling.** Since all writes funnel through one serialized point, commit latency directly caps maximum system-wide writes per second. For a solo user with a handful of agents doing occasional field updates, this ceiling is very likely irrelevant — but it is now a real, named constraint of the design rather than a free operation.
3. **New failure mode.** A commit can fail in ways a plain rename essentially cannot (no git identity configured, disk full, a rejecting hook) — meaning the write handler now needs an explicit answer for "file correctly written, but commit failed," which did not previously exist as a case.
4. **Impact on other systems.** Any external git command touching the same repository outside the write server (a manual `git commit`, a rebase, a pull) can race the server's own commit and reopen exactly the "side door bypasses the guarantee" problem discussed earlier — the discipline of "only the server writes to this repo" must hold for this too.

**Mitigation, if the cost ever matters:** decouple write from commit — keep the atomic rename and version check synchronous (this part must stay instant, as it is the correctness guarantee), but push the `git commit` step just after the response is sent, or batch it (every N seconds or N pending writes). This trades a small, bounded window — where a change is correctly on disk and already returned to the caller, but not yet committed — for lower caller-facing latency. This is a strictly smaller and different risk than the original write-race problem, and one whose size is fully controlled by how often the batch flushes.

For the stated scale (a solo user, a handful of agents), synchronous commit-per-write is a reasonable default, with batching kept available as a one-line change if write latency is ever actually observed to matter.

---

## 13. The General Pattern: Single-Writer-Per-Resource, and Why It Scales

A late realization tied this entire design back to a much more general and well-known pattern: **single-writer-per-resource**, sometimes called the actor model. The rule is: for any given piece of mutable state, all writes to _that specific piece_ must pass through exactly one serialization point — but different pieces of state can have entirely independent serialization points, running fully in parallel.

This generalizes cleanly to scale:

- **The correct partition key is the resource being written (e.g., the issue/file), not the caller doing the writing.** Partitioning by user would still serialize a single busy user's own unrelated work unnecessarily, while gaining nothing against the actual source of contention — two agents touching the same issue.
- **At larger scale**, this becomes: N workers, each owning a subset of resources (e.g., by hashing the issue ID), each internally running the same simple single-writer queue as the solo case. Writes to different issues never contend at all; only two writes to the _same_ issue ever wait on each other — which, per the real-world comparison in Section 9, is already the rare case in practice.
- **The genuinely hard part at scale** — not required for the current use case — is what happens when the server owning a partition dies: a new owner must take over without either replaying an already-completed write or losing one that was in flight. This is the domain of consensus and partition-rebalancing systems (as in Kafka partition rebalancing or Raft-based systems) — real infrastructure, because "which server is authoritative for this partition right now" is itself a distributed-agreement problem.
- **For a solo user, there is exactly one partition and one worker** — none of the rebalancing/failover complexity applies, because there is nothing to shard. The conceptual model is identical at every scale; larger scale just means more copies of the same primitive plus a partitioning and failover layer on top.

It was noted, correctly, that this entire pattern is not novel — it is precisely what mature relational databases already provide: a write-ahead log for atomic, non-corrupting writes; MVCC and row-level version checks for stale-basis detection without silent overwrite; and row-level locking for serializing contention only where two operations actually touch the same row. A hand-built file-based system converges, by necessity, on a weaker version of exactly what systems like Postgres or SQLite already ship — which is a legitimate reason to weigh "files" against "an embedded database" deliberately rather than by default, even though the reasons to prefer files (human readability, direct editability, git-friendly diffs, no schema migrations) remain real and were an explicit, considered choice, not an oversight.

---

## 14. The Practical, Human Corollary

Two closing observations connected this architecture back to lived experience rather than pure design:

1. **The solo-to-small-team version of this system is discipline, not infrastructure.** Manually partitioning work ("don't touch this file while I'm in this one"), updating the board one change at a time, and using git worktrees for genuinely divergent experimental work is, functionally, the pessimistic-locking model — just enforced by a human coordinating in real time rather than code checking a version field. This is a perfectly valid, load-bearing strategy for as long as one person can reliably hold the current partitioning in their head. The signal to formalize it into actual code is not "this system has multiple writers" — it's the moment coordination starts being forgotten or getting out of sync.
2. **Agents and junior developers fail in different, non-overlapping ways, even though their surface behavior can look similar.** A junior developer's eventual pragmatism is earned judgment, built from feedback over time, that persists on its own without being re-supplied. An agent's apparent pragmatism is the _absence_ of a strong default combined with no memory across sessions — it looks similar ("doesn't care how it's written, just wants it working") but requires the instructions to be re-established every time, because nothing is being learned or carried forward the way a person's judgment is. This distinction matters directly for the system described here: the discipline that keeps a small team of either humans or agents safe without heavyweight infrastructure is the same discipline in both cases, but the enforcement mechanism likely needs to be more explicit and repeated for agents, precisely because they will not organically notice or flag a conflict the way a person eventually will.

---

## 15. Summary: The Minimal Design That Meets the Original Bar

Distilled to what actually satisfies "never lose data, period," without the extensions that are available but not required at the current scale:

1. **Atomic writes** — every write goes to a temp file, then an atomic rename. This alone eliminates corruption, unconditionally.
2. **A single write server** — every writer (agent, TUI, webapp) sends a request rather than touching the file directly; requests are processed strictly one at a time.
3. **A version field per issue**, checked against the version the requester's decision was based on, at the moment of write.
4. **A synchronous request/response API** — the write call's result (applied, auto-merged, or conflict) comes back in the same call; for Claude agents specifically, this integrates naturally into the existing tool-call/reasoning loop with no special handling required.
5. **A git commit folded into the same write handler**, giving complete, gapless, per-change history for free — with the understanding that this trades a small, generally negligible amount of per-write latency for that guarantee, and can be decoupled/batched later if that cost is ever actually observed to matter.
6. **No lock spanning the full read-decide-write cycle**, avoiding the stuck-lock risk that would otherwise apply specifically to autonomous, unattended agents.
7. **Tiered conflict resolution reserved for when it's actually needed** — most fields can reasonably default to a simple, logged resolution policy, following the same pattern real-world tools (Trello, GitHub) already use in practice, rather than a heavyweight resolution UI built in advance of any evidence it's required.
8. **Real-time UX safeguards (reflow freezing, undo-based archiving) treated as a separate concern**, addressed only once — and if — a live-updating interface with a human actively interacting with it is actually part of the system, rather than solved preemptively for a usage pattern that may not occur.

The overarching lesson of the discussion is that "never lose data" decomposes into a small number of genuinely solvable, mechanical guarantees (no corruption, no lost updates) and a genuinely unsolvable-in-the-absolute-sense but fully boundable problem (staleness of a reader's view) — and that the right amount of machinery to build for any of this scales with actual, observed contention, not with the theoretical worst case.
