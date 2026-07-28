---
name: tmux-control
version: 1.1.6
description: |
  Safe tmux pane operations for Claude Code: send commands, restart individual
  panes without killing others, inspect state. Built to dodge the multi-line
  shell-mangling bug that occurs when Claude Code's Bash tool pipes commands
  through tmux send-keys. Use when you need to: operate on a tmux session,
  restart a scheduler/monitor while preserving training panes, inspect what's
  running in each pane, or safely send a command to a specific pane. Use when
  asked to "restart tmux", "send command to tmux", "check tmux status", or
  "restart scheduler without killing training".
triggers:
  - tmux
  - restart pane
  - send command to pane
  - restart scheduler
  - tmux status
  - kill session guard
allowed-tools:
  - Bash
  - Read
---

# tmux-control — Safe tmux pane operations

## Why this skill exists

Claude Code's Bash tool occasionally **mangles multi-line commands** when they
flow through `tmux send-keys`. The target pane's shell ends up running garbage
like:

```
$ sleep1tmuxsend-keys-tpalm_training_agent:scheduler
bash: sleep1tmuxsend-keys...: command not found
```

What happened: the newlines in the Bash tool's `command1\nsleep 1\ncommand2`
got stripped, fusing `sleep 1` with the next `tmux` invocation into
`sleep1tmux...`. The pane's shell tried to execute that as a single command.

This skill routes every tmux send through a helper that:

1. **Writes the command to a temp file**, then uses `tmux load-buffer` +
   `tmux paste-buffer` so the shell never sees the command until pasted as
   a literal string.
2. **Atomically clears the pane's prompt** before sending the new command
   (`C-c` + `C-u` + `Enter` + `C-u`), so any garbage left from a previous
   bad send is wiped first.
3. **Always captures pane output to a file**, not stdout — sidesteps the
   broken `| tail -n N` parsing.

## The five scripts

All scripts live in `~/.claude/skills/tmux-control/bin/`. They are safe to
call directly from the Bash tool — they were designed for that.

All scripts strip obvious shell-redirect residue (`2>&1`, `>`, `&`, `|`,
etc.) from their argv before parsing, so you can safely invoke them via the
Bash tool even when the call picks up trailing redirect tokens. (Bare `1` /
`2` are NOT stripped — those are often legitimate numeric arguments.)

### 1. `tmux-status` — inspect sessions/panes without pipe issues

```bash
# Overview of all sessions
~/.claude/skills/tmux-control/bin/tmux-status

# One session + last 5 lines per pane
~/.claude/skills/tmux-control/bin/tmux-status palm_training_agent --tail 5

# Include matching python/training processes
~/.claude/skills/tmux-control/bin/tmux-status --procs
```

Writes the report to `/tmp/tmux-status.txt` and cats it. No `| tail | grep`
chains that break.

### 2. `tmux-exec` — send a command to a pane, safely

This is the **default way to send a command**. Never use raw
`tmux send-keys` from the Bash tool for anything non-trivial.

```bash
# Simple command
~/.claude/skills/tmux-control/bin/tmux-exec palm_training_agent:scheduler \
    'python tmux_scheduler.py --loop --interval 60'

# Multi-line command via stdin
~/.claude/skills/tmux-control/bin/tmux-exec palm_training_agent:monitor --stdin <<'EOF'
source .venv/bin/activate
cd /cache/richard/work/palm-agent
python tmux_scheduler.py --monitor-loop
EOF

# With cwd, longer wait, more output captured
~/.claude/skills/tmux-control/bin/tmux-exec mysession:0 \
    --cwd /path/to/project --wait 8 --lines 30 -- python -m mymodule
```

Options: `--cwd`, `--wait <sec>` (default 3), `--lines <n>` (default 20),
`--file <path>`, `--stdin`, `--bg` (don't capture after), `--no-clear`.

### 3. `tmux-restart-pane` — stop a process in one pane and restart it

The killer feature: restart the scheduler/monitor pane while GPU training
panes keep running. **Do not use `tmux_scheduler.py --stop`** when GPUs are
busy — it kills the whole session.

```bash
# Dry run — see what would be sent and which panes would be verified
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    palm_training_agent:scheduler --dry-run -- \
    python tmux_scheduler.py --loop --interval 60

# Restart scheduler in place
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    palm_training_agent:scheduler -- \
    python tmux_scheduler.py --loop --interval 60

# Restart monitor, verify GPU panes are still alive afterward
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    palm_training_agent:monitor \
    --verify palm_training_agent:gpu0 \
    --verify palm_training_agent:gpu1 \
    --verify palm_training_agent:gpu4 -- \
    python tmux_scheduler.py --monitor-loop
```

What it does:
1. Snapshots each `--verify` pane's **process tree** (via tmux `#{pane_pid}`
   + recursive ps walk). Records the set of descendant PIDs.
