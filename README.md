# todo.txt-dsl

A modular DSL layered on Gina Trapani's **todo.txt** format, executed through
`todo.sh`, with one list synchronised to **Microsoft To Do**.

**Clients:** `todo.sh` + text editor only.  No mobile or GUI app.
**Sync target:** Microsoft To Do (single "Logbook" list).

---

## Quick start

```sh
# 1. Install todo.sh  (https://github.com/todotxt/todo.txt-cli)
# 2. Copy addons into your $TODO_ACTIONS_DIR
cp addons/* ~/.todo/actions/
chmod +x ~/.todo/actions/*

# 3. Copy and edit the example config
cp config/todo.cfg.example ~/.todo/config
$EDITOR ~/.todo/config

# 4. Create the global ID counter
echo 0 > ~/.todo/.idseq

# 5. Capture your first task
todo.sh capture "Write spec draft due:2026-08-15 s:next"

# 6. Validate your files
todo.sh lint
```

---

## Repository layout

```
.
├── SPEC.md                  Full specification (Parts 0–4)
├── README.md                This file
├── addons/
│   ├── capture              [ADDON] Quick-add a task with auto id:
│   ├── lint                 [ADDON] Validate, backfill, and repair
│   ├── resolve              [ADDON] Find a task by id: number
│   └── sync                 [ADDON] Sync logbook ↔ Microsoft To Do
└── config/
    └── todo.cfg.example     Example shell configuration
```

## Key schema (todo.txt lines)

```
(A) Write spec draft due:2026-08-15 s:next id:7
    ^   ^              ^            ^      ^
    |   text           due date     GTD    global ID
    priority
```

| Key | Format | Notes |
|---|---|---|
| `id:<n>` | positive integer | Global monotonic; from `~/.todo/.idseq` |
| `p:<n>` | positive integer | Parent task ID |
| `due:YYYY-MM-DD` | ISO 8601 date | Maps to `dueDateTime` in To Do |
| `t:YYYY-MM-DD` | ISO 8601 date | Threshold/defer [CONVENTION: topydo] |
| `rem:YYYY-MM-DDTHHMM` | compact datetime | Reminder; no second colon |
| `star:1` | boolean | Maps to `importance:high` in To Do |
| `s:next\|wait\|someday\|blocked` | enum | GTD state (mutually exclusive) |
| `wait:@person` | @-prefixed name | Delegation target |
| `myday:YYYY-MM-DD` | ISO 8601 date | Daily focus date (local only) |
| `e:low\|med\|high` | enum | Energy estimate (local only) |

See [SPEC.md](SPEC.md) for the full specification including all decisions,
the lossiness matrix, sync algorithm, and pattern library.