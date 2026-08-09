# todo.txt DSL Specification

> **Status:** v0.1 — 2026-08-09
> **Clients:** `todo.sh` + text editor only.  No mobile or GUI app.
> **Sync target:** Microsoft To Do (single list).  Google Tasks: future, not specified.
> **Ground rules** are in the problem statement and are not repeated here; they
> are enforced silently throughout.

---

## Part 0 — Provider Capability Assumptions

*Assumed on 2026-08-09.  Each claim must be re-verified against the current
Microsoft Graph `/me/todo` documentation before any production deployment.*

### 0.1 Capability Table

| Capability | Microsoft To Do | Confidence | Verify at |
|---|---|---|---|
| Max nesting depth (`checklistItems`) | 1 level only; `checklistItems` cannot contain sub-items | [HIGH] | `GET /me/todo/lists/{id}/tasks/{id}/checklistItems` |
| Priority / importance levels | Boolean: `importance: high \| normal` | [HIGH] | `PATCH /me/todo/lists/{id}/tasks/{id}` `importance` field |
| Reminder with time-of-day | Yes — `reminderDateTime` is an ISO 8601 `dateTimeTimeZone` object | [HIGH] | `todoTask` resource schema |
| Start date vs due date | **Due date only** — `dueDateTime` (`dateTimeTimeZone`); no `startDate` field | [HIGH] | `todoTask` resource schema |
| Recurrence expressiveness | `recurrencePattern` object: `daily`, `weekly`, `absoluteMonthly`, `relativeMonthly`, `absoluteYearly`, `relativeYearly` | [HIGH] | `patternedRecurrence` resource schema |
| Body / notes field | Yes — `body.contentType: text` only (HTML is ignored) | [MEDIUM] | `itemBody` resource schema |
| **Task ID character alphabet** | GUID: `[0-9a-f-]`, e.g. `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | [HIGH] | Affects D5; safe for `map.tsv`; never put on task line |
| **ID preserved across cross-list move** | **No** — a cross-list move via the API deletes and re-creates, issuing a new GUID | [MEDIUM] | Affects D9; verify with `POST /me/todo/lists/{dst}/tasks` + delete |
| Delta / incremental change endpoint | Yes — `GET /me/todo/lists/{id}/tasks/delta` + `@odata.deltaLink` | [HIGH] | Affects sync cost; verify that deleted tasks appear as `@removed` |

### 0.2 Capability Matrix

For each DSL key, whether its value is **native** (a first-class To Do field),
**emulated** (approximated in the task body or title prefix), or **dropped**
(no faithful representation; local only).

| DSL key / construct | To Do field | Status | Notes |
|---|---|---|---|
| `id:<n>` | — | dropped | Lives in `map.tsv`; never sent to To Do |
| `p:<n>` (parent link) | — | dropped | No parent-child on tasks; children sync as independent tasks |
| `due:YYYY-MM-DD` | `dueDateTime` | **native** | UTC midnight is used when no time is stored |
| `t:YYYY-MM-DD` (threshold) | — | dropped | No start-date field; local topydo/filter only |
| `rem:YYYY-MM-DDTHHMM` | `reminderDateTime` | **native** | See §1.3.2 for format rationale |
| `star:1` | `importance: high` | **native** | Boolean; see §1.1.2 |
| `(A)`–`(Z)` priority | — | dropped | No ordinal field; see §1.1.2 |
| `s:next\|wait\|someday\|blocked` | — | dropped | Stored locally only |
| `wait:@person` | — | dropped | No personal-task assignee field |
| `myday:YYYY-MM-DD` | — | dropped | No "My Day" API field; see §1.1.1 |
| `e:low\|med\|high` | — | dropped | No energy/effort field |
| `@context` | — | dropped | Kept in title text; not a native To Do concept |
| `+project` | — | dropped | Kept in title text |
| Recurrence (To Do → local) | `recurrencePattern` | **emulated** | Pattern stored in `map.tsv`; see §3.3 |
| Completion `x ` | `status: completed` | **native** | Round-trips |
| Completion date | `completedDateTime` | **native** | Round-trips |
| Creation date | `createdDateTime` (read-only) | **native** | Pulled on first create; not written back |

---

## Part 1 — Core Schema

### 1.1 Resolved Collisions

#### 1.1.1 MyDay as a Project

**Problem.** Using `+MyDay` writes ephemeral daily state into the permanent
project namespace.  `todo.sh listproj` lists it alongside real projects;
`+MyDay` lines from yesterday give false positives; removing the tag on a new
day leaves a diff that looks like an edit, not a state change.

**Resolution.** Use the dated key `myday:YYYY-MM-DD`.

- `myday:` is not a project tag; `listproj` is unaffected.
- The date makes it self-expiring: `grep "myday:$(date +%F)" todo.txt` gives
  today's My Day list.
- **To Do mapping:** dropped.  The MS Graph API has no "My Day" field on a
  task; the concept is a UI view that resets nightly and cannot be pinned via
  API.  Using `reminderDateTime` as a proxy is explicitly rejected: it would
  corrupt the `rem:` key on tasks that have a real reminder set.
- **Google note:** Google Tasks has no My Day concept either.
- **Migration cost:** `sed -i 's/ +MyDay/ myday:'"$(date +%F)"'/g' ~/.todo/*/todo.txt`
  Run once; the tag is removed from all project listings immediately.

#### 1.1.2 Star vs. Priority

**Problem.** Mapping `(A)` → `importance:high` is lossy in both directions:
it conflates an ordinal rank with a boolean flag, and pulling `importance:high`
back would force `(A)` onto tasks that the user may have intentionally left
un-prioritised.

**Resolution.** Two independent fields with an explicit projection rule.

| Field | Format | Semantics |
|---|---|---|
| todo.txt priority | `(A)`–`(Z)` at line start | Ordinal; local only; not synced |
| `star:1` | key-value | Boolean importance; maps to `importance:high` in To Do |

**Projection rule (outgoing):** `star:1` → `importance: high`; absent or
`star:0` → `importance: normal`.  Priority `(A)` does **not** automatically
set `star:1`.

**Projection rule (incoming):** `importance: high` → add `star:1` if absent;
`importance: normal` → remove `star:1` if present.  Priority field is never
touched by sync.

### 1.2 Key Set

All keys follow the todo.txt `key:value` rule: the value **must not contain
spaces**.  Violations are rejected by the `lint` addon, not silently ignored.

#### 1.2.1 Identity & Structure

| Key | Format | Source | Notes |
|---|---|---|---|
| `id:<n>` | positive integer | `~/.todo/.idseq` | Global monotonic; see D3 |
| `p:<n>` | positive integer | hand-edited or addon | Parent task ID; see sub-item section |

**ID allocation.**  `~/.todo/.idseq` holds a single integer (the last
allocated ID).  Incrementing and reading is done atomically with `flock` or
equivalent:

```sh
id=$(flock ~/.todo/.idseq sh -c 'n=$(cat ~/.todo/.idseq); echo $((n+1)) > ~/.todo/.idseq; echo $((n+1))')
```

**Archive and `p:` references.**  `todo.sh archive` moves completed tasks from
`todo.txt` to `done.txt`.  A task with `p:42` where task 42 is now in
`done.txt` is **not** an orphan — `resolve 42` finds it in `done.txt`.  An
orphan is a `p:` value for which `resolve` returns nothing (the parent was
deleted rather than archived, or its `id:` was corrupted).  `lint` reports
orphans and offers to remove the dangling `p:` key.

**Parent deleted or completed.** When a parent is completed, its `p:<id>`
children are not automatically completed.  This is intentional: sub-tasks may
outlive a parent milestone.  `lint --orphans` flags children of deleted
parents.  The user decides: complete them, re-parent them, or remove `p:`.

#### 1.2.2 Dates

| Key | Format | Standard | Notes |
|---|---|---|---|
| `due:YYYY-MM-DD` | ISO 8601 date | [CORE] `do` filter | Maps to `dueDateTime` |
| `t:YYYY-MM-DD` | ISO 8601 date | [CONVENTION: topydo] | Threshold / defer-until; local only |
| `rem:YYYY-MM-DDTHHMM` | Date + `T` + 4-digit time (no colon) | [ADDON] | See rationale below |

**Reminder format rationale.**  `rem:2026-08-09T1430` places the full
datetime in a single colon-free token after `rem:`.  Any splitter that reads
only the first `:` yields `key=rem`, `value=2026-08-09T1430` — correct.  ISO
8601 extended time (`14:30`) introduces a second colon; naive splitters then
yield `value=2026-08-09T14`, silently truncating the minutes.  The compact
`HHMM` format (`1430`) eliminates the second colon entirely.  The `sync`
addon expands this to `2026-08-09T14:30:00.0000000` when writing to Graph.

#### 1.2.3 GTD Lifecycle

**Single `s:` key vs. multiple boolean flags.**

A single mutually-exclusive key (`s:next`, `s:wait`, `s:someday`, `s:blocked`)
is chosen over multiple independent flags (`next:1`, `wait:1`, etc.) for three
reasons:

1. GTD states are mutually exclusive.  Multiple flags invite contradictions
   (`s:next s:someday`) that silently exist in a text file until lint runs.
2. `grep 's:next' todo.txt` is unambiguous.  `grep 'next:1'` matches any key
   that contains the substring.
3. A single key is trivially validated: the value must be one of four strings.

| Key | Value | Semantics |
|---|---|---|
| `s:` | `next` | In the Next Actions list |
| `s:` | `wait` | Waiting on someone or something |
| `s:` | `someday` | Someday/Maybe — not active |
| `s:` | `blocked` | Blocked by another task or event |
| `wait:` | `@person` | Who/what you are waiting on (used with or without `s:wait`) |
| `e:` | `low\|med\|high` | Energy / effort estimate |
| `@context` | — | Native todo.txt context tag |

`wait:` and `s:` are independent: you can have `s:next wait:@alice` if you
need to do something once Alice responds, or `s:wait wait:@alice` if the task
is fully blocked on her reply.

**To Do mapping:** `s:`, `wait:`, and `e:` are dropped (no native fields).
All three survive round-trips through the local text file; they are invisible
to To Do.

### 1.3 Sub-item Encoding

The problem statement rules out `step:<index>:<text>` because the value would
contain spaces, which violates the spec.  Three legal alternatives:

| Option | Example | Round-trip to To Do | `grep`-ability | Editor ergonomics |
|---|---|---|---|---|
| **A. Separate task with `p:<id>`** | `Buy milk p:7 id:12` | Children sync as independent tasks; no hierarchy in To Do UI | `grep 'p:7'` → all children | Natural; each step is a full task line |
| B. Slug value `step:n:hyphen-text` | `step:1:buy-milk` | Stored in task body; formatting lost | `grep 'step:1'` works | Hyphenation destroys natural language |
| C. Sidecar file `steps/id.txt` | `~/.todo/logbook/steps/7.txt` | Out-of-band; never sent | Requires `cat` | Invisible in editor; breaks grep |

**Choice: Option A.**

Children are first-class task lines.  Every todo.txt operation (`do`, `del`,
`pri`, `move`) applies without modification.  The parent-child relationship
is expressed solely by `p:<parent-id>`.  Sync creates children as independent
To Do tasks — the parent-child link is visible only in the local file.

**Lossiness acknowledged:** MS To Do `checklistItems` (single-level, no dates,
no priorities) are not used for children.  Children that arrive from To Do as
`checklistItems` during a pull are written as new task lines with `p:<parent-id>`
and assigned local IDs.

---

## Decision Log

Each entry follows: *options considered → choice → consequence*.

**DL-1: MyDay representation.**
Options: `+MyDay` project tag; `myday:YYYY-MM-DD` key; `@myday` context tag.
Choice: `myday:YYYY-MM-DD`.
Consequence: `listproj` is clean; the date is self-documenting; `@myday`
would have polluted the context namespace instead.  No To Do mapping is
possible; this is explicitly declared as local-only.

**DL-2: Star vs. priority.**
Options: `(A)` = `importance:high`; `star:1` independent of priority; no
importance field at all.
Choice: `star:1` independent of `(A)`–`(Z)`.
Consequence: the priority ladder is preserved for local sorting; importance
round-trips faithfully through To Do; there is no automatic coupling, so a
user who sets `(A)` but not `star:1` gets `importance:normal` in To Do —
intentional.

**DL-3: Reminder format.**
Options: `rem:YYYY-MM-DDTHH:MM` (ISO extended, second colon); `rem:YYYY-MM-DDTHHMM`
(compact, no second colon).
Choice: `YYYY-MM-DDTHHMM`.
Consequence: all splitters produce a correct parse; topydo, todo.txt-rs, and
grep all see a clean `key:value` with a space-free value.

**DL-4: GTD state encoding.**
Options: single `s:` key with four allowed values; four independent boolean
keys (`next:1`, `wait:1`, etc.).
Choice: single `s:` key.
Consequence: states are mutually exclusive by construction; `lint` validates
the value set; grep is unambiguous; adding a fifth state requires only a new
allowed value.

**DL-5: Sub-item encoding.**
Options: `step:n:text` (invalid — spaces); `step:n:slug` (lossy); separate
task with `p:id` (full feature set); sidecar file (breaks grep).
Choice: separate task with `p:id`.
Consequence: children have their own due dates, priorities, and IDs; they sync
as independent To Do tasks; the parent-child link is local-only; no MS To Do
`checklistItems` are used.

**DL-6: `wait:` key vs. `@waiting` context.**
Options: `@waiting` context tag; `wait:@person` key.
Choice: `wait:@person`.
Consequence: the person being waited on is recorded explicitly; `grep 'wait:@alice'`
finds all tasks delegated to Alice; `@waiting` would have appeared in context
listings with no person attached.

**DL-7: ID allocation mechanism.**
Options: per-file counter (breaks D3); namespaced `id:logbook-101` (breaks
D3); global `~/.todo/.idseq` with `flock`.
Choice: `~/.todo/.idseq` with `flock`.
Consequence: IDs are globally unique; `resolve` can locate any task without
knowing which file it is in; the cost is one `flock`-guarded file write per
new task.

**DL-8: Recurrence handling.**
Options: `rec:1d` key on the task line; store recurrence pattern in `map.tsv`;
ignore recurrence entirely.
Choice: store in `map.tsv` extra column; no key on the task line.
Consequence: the todo.txt line remains spec-conformant (no invented `rec:`
key with complex value); recurrence state survives sync restarts; a recurring
task completing in To Do is handled as a new remote task on the next pull (see
§3.3).

---

## Pattern Index

Patterns to be expanded on request.  Each will follow the format specified in
the problem statement (Intent, Context, Problem, DSL command, Preconditions,
Before, After, Sync effect, To Do mapping, Google note, Failure modes, Verify).

| # | Pattern name | Tier | Status |
|---|---|---|---|
| P01 | **Capture** | Create | [ADDON] `todo.sh capture <text>` |
| P02 | **Resolve** | Utility | [ADDON] `todo.sh resolve <id>` |
| P03 | **Lint** | Utility | [ADDON] `todo.sh lint [--fix] [--orphans]` |
| P04 | **ID Backfill** | Utility | Part of P03 (lint) |
| P05 | **Complete-with-Children** | Execute | [ADDON] `todo.sh done` extended |
| P06 | **Archive-Orphan-Repair** | Execute | [ADDON] post-archive hook or lint pass |
| P07 | **Move-to-Logbook** | Execute | [CORE] `todo.sh move` + sync create |
| P08 | **Move-from-Logbook** | Execute | [CORE] `todo.sh move` + sync delete |
| P09 | **Sync-Pull** | Sync | [ADDON] `todo.sh sync pull` |
| P10 | **Sync-Push** | Sync | [ADDON] `todo.sh sync push` |
| P11 | **Sync-Conflict** | Sync | [ADDON] conflict resolution policy |
| P12 | **Recurrence** | Sync | [ADDON] recurring task completion from To Do |
| P13 | **Delete-Threshold** | Sync | [ADDON] D10 bulk delete guard |
| P14 | **Bulk-Inbox-Triage** | Review | [ADDON] `todo.sh triage` |
| P15 | **Reparent** | Execute | [ADDON] `todo.sh reparent <child-id> <new-parent-id>` |

---

## Lossiness Matrix

Every construct that cannot survive a round-trip through Microsoft To Do,
with its degradation rule.

| Construct | MS To Do field | Direction | Lossiness | Degradation rule |
|---|---|---|---|---|
| `(A)`–`(Z)` priority | none | push | **Full loss** | Not written to To Do; priority survives in `todo.txt` only |
| `myday:YYYY-MM-DD` | none | both | **Full loss** | Not synced; local filter alias `alias myday="grep myday:$(date +%F)"` |
| `t:YYYY-MM-DD` threshold | none | both | **Full loss** | Not synced; used by topydo and local filter scripts only |
| `e:low\|med\|high` energy | none | both | **Full loss** | Not synced; local only |
| `wait:@person` | none | both | **Full loss** | Not synced; local only |
| `s:wait\|someday\|blocked` | none | both | **Full loss** | Not synced; local only |
| `p:<id>` parent link | none | push | **Full loss** | Children sync as independent tasks; no hierarchy in To Do |
| Recurrence pattern | `recurrencePattern` | pull | **Partial loss** | Pattern stored in `map.tsv`; completing in To Do spawns new remote task; handled by P12 |
| `checklistItems` (pull) | `checklistItems` | pull | **Partial loss** | Converted to child task lines with `p:<parent-id>`; checked-off items become completed task lines |
| `@context` tags | none | push | **Full loss** | Kept in title text; not a To Do concept; survive round-trip as literal text |
| `+project` tags | none | push | **Full loss** | Kept in title text; survive round-trip as literal text |
| Creation date (second field) | `createdDateTime` (read-only) | pull | **Partial loss** | Pulled once on create; `createdDateTime` cannot be set via PATCH |
| `star:1` | `importance: high` | both | **Round-trips** | — |
| `due:YYYY-MM-DD` | `dueDateTime` | both | **Round-trips** | — |
| `rem:YYYY-MM-DDTHHMM` | `reminderDateTime` | both | **Round-trips** | Format expansion handled by sync addon |
| Completion `x ` | `status: completed` | both | **Round-trips** | — |
| Completion date | `completedDateTime` | both | **Round-trips** | — |
| Task title text | `title` | both | **Round-trips** | Context and project tags survive as literal text |

---

## Part 2 — Action Layer

Actions are grouped into three tiers: **Capture/Create**, **Clarify/Execute**,
and **Workflow/Review**.

### 2.1 Tier 1 — Capture / Create

#### `capture` [ADDON]

**Preconditions:** `$TODO_FILE` is writable; `~/.todo/.idseq` exists and is
writable (created on first run with value `0`).

**Behaviour:**

1. Allocate next ID from `~/.todo/.idseq` (atomic `flock`).
2. Append `<text> id:<n>` to `$TODO_FILE`.  Creation date is prepended if
   `$DATE_ON_ADD=1` in config.
3. If `$TODO_FILE` is inside `logbook/`, mark the new row in `map.tsv` with
   `state=new` so the next sync push creates it remotely.
4. **Idempotency:** a second `capture` of the same text is a new task with a
   new ID.  There is no deduplication.

**Failure:** if `$TODO_FILE` is unwritable, print error and exit 1.  Never
partially write.

**Scope:** operates on the current `$TODO_FILE` (one list).

#### `todo.sh add` [CORE]

Native `todo.sh add` can be used directly.  The only difference: `capture`
also allocates and appends `id:<n>`.  After a plain `add`, the next `lint --fix`
run backfills the missing ID.

### 2.2 Tier 2 — Clarify / Execute

#### `todo.sh do` [CORE] — Completion with child handling

`todo.sh do <n>` marks task `<n>` complete.  Extended behaviour:

- If the task has children (`grep "p:<id>"` in the same file), `lint --fix`
  run afterward flags open children.  The user decides; no automatic cascade.
- In `logbook/`, the next sync push sets `status: completed` on the remote task.
- `todo.sh archive` moves the completed line to `done.txt`.  Children with
  `p:<completed-id>` are then resolvable via `resolve` (done.txt is searched).

#### `todo.sh move` [CORE] — Move into / out of logbook

Moving a task **into** `logbook/`:

1. `todo.sh move <n> logbook` moves the line.
2. The task has no `map.tsv` row → sync treats it as new (D8 row 3).
3. If the task has a stale map row (D9), `lint` removes it before the next sync.

Moving a task **out of** `logbook/`:

1. `todo.sh move <n> <other-list>` moves the line.
2. The `map.tsv` row remains temporarily.  On the next sync, the task is
   missing from the directory → D8 row 1 → delete remotely.
3. The local task survives in `<other-list>`; only the remote copy is deleted.

#### `reparent` [ADDON]

`todo.sh reparent <child-id> <new-parent-id>`

- Validates both IDs exist via `resolve`.
- Rewrites `p:<old>` to `p:<new>` on the child's line using `sed -i`.
- Updates `last_hash` in `map.tsv` so the next sync sees the change and PATCHes
  the remote task title (the `p:` key survives in the title text).
- **Idempotency:** running twice is a no-op if the parent is already correct.

### 2.3 Tier 3 — Workflow / Review

#### `triage` [ADDON]

`todo.sh triage`

Presents tasks in `$TODO_FILE` with no `id:` (not yet captured cleanly), then
tasks with no `s:` and no `due:` (unprocessed inbox items), one at a time.
For each, prompts: `(n)ext (s)omeday (d)ue=YYYY-MM-DD (p)roject (skip)`.

- **Preconditions:** `$TODO_FILE` readable; stdin is a terminal.
- **Scope:** one list only.
- **Idempotency:** skipped tasks are re-presented on the next run.
- **Failure:** if stdin is not a terminal, print usage and exit 1.

#### `lint` [ADDON]

See §1 D11 and Pattern P03.  Full specification in §4.

#### `resolve` [ADDON]

See D11 and Pattern P02.  Full specification in §4.

---

## Part 3 — Sync Layer

### 3.1 Change Detection Without mtimes

**Problem.** `mtime` is unreliable: editors touch files, rsync resets it,
`todo.sh` rewrites the whole file on every `add`/`do`/`del`.

**Solution.** A per-line canonical hash stored in `map.tsv` at the time of the
last successful sync.  On the next sync run, the hash of the current line is
compared to the stored hash.  A mismatch means a local edit.

### 3.2 Canonicalization

The hash input is a canonical string derived from the task line by applying
these transformations in order, then passing the result to `md5sum`:

1. **Strip the `id:` key.**  It carries no content; its presence is assumed.
2. **Strip the `star:` key.**  It is synced via `importance`; hashing it
   separately would double-count it.
3. **Normalise whitespace.**  Collapse runs of spaces to a single space; trim
   leading and trailing spaces.
4. **Sort remaining `key:value` tokens lexicographically,** leaving the
   human-readable text (everything that is not a `key:value` token, `(X)`,
   `x `, or a date field) in its original position.
5. **Hash the result with `md5sum`.**

Rationale: sorting keys means that `due:2026-08-10 s:next` and
`s:next due:2026-08-10` produce the same hash, so an editor that reorders keys
does not appear as an edit to the sync engine.  Stripping `id:` and `star:`
prevents those fields from triggering spurious hash mismatches.

```sh
canonical() {
    local line="$1"
    # Strip id: and star: tokens
    line=$(echo "$line" | sed 's/ id:[^ ]*//g; s/ star:[0-9]*//g')
    # Normalise whitespace
    line=$(echo "$line" | tr -s ' ' | sed 's/^ //; s/ $//')
    echo "$line" | md5sum | cut -d' ' -f1
}
```

### 3.3 D8 State Table — Full Reconciliation Pass

The reconciliation pass runs once per `sync` invocation, after fetching the
delta from MS To Do.

```
For each row in map.tsv:
    local_present  = task line with id:<local_id> exists in logbook/ (todo.txt or done.txt)
    remote_present = task GUID appears in delta snapshot (not @removed)
    hash_match     = canonical(current line) == stored last_hash

    if in_map and NOT local_present and remote_present:
        → local delete: DELETE /me/todo/lists/{lid}/tasks/{msft_id}
          remove map row

    if in_map and local_present and remote_present:
        local_changed  = NOT hash_match
        remote_changed = delta shows changed fields since last deltaLink

        if local_changed and NOT remote_changed:
            → push: PATCH remote with local fields
              update last_hash

        if NOT local_changed and remote_changed:
            → pull: rewrite local line with remote fields
              update last_hash

        if local_changed and remote_changed:
            → conflict: local wins
              save remote version to logbook/conflicts/<id>.txt
              push local to remote
              update last_hash

        if NOT local_changed and NOT remote_changed:
            → no-op