2. Sends `C-c` to the target pane.
3. Polls up to 10s for the target pane's process tree to become empty.
4. Sends second `C-c` if needed; aborts if still running (exit 5).
5. Calls `tmux-exec` to send the new command.
6. Re-walks each `--verify` pane's process tree; reports any **dead PIDs**
   (exit 4 if protected processes died during restart).

Exit codes: 0=success, 2=usage, 3=target not found, 4=protected pane
regressed, 5=target didn't stop.

**Detection mechanism** (v1.1): walks tmux `#{pane_pid}` recursively through
`ps -eo pid,ppid,etime,args`. Replaces the fragile md5-hash-of-pane-text
approach which had two failure modes (frozen process producing log lines
from another thread → hash changes → false "alive"; idle shell prompt → hash
stable → false "dead").

### 4. `check-tmux-destructive` — guard against accidental mass kill

A PreToolUse hook that blocks `tmux kill-session`, `tmux kill-server`,
`tmux kill-window`, `tmux kill-pane`, `tmux_scheduler.py --stop`, and similar
commands when training processes are detected in any tmux session. To install:

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/tmux-control/bin/check-tmux-destructive"
      }]
    }]
  }
}
```

When triggered, returns `permissionDecision: "ask"` with a warning that
lists running training processes and suggests `tmux-restart-pane` instead.

**Per-session descendant mapping** (v1.1): walks the process tree from each
tmux session's `#{pane_pid}` set, NOT global `ps`. Filters to training-like
processes only (`clsTrainer`, `palm_train.sh`, `palm_det_train.sh`,
`tmux_scheduler.py`, `DetTrainer`, `train.py`) so the dev shell session
containing Claude Code / node / MCP servers doesn't drown out the warning.
Identical worker processes (e.g. 4 dataloader workers per trainer) are
collapsed to `+N workers` suffixes. Typical runtime: 0.3-0.5s.

Direct invocation:
```bash
~/.claude/skills/tmux-control/bin/check-tmux-destructive "python tmux_scheduler.py --stop"
# Prints warning to stderr, exit 0
```

## Decision tree: which script do I need?

```
I need to...
│
├─ ...see what's running
│   └─ tmux-status [--procs] [--tail N]
│
├─ ...send a single command to a pane
│   └─ tmux-exec <target> '<command>'
│       NEVER use raw `tmux send-keys` from Bash tool
│
├─ ...restart a long-running process in one pane
│   (e.g. load new code in scheduler while training continues)
│   └─ tmux-restart-pane <target> [--verify <other_pane>] -- <command>
│
├─ ...stop the whole session
│   └─ ⚠️ STOP. Are there training processes running? Use tmux-status --procs.
│      If yes: do NOT use --stop. Use tmux-restart-pane on the offending pane.
│      The check-tmux-destructive hook should also catch this.
│
├─ ...kill a pane/window
│   └─ tmux-kill-pane <target>
│       Refuses if training-like processes are in the pane.
│       Use --force to override, --allow '<pattern>' to allowlist.
│       Use --dry-run to preview.
│
└─ ...mass-stop the whole session / kill-server
    └─ ⚠️ STOP. Are there training processes running? Use tmux-status --procs.
       If yes: do NOT use --stop. Use tmux-restart-pane on the offending pane.
       The check-tmux-destructive hook should also catch this.
```

### 5. `tmux-kill-pane` — safe `kill-pane` with pre-flight check

Refuses to kill a pane whose process tree contains training-like processes
(`clsTrainer`, `palm_train.sh`, `tmux_scheduler.py`, etc.) unless you pass
`--force`. Use this INSTEAD of raw `tmux kill-pane -t ...`.

```bash
# Refuses if training detected
~/.claude/skills/tmux-control/bin/tmux-kill-pane palm_training_agent:gpu4

# Kill anyway
~/.claude/skills/tmux-control/bin/tmux-kill-pane palm_training_agent:gpu4 --force

# Allow specific pattern (e.g. only kill if process matches 'clsTrainer')
~/.claude/skills/tmux-control/bin/tmux-kill-pane palm_training_agent:gpu4 --allow 'clsTrainer'

# Preview without killing
~/.claude/skills/tmux-control/bin/tmux-kill-pane palm_training_agent:gpu4 --dry-run
```

Exit codes: 0=killed (or dry-run), 2=usage, 3=target not found, 6=refused
(training detected, use --force).

## Hard-won lessons (do not repeat)

1. **Never pipe `tmux capture-pane | tail -n N` directly.** Sometimes the
   Bash tool merges my multi-line command with the tmux output and feeds the
   result back to the target pane. Always write output to a file, then cat.

2. **Do NOT use `printf '%q'` to quote args for tmux send-keys / paste-buffer.**
   `printf '%q'` produces shell-escape sequences like `python\ tmux_scheduler.py`
   (with literal backslashes). When pasted via `tmux paste-buffer`, the target
   shell sees those backslashes as literal characters and the command becomes
   garbage. Use plain `"${CMD_ARGS[*]}"` (space-joined) — Claude Code's Bash
   tool already parses the args correctly before invoking our script.

3. **Avoid `2>&1 | grep` etc. when invoking these helpers from the Bash tool.**
   In some interactions the trailing `2>&1` token has been observed appended
   to the command sent into the target pane (e.g. `python foo.py 2`). All
   scripts now strip obvious redirect residue (`2>&1`, `>`, `&`, `|`, etc.)
   from their argv before parsing, but you should still call helpers directly
   without redirection and read `/tmp/*.txt` outputs after.

4. **`C-u` alone doesn't clear a half-typed command.** If a previous
   send-keys left garbage in the prompt, the next send gets prepended with
   that garbage. Use the full sequence: `C-c` → `C-u` → `Enter` → `C-u`.
   `tmux-exec` does this for you.

5. **`tmux_scheduler.py --stop` is a sledgehammer.** It marks every running
   experiment as `failed` with reason `scheduler_stopped`, THEN kills the
   tmux session. You lose hours of GPU training. Only use when no GPUs are
   busy. Always check with `tmux-status --procs` first.

6. **Identifying "the scheduler pane" requires capturing it.** Window names
   can lie (an `attach` followed by `cd` doesn't rename the window). Use
   `tmux-status <session> --tail 5` and look for `[SCHEDULE]` /
   `[ MONITORING ]` / iteration markers.

7. **`tmux list-panes` over a window without panes returns the window's
   implicit single pane.** `tmux-status` handles this; manual code often
   doesn't.

8. **If you send a command and the pane shows "command not found" with your
   own keystrokes echoed back**, the prompt buffer is contaminated. Run
   `tmux-exec <target> true` (a no-op) to force a clean clear+send cycle.

9. **Beware the "leaked positional arg" bug.** Original `tmux-status` had
   `*) SESSION="$1"; shift;;` which meant any stray token (including a `2`
   from `2>&1`) would silently overwrite SESSION. Fixed: set SESSION only
   on the first positional; ignore subsequent ones. All scripts now go
   further and strip redirect residue from argv before the parse loop.

10. **Atomic paste: include `\n` inside the buffer, not as a separate
    `send-keys Enter`.** The previous tmux-exec did `paste-buffer` then
    `send-keys Enter`. Under some timings the shell saw the paste and the
    Enter as separate events, causing the command to be inserted without
    execution, then the next paste concatenated onto it. Fix: write the
    command + `\n` to the temp file, paste once, delete the buffer — no
    second send-keys.

11. **`tmux list-panes -t <session> -a` returns panes from ALL sessions, not
    just the named one.** The `-a` flag means "all sessions globally".
    Either filter the output by `#{session_name}` yourself, or use `-s`.
    `check-tmux-destructive` v1.1 filters via awk on `#{session_name}` for
    reliability across tmux versions.

12. **Heredoc + piped stdin conflict in `$(...)` substitution.**
    `ps -eo ... | python3 - <<'PYEOF'` inside a bash function called from
    `$(...)` returns empty: the heredoc feeds the script source to python's
    stdin, but the piped ps output *also* wants stdin — one of them loses.
    Fix: write ps output to a temp file, pass the path as argv:
    `ps -eo ... > "$psfile"; python3 - "$psfile" <<'PYEOF'`.

13. **`mapfile` cannot write to associative-array slots.**
    `mapfile -t BEFORE_OUTPUT["$p"] < <(...)` fails with "not a valid
    identifier". Use parallel arrays: one regular array of keys, plus two
    associative arrays indexed by the same key. Or just assign directly:
    `BEFORE_OUTPUT["$p"]="$(...)"` and walk the output with
    `while IFS= read -r line`.

14. **`|&` operator in case patterns.** A case alternative ending with
    `|&` (e.g. `'2>&1'|'>'|'&>'` written unquoted as `2>&1|>|&>`) parses as
    the bash pipe-stderr operator. Always single-quote each alternative:
    `'2>&1'|'>'|'1>'|'2>'|'&>'`.

15. **Filter to training-like processes when warning about kills.** A dev
    tmux session containing Claude Code / node / MCP servers easily has 30+
    descendants; warning about all of them drowns out the real concern.
    Filter to `clsTrainer`, `palm_train.sh`, `tmux_scheduler.py`, etc. and
    collapse identical worker commands via `+N workers` suffixes.

16. **`tmux-status` must exit non-zero when a named session is not found.**
    Originally it printed "(not found)" to the report file but exited 0,
    making it useless for shell-script guards like
    `if tmux-status mysession; then ...`. Fixed in v1.1.1: a NOT_FOUND
    flag is set in the body block, and the script exits 3 after printing
    the report.

17. **Hooks may run with a degraded PATH.** A PreToolUse hook
    (`check-tmux-destructive`) invoked by Claude Code's Bash tool can
    sometimes see an environment where `timeout` and `grep` are not on
    PATH. Use bash builtins where possible:
      - Replace `timeout 1 cat` with `read -t 1 -r` (no PATH dependency)
      - Replace `grep -qE` with `[[ =~ ]]` bash regex when `grep` is
        unavailable (check via `command -v grep`)
    Otherwise the hook silently fails to detect destructive commands.

18. **`tmux kill-window` is just as destructive as `kill-session`.**
    A window contains panes, each of which may have training processes.
    The original `DESTRUCTIVE_RE` only matched `kill-(session|server)` and
    `kill-pane`, missing `kill-window` entirely. This meant
    `tmux kill-window -t session:gpu0` would bypass the hook and silently
    kill GPU training. Fixed in v1.1.2: regex now matches
    `kill-(session|server|window|pane)`. Always include ALL tmux kill
    subcommands in destructive-pattern checks.

19. **`[[ "$var" -gt 0 ]]` triggers arithmetic evaluation under `set -u`.**
    If `$var` is a non-numeric string like `"abc"`, bash's `[[ ... -gt ... ]]`
    tries to evaluate it as an arithmetic expression, causing
    `abc: unbound variable` and exiting the script. This affected
    `tmux-status --tail` when a non-integer value was passed.
    Fixed in v1.1.3: validate `--tail` with `[[ "$TAIL" =~ ^[0-9]+$ ]]`
    at parse time, before any arithmetic comparison. Also guard `--out`
    against missing values with `[[ $# -ge 2 ]]`. Always validate numeric
    args with regex before using them in arithmetic context.

20. **`[[ $# -ge 2 ]]` alone is insufficient to guard flag values.**
    The guard catches the case where a flag is the *last* argument
    (e.g., `--wait` with nothing after it), but NOT the case where the
    next argument is another flag (e.g., `--wait --dry-run 'echo hi'`).
    In that case `$#=3`, so `$# -ge 2` passes, and `--dry-run` is
    captured as the `--wait` value. This caused a **critical production
    safety bug** in `tmux-kill-pane --allow --dry-run`: the `--dry-run`
    was eaten as the allow pattern, the dry-run protection was silently
    disabled, and the pane was actually killed. Fixed in v1.1.4: all
    flag-value guards now use `[[ $# -ge 2 && "$2" != -* ]]` — the
    value must not start with `-` (i.e., must not look like a flag).
    For numeric flags (`--wait`, `--lines`, `--tail`), the regex check
    `[[ "$var" =~ ^[0-9]+$ ]]` provides a second layer of defense.
    Always use `[[ $# -ge 2 && "$2" != -* ]]` for flag-value guards.

21. **TRAINING_RE must be kept in sync across scripts.**
    `check-tmux-destructive` and `tmux-kill-pane` both define
    `TRAINING_RE` to identify training-like processes. A duplicate
    `clsTrainer` entry in one script's regex was harmless (regex `|`
    tolerates duplicates) but violated the "must match" comment. Fixed
    in v1.1.4. When two scripts share a pattern constant, keep them
    byte-identical or extract to a shared file.

22. **The `-*|--` case branch eats legitimate command flags.**
    In `tmux-exec`, the parse loop had a `-*|--` branch to swallow
    unknown flags leaked by the Bash tool (e.g. `2>&1` residue). But
    after TARGET was set, any `-`-prefixed argument from the user's
    command (e.g. `python -c`, `grep -E`, `python --version`) would
    also hit this branch and be silently dropped. The command arrived
    at the target pane with flags missing.

    First fix attempt (v1.1.5): unconditionally treat ALL args after
    TARGET as command. This broke `target --cwd /path --wait 5 cmd`
    because script flags after target were no longer recognized.

    Final fix (v1.1.6): the `-*` branch checks whether TARGET is set.
    Before TARGET: eat unknown flags (redirect residue). After TARGET:
    pass them to CMD_ARGS (they belong to the user's command). Known
    script flags (`--cwd`, `--wait`, etc.) are always matched by their
    own case branches regardless of position.
    ```bash
    -*)
        if [[ -n "$TARGET" && "$CMD_MODE" == "args" ]]; then
            CMD_ARGS+=("$1"); shift;
        else
            shift;  # eat residue before target
        fi
        ;;
    ```
    Lesson: in parsers that accept `<target> <flags...> <command...>`,
    distinguish KNOWN flags (always handle) from UNKNOWN flags (context-
    dependent: eat before target, preserve after target).

23. **`--` after separator in tmux-restart-pane strips redirect residue
    by design.** The `--` handler in `tmux-restart-pane` calls
    `is_redirect_residue` on each arg, so `-- python foo.py 2>&1` will
    have `2>&1` stripped. This is intentional: the Bash tool frequently
    leaks `2>&1` as a trailing token, and in `--` context it's
    ambiguous whether the user meant the redirect literally or it was
    leaked. The trade-off favors safety (strip it) over fidelity
    (preserve it). If you need a literal redirect in the command, use
    `tmux-exec --file` or `--stdin` instead.

## Common workflows

### Workflow A: Restart scheduler/monitor after code change

```bash
# 1. Check current state
~/.claude/skills/tmux-control/bin/tmux-status palm_training_agent --tail 3 --procs

# 2. If GPUs are busy, restart ONLY the scheduler pane (preserve training)
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    palm_training_agent:scheduler \
    --verify palm_training_agent:gpu0 \
    --verify palm_training_agent:gpu1 \
    --verify palm_training_agent:gpu4 -- \
    python tmux_scheduler.py --loop --interval 60

# 3. Same for monitor
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    palm_training_agent:monitor -- \
    python tmux_scheduler.py --monitor-loop

# 4. Confirm
~/.claude/skills/tmux-control/bin/tmux-status palm_training_agent --tail 8 --procs
```

### Workflow B: Stop everything cleanly

```bash
# 1. Are there trainings? List them.
~/.claude/skills/tmux-control/bin/tmux-status --procs

# 2. If no trainings, safe to stop:
tmux kill-session -t palm_training_agent

# 3. If yes — either wait, or explicitly decide to lose progress.
#    The check-tmux-destructive hook will intercept this and require
#    confirmation.

# 4. To kill ONE pane (refuses if training is in it):
~/.claude/skills/tmux-control/bin/tmux-kill-pane palm_training_agent:gpu7
#   Add --force if you really mean it.
```

### Workflow C: Send a quick diagnostic command to a pane

```bash
# Read a config value, check a process, etc — without leaving the pane
# in a weird state
~/.claude/skills/tmux-control/bin/tmux-exec palm_training_agent:gpu0 \
    'tail -n 5 logs/exp123/train.log'
```

(Though usually you'd just `Read` the log file directly — this is for when
the pane has its own state you need to poke at.)
