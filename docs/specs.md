# File-Based Tracker — Product Spec

A file-based issue tracker for a solo developer working with multiple AI agents.
Issues live as YAML files in a git repo. All writes go through a CLI. The CLI's
job is to make data corruption and data loss impossible under normal use.

## North star

**No data corruption. No data loss.** Every design decision is judged against this.

## Decisions

| # | Decision | Choice |
|---|----------|--------|
| 1 | Reliability goal | No data corruption, no data loss |
| 2 | Who writes | Humans and agents — but never edit files directly |
| 3 | Storage | One folder, one YAML file per issue |
| 4 | Structure | Structured YAML, validated on every write |
| 5 | Merge policy | Cross-branch conflicts stop for a human (plain git) |
| 6 | Audience | Solo human + multiple agents — no assignee, no accounts, no server |
| 7 | Concurrency | CLI writes atomically and locks per file |

## What it is

- A folder of issues. Each issue is one YAML file.
- A CLI that is the **only** way to create or change an issue.
- Git is the database. History, branches, and backups come from git.

## What it is not

- Not a manual-edit format. Humans and agents both go through the CLI; hand-editing
  a YAML file is unsupported and is the one thing that can corrupt data.
- Not multi-user. No assignee, no accounts, no permissions, no notifications.
- Not a server. No sync daemon, no web board, no real-time updates.

## Core rule: CLI is the only write path

Direct edits are the corruption risk the CLI exists to prevent. The CLI:

- **Validates** every write against the schema before touching disk. An invalid
  status or malformed field is rejected, not written.
- **Writes atomically** — write to a temp file, then rename into place. A crash
  mid-write never leaves a half-written issue.
- **Locks per file** during a write, so two agents running at once cannot clobber
  each other.

## Concurrency

Multiple agents may run the CLI at the same time against the same folder. The CLI
must survive that. Atomic write + per-file lock covers simultaneous writes,
separate worktrees, and one-at-a-time — no configuration needed.

## Merge

Issue files are edited on branches and merged with git. When the same issue is
changed on two branches, git raises a normal merge conflict and a human resolves
it. This is the accepted trade-off; there is no automatic field-level merge in v1.

## Open questions

- Exact YAML schema for an issue (fields, allowed statuses, ids).
- CLI command surface (`create`, `list`, `view`, `set`, status transitions).
- How ids are generated so parallel agents don't collide.
- JSON output mode for agent consumption.