For each task in logbook/ with no map row:
    → new local task: POST to remote
      add map row with new msft_id and last_hash

For each @removed entry in delta with a map row:
    → remote delete: check local_present
      if local_present:
          task was deleted remotely while local copy exists
          write local line to logbook/conflicts/<id>.txt
          remove local line
      remove map row

For each task in delta with no map row and not in local files:
    → new remote task: write new task line to logbook/todo.txt
      allocate local id:, add map row
```

**Recurrence (P12).**  A recurring task in To Do appears as `@removed` followed
by a new task GUID when completed remotely.  The sync engine:

1. Detects the `@removed` entry and removes the old map row.
2. Detects the new task GUID (same title) with no map row.
3. Creates a new local task line with a new `id:` and `p:` preserved.
4. Stores the recurrence pattern (from `map.tsv` extra column) on the new line
   as a comment — **not** as a key:value, since there is no spec-conformant
   recurrence key.  A future `rec:` extension may be added when a standard
   emerges.

### 3.4 Deletion Threshold (D10)

**Default threshold:** the lower of **10 tasks** or **20% of the list** (by
task count at the start of the sync run), whichever triggers first.

**Configuration** (in `$TODO_DIR/.list-meta` or `todo.cfg`):

```sh
SYNC_DELETE_ABSOLUTE=10   # stop if more than this many deletes
SYNC_DELETE_FRACTION=0.20 # stop if deletes exceed this fraction of list
```

**Behaviour when threshold is exceeded:**

1. Print the exact task lines that would be deleted, one per line, with file
   path and line number.
2. Print: `Sync aborted: N deletions exceed threshold (max M or P%).`
3. Print: `Re-run with --force-delete to proceed, or --dry-run to review.`
4. Exit 2 (distinct from exit 1 for errors).

**Flags:**

| Flag | Behaviour |
|---|---|
| `--dry-run` | Print all planned operations; make no changes; exit 0 |
| `--force-delete` | Override threshold; proceed with all deletions |
| `--max-delete=N` | Override `SYNC_DELETE_ABSOLUTE` for this run |

### 3.5 Conflict Policy

When both the local line and the remote task have changed since `last_hash`:

1. **Local wins.**  The local line is pushed to the remote.
2. The remote version (as received in the delta) is written to
   `logbook/conflicts/<local-id>-<timestamp>.txt` in a human-readable format.
3. No field-level merge is attempted.
4. `lint` reports the existence of unresolved conflict files.

### 3.6 Secrets

OAuth tokens (client ID, client secret, access token, refresh token) **must
not** be stored under `~/.todo/` if any part of that tree is version-controlled
(e.g. Dropbox, git, Nextcloud sync).

**Required location:** `~/.config/todo-sync/credentials` (mode `0600`).

```sh
# ~/.config/todo-sync/credentials
MSFT_CLIENT_ID=...
MSFT_CLIENT_SECRET=...
MSFT_ACCESS_TOKEN=...
MSFT_REFRESH_TOKEN=...
MSFT_TOKEN_EXPIRY=...   # Unix timestamp
```

The `sync` addon sources this file at startup and refuses to run if the file
is world-readable (`stat -c %a` != `600`).

---

## Part 4 — Pattern Library

### Pattern: Resolve

**Intent:** Locate a task line by its `id:` value, searching both `todo.txt`
and `done.txt` in every list directory.

**Context:** Any operation that needs to follow a `p:<id>` link, confirm a
task exists, or find which file a task lives in.

**Problem:** Without `resolve`, finding a task by ID requires knowing which
file it is in.  After `archive`, a parent moves to `done.txt`; without
searching both files, `p:` links appear broken.

**DSL command:** `todo.sh resolve <id>` [ADDON]

**Preconditions:** `$TODO_DIR` is set and readable; `id` is a positive integer.

**Before:** `~/.todo/logbook/todo.txt` contains:

```
(A) Write report due:2026-08-10 id:7
Buy milk p:7 id:12
```

**After:** (no file changes; output only)

```
~/.todo/logbook/todo.txt:1: (A) Write report due:2026-08-10 id:7
```

**Sync effect:** none.

**To Do mapping:** none; local lookup only.

**Google note:** n/a.

**Failure modes:**
- ID not found in any file → exit 1, print `resolve: id:<n> not found`.
- ID found in multiple lines (duplicate) → print all matches, exit 2.
- `$TODO_DIR` unreadable → exit 1, print error.

**Verify:**

```sh
todo.sh resolve 7 | grep -q 'todo.txt:1:'
```

---

### Pattern: Lint

**Intent:** Validate and repair `todo.txt` and `done.txt` files; backfill
missing `id:` keys; report duplicates; flag dangling `p:` references; GC
orphan `map.tsv` rows.

**Context:** Run after any batch edit, after `archive`, and as a pre-sync
check.

**Problem:** Hand-edited files can lose `id:` keys, accumulate duplicates, and
leave `p:` references pointing at deleted tasks.  The sync engine depends on
`id:` uniqueness; orphan map rows cause phantom pushes or deletions.

**DSL command:** `todo.sh lint [--fix] [--orphans]` [ADDON]

**Preconditions:** `$TODO_DIR` and `$DONE_FILE` are readable.  `--fix` requires
write permission on both files and `~/.todo/.idseq`.

**Before:** `~/.todo/logbook/todo.txt` contains:

```
Write report due:2026-08-10
Buy milk p:7 id:12
```

(First line is missing `id:`; `p:7` points at a task not found anywhere.)

**After (`--fix --orphans`):**

```
Write report due:2026-08-10 id:13
Buy milk p:7 id:12
```

Plus output:

```
WARN  p:7 not found in any file (logbook/todo.txt line 2: Buy milk p:7 id:12)
INFO  backfilled id:13 on line 1
```

**Sync effect:** updated `map.tsv` with `last_hash` for the newly-ID'd line;
orphan map rows (IDs in `map.tsv` with no matching task line) removed.

**To Do mapping:** none directly; side effect is that the next sync push will
create the newly-ID'd task remotely.

**Google note:** n/a.

**Failure modes:**
- Duplicate `id:` values — printed; `--fix` assigns a new ID to the later
  occurrence.
- `~/.todo/.idseq` unwritable — `--fix` aborts with exit 1.
- File is being written by another process — `flock` blocks; times out after
  30 s with exit 1.

**Verify:**

```sh
grep -c 'id:[0-9]' ~/.todo/logbook/todo.txt | grep -qx "$(wc -l < ~/.todo/logbook/todo.txt)"
```

---

### Pattern: Capture

**Intent:** Add a new task with an automatically allocated `id:` key in a
single atomic operation.

**Context:** Quick-capture during a meeting or at the keyboard; avoids a
subsequent `lint --fix` pass to backfill the ID.

**Problem:** `todo.sh add` does not assign `id:`.  Deferring ID assignment
creates a window during which the task is invisible to `resolve` and to the
sync engine.

**DSL command:** `todo.sh capture "<text>"` [ADDON]

**Preconditions:** `$TODO_FILE` writable; `~/.todo/.idseq` exists.

**Before:** `~/.todo/logbook/todo.txt` is empty.  `~/.todo/.idseq` contains `5`.

**After:**

```
# ~/.todo/logbook/todo.txt
2026-08-09 Write spec draft due:2026-08-15 s:next id:6
```

`~/.todo/.idseq` contains `6`.

**Sync effect:** if the file is in `logbook/`, a new `map.tsv` row is written
with `state=new` so the next `sync push` creates the task remotely.

**To Do mapping:** new remote task created on next push with `title` =
`Write spec draft due:2026-08-15 s:next`, `dueDateTime` = `2026-08-15`.

**Google note:** same mapping; Google Tasks also has a `due` field.

**Failure modes:**
- `~/.todo/.idseq` missing → created with value `0`; ID `1` is allocated.
- Race condition on `~/.todo/.idseq` → `flock` serialises; no duplicate IDs.
- Disk full → partial write → `todo.sh` truncates file to last complete line
  on next read; ID counter is already incremented.  Run `lint` to detect.

**Verify:**

```sh
tail -1 ~/.todo/logbook/todo.txt | grep -q 'id:6'
```

---

### Pattern: Move-to-Logbook (create remotely)

**Intent:** Move a task from a local-only list into `logbook/`, making it
visible to the sync engine and causing it to be created in Microsoft To Do.

**Context:** A task originally captured in a local list (e.g. `inbox/`) is
processed and belongs in the synced list.

**Problem:** Copying a line manually leaves the `id:` key intact but creates
no `map.tsv` entry, so the sync engine does not know to push it.

**DSL command:** `todo.sh move <n> logbook` [CORE]

Followed by: next `todo.sh sync push` [ADDON]

**Preconditions:** Task exists in `$TODO_FILE`; `logbook/` directory exists;
`logbook/map.tsv` is writable.

**Before:**

```
# ~/.todo/inbox/todo.txt  line 3:
Review Q3 budget id:9 due:2026-08-20
```

```
# ~/.todo/logbook/map.tsv  (no row for id:9)
```

**After:**

```
# ~/.todo/inbox/todo.txt  (line 3 removed)
# ~/.todo/logbook/todo.txt  (appended):
Review Q3 budget id:9 due:2026-08-20
```

```
# ~/.todo/logbook/map.tsv  (after sync push):
9    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    <hash>    synced
```

**Sync effect:** new row in `map.tsv`; remote task created.

**To Do mapping:** new To Do task with `title = "Review Q3 budget id:9
due:2026-08-20"`, `dueDateTime = 2026-08-20`.

**Google note:** same.

**Failure modes:**
- Stale `map.tsv` row for `id:9` (D9) — `lint` detects and removes before
  sync push; if not run, sync PATCHes a non-existent GUID and gets 404.
- `move` writes to `logbook/todo.txt` but sync has not run yet — task is
  visible locally but not remotely until next push.

**Verify:**

```sh
grep -q 'id:9' ~/.todo/logbook/todo.txt && ! grep -q 'id:9' ~/.todo/inbox/todo.txt
```

---

### Pattern: Move-from-Logbook (delete remotely)

**Intent:** Move a task out of `logbook/`, removing it from Microsoft To Do on
the next sync.

**Context:** A task is demoted to a local-only list or deleted.

**Problem:** Without a map row, the sync engine cannot know the GUID of the
remote task to delete.  The map row must outlive the local line until the
delete is confirmed.

**DSL command:** `todo.sh move <n> <dest-list>` [CORE] or `todo.sh del <n>` [CORE]

Followed by: next `todo.sh sync push` [ADDON]

**Preconditions:** Task has a `map.tsv` row; destination list directory exists.

**Before:**

```
# ~/.todo/logbook/todo.txt  line 2:
Old task id:5 s:someday
# ~/.todo/logbook/map.tsv:
5    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    <hash>    synced
```

