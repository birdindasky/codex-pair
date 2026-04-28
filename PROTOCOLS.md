# Collaboration Protocols

This document defines the file-based protocols that Claude and Codex use to coordinate when running the `codex-pair` workflow. These protocols are deliberately simple — plain JSONL files in the project's `.claude/` directory — so they work without any new infrastructure.

There are three protocols:

1. **Worklog** — append-only progress log so each agent can see what the other has done.
2. **Pending commits** — Codex writes commit intent here; Claude validates and executes the actual `git commit`.
3. **venv detection** — convention for running Python commands inside the right virtual environment (sandbox-friendly).

---

## 1. Worklog

**File:** `<project>/.claude/worklog.jsonl`

Append one JSON line each time an agent starts or finishes a discrete unit of work. Both agents read the worklog before starting so they know what the other has done.

### Entry format

```json
{
  "ts": "2026-04-28T12:34:56Z",
  "agent": "claude" | "codex" | "<custom role>",
  "phase": "start" | "end",
  "action": "human-readable description of the unit",
  "files": ["path/to/changed1.ts", "path/to/changed2.ts"],
  "notes": "anything an outside reader would need to make sense of this entry"
}
```

### Rules

- **Append-only.** Never edit or delete prior entries — they are the audit trail.
- **One JSON object per line.** No commas between lines. This is JSONL, not JSON.
- **Read before writing.** When an agent starts a task, it reads the entire file first to understand prior state.
- **Write at start AND end.** Half-finished work is visible because there's a `start` entry without a matching `end`.
- **Files list is optional but encouraged.** It lets the other agent know what was touched without re-running `git diff`.

### When to use

- Whenever 2+ agents are working on the same project and one needs to know what the other did.
- Especially valuable for parallel work (e.g. `worktree` mode where each agent works in an isolated working tree).
- For solo work, the worklog is unnecessary overhead.

---

## 2. Pending commits

**File:** `<project>/.claude/pending-commits.jsonl`

Codex runs in a sandbox that blocks writes to `.git/`, so it cannot execute `git commit` itself. Instead, when Codex finishes a logical unit of work, it appends a commit intent to this file. Claude (which has git access) validates and executes the actual commit.

### Entry format

```json
{
  "ts": "2026-04-28T12:34:56Z",
  "agent": "codex",
  "files_to_stage": ["path/to/file1", "path/to/file2"],
  "commit_message": "subject line\n\nBody paragraph explaining what and why.",
  "why": "one-sentence trigger that produced this commit",
  "ready": true
}
```

### Rules

- **Codex never runs `git`** — not even `git status`. The sandbox enforces this.
- **`ready: true` means "Claude can execute this now."** If `ready: false`, Codex is still adjusting; Claude waits.
- **`files_to_stage` is explicit, not a wildcard.** Codex lists every file that should be in the commit. Claude rejects entries that try to stage files Codex didn't actually modify.
- **`commit_message` is the final message.** Subject line ≤ 72 chars, blank line, then body. Claude does not rewrite the subject — but Claude may refuse to commit if the message is misleading.
- **Claude validates before committing.** Specifically:
  - Diff matches the listed files (no extra, no missing).
  - Commit message is not empty / placeholder / contains the word "TODO" in the subject.
  - No destructive flags would slip in (e.g. force, skip-hooks).
- **One intent → one commit.** If Codex wants 3 commits, it writes 3 entries. Claude executes them in order.

### Why not just give Codex git access?

The sandbox restriction is a feature, not a bug. It forces a checkpoint where Claude (and by extension the user) reviews every commit before it lands. This is the user's only real chance to catch a bad commit, since they cannot read the code themselves.

---

## 3. venv detection

When Codex needs to run Python commands inside a project, the sandbox's system Python does **not** have project dependencies installed. Naively running `python` or `pytest` will either fail with `ImportError` or use the wrong package versions.

### Detection ladder (first hit wins)

| If this exists | Run commands as |
|---|---|
| `<project>/venv/bin/activate` | `source <project>/venv/bin/activate && <cmd>` |
| `<project>/.venv/bin/activate` | `source <project>/.venv/bin/activate && <cmd>` |
| `<project>/uv.lock` | `uv run <cmd>` |
| `<project>/poetry.lock` | `poetry run <cmd>` |
| `<project>/Pipfile.lock` | `pipenv run <cmd>` |

### If none of the above exist

**Stop and report.** Do not pip-install dependencies into the sandbox — the install will succeed locally but the resulting environment won't match the project's actual runtime, leading to false test results.

### Why this matters

Pythagorean Python error: dependencies installed in the wrong environment look identical to a working install until tests start failing for reasons that don't reproduce outside the sandbox. The detection ladder catches this before it becomes mysterious.

---

## File locations summary

```
<project>/
└── .claude/
    ├── worklog.jsonl          (append-only, both agents read+write)
    └── pending-commits.jsonl  (Codex writes, Claude consumes)
```

Both files are project-local, not global. They can be safely committed to git (they're audit logs) or gitignored (if you treat them as ephemeral). Most teams gitignore them.

---

## Why file-based, not API-based?

- Works on any machine without setting up infrastructure.
- Survives agent restarts — state is on disk.
- Trivially inspectable by humans (`cat`, `tail -f`).
- No coupling to any specific orchestration tool.
- Same protocols work whether agents run in CI, locally, or in a sandbox.

The cost is some boilerplate JSON. Worth it.