**After:**

```
# ~/.todo/logbook/todo.txt  (line 2 removed)
# ~/.todo/logbook/map.tsv:  (row for id:5 removed after confirmed delete)
```

Remote task deleted.

**Sync effect:** `map.tsv` row removed after successful `DELETE` to Graph API.

**To Do mapping:** `DELETE /me/todo/lists/{lid}/tasks/{msft_id}`.

**Google note:** same.

**Failure modes:**
- Network failure during delete → map row retained; retry on next sync.
- Remote task already deleted (GUID returns 404) → map row removed; no error.
- D10 threshold triggered if bulk move-out exceeds limit → sync aborts; user
  must `--force-delete`.

**Verify:**

```sh
! grep -q 'id:5' ~/.todo/logbook/todo.txt && ! grep -q $'\t5\t' ~/.todo/logbook/.sync/map.tsv
```

---

### Pattern: Sync-Conflict

**Intent:** Handle the case where both the local line and the remote task have
changed since the last sync; apply local-wins policy; preserve the remote
version for manual review.

**Context:** Sync detects that `canonical(current_line) != last_hash` and the
delta also shows changes to the same task.

**Problem:** Field-level merge of a todo.txt line and a JSON Graph response is
not reliably possible with POSIX tools.  A deterministic policy is required.

**DSL command:** `todo.sh sync` [ADDON] (invoked automatically during reconciliation)

**Before:**

```
# ~/.todo/logbook/todo.txt:
(A) Write spec due:2026-08-10 star:1 id:7
# map.tsv last_hash: abc123  (hash of line at last sync)
# Remote title in delta: "Write spec due:2026-08-12" (due date changed remotely)
# Local line hash: def456 != abc123  (priority changed locally)
```

**After:**

```
# ~/.todo/logbook/todo.txt  (unchanged — local wins):
(A) Write spec due:2026-08-10 star:1 id:7
# ~/.todo/logbook/conflicts/7-20260809T014800.txt:
REMOTE at 2026-08-09T01:48:00Z:
  Write spec due:2026-08-12
LOCAL at 2026-08-09T01:48:00Z (kept):
  (A) Write spec due:2026-08-10 star:1 id:7
```

Remote task PATCHed to match local.

**Sync effect:** `last_hash` updated to hash of local line; conflict file written.

**To Do mapping:** `PATCH` remote to `title = "Write spec due:2026-08-10 star:1 id:7"`,
`dueDateTime = 2026-08-10`.

**Google note:** same policy; field names differ.

**Failure modes:**
- Conflict file directory unwritable → sync aborts rather than silently
  losing the remote version.
- Repeated conflicts on same task → multiple timestamped files; `lint`
  reports count.

**Verify:**

```sh
ls ~/.todo/logbook/conflicts/7-*.txt | grep -q '.'
```

---

### Pattern: Delete-Threshold

**Intent:** Abort a sync run that would delete more tasks than a configured
safety limit, requiring explicit user confirmation.

**Context:** D10 guard against accidental bulk deletion caused by a slipped
`dd` in the editor or a file-system mishap.

**Problem:** From the sync engine's perspective, a missing line is
indistinguishable from an intentional delete.  Without a threshold, a single
editor accident can destroy dozens of remote tasks.

**DSL command:** `todo.sh sync [--dry-run] [--force-delete] [--max-delete=N]` [ADDON]

**Preconditions:** Reconciliation pass has computed the delete set before any
network calls are made.

**Before:** 15 tasks are missing from `logbook/todo.txt`; list has 40 tasks;
`SYNC_DELETE_ABSOLUTE=10`, `SYNC_DELETE_FRACTION=0.20`.

**After (threshold triggered):**

```
Planned deletions (15):
  ~/.todo/logbook/todo.txt was: "(A) Write report id:3 ..."
  ...
Sync aborted: 15 deletions exceed threshold (max 10 or 20% of 40 = 8).
Re-run with --force-delete to proceed, or --dry-run to review.
```

Exit code 2.  No network calls made.

**Sync effect:** none until user re-runs with `--force-delete`.

**To Do mapping:** deletions proceed normally once confirmed.

**Google note:** same mechanism; threshold values are config, not code.

**Failure modes:**
- Legitimate bulk triage (user intentionally deleted many tasks) → user runs
  `--force-delete`; normal outcome.
- Threshold set to 0 — every delete is blocked; user must always pass
  `--force-delete`.

**Verify:**

```sh
todo.sh sync --dry-run 2>&1 | grep -q 'Planned deletions'
```
