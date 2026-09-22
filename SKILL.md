---
name: tmux-control
version: 2.0.0
description: |
  Safe, general-purpose tmux pane operations for Claude Code: inspect sessions,
  send commands, restart individual panes without killing others, and guard
  against destructive kills. Built to dodge the multi-line shell-mangling bug
  that occurs when Claude Code's Bash tool pipes commands through tmux
  send-keys. Protects long-running jobs (training, servers, workers) via a
  configurable PROTECTED_RE. Use when asked to "send command to tmux",
  "restart tmux pane", "check tmux status", "kill tmux pane/session safely",
  or when operating any long-running process inside tmux.
triggers:
  - tmux
  - tmux status
  - send command to pane
  - restart pane
  - kill pane
  - kill session guard
allowed-tools:
  - Bash
  - Read
---

# tmux-control — Safe, general-purpose tmux pane operations

## Why this skill exists

Claude Code's Bash tool occasionally **mangles multi-line commands** when they
flow through `tmux send-keys`. The target pane's shell ends up running garbage
like:

```
$ sleep1tmuxsend-keys-tmysession:scheduler
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

Additionally, every destructive path (kill pane / kill session) is guarded by
a configurable **PROTECTED_RE** so long-running jobs are not silently lost.

## Configuration (make it yours)

All scripts source `~/.claude/skills/tmux-control/config`. Edit it once per
machine/project:

```bash
# Which processes count as "long-running / important" (ERE):
PROTECTED_RE='train\.py|trainer|scheduler|monitor|jupyter|serve|uvicorn|celery|worker'

# What `tmux-status --procs` lists:
PROCS_RE="python|node|$PROTECTED_RE"

# Extra destructive command patterns your project uses (optional):
# EXTRA_DESTRUCTIVE_RE='myscript\.py\s+--stop\b'
```

Env override without editing the file: `TMUX_CONTROL_PROTECTED_RE='...'`.

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
~/.claude/skills/tmux-control/bin/tmux-status myproj --tail 5

# Include protected/python processes
~/.claude/skills/tmux-control/bin/tmux-status --procs
```

Writes the report to `/tmp/tmux-status.txt` and cats it. No `| tail | grep`
chains that break.

### 2. `tmux-exec` — send a command to a pane, safely

This is the **default way to send a command**. Never use raw
`tmux send-keys` from the Bash tool for anything non-trivial.

```bash
# Simple command
~/.claude/skills/tmux-control/bin/tmux-exec myproj:scheduler \
    'python scheduler.py --loop --interval 60'

# Multi-line command via stdin
~/.claude/skills/tmux-control/bin/tmux-exec myproj:worker --stdin <<'EOF'
source .venv/bin/activate
cd /path/to/project
python worker.py --loop
EOF

# With cwd, longer wait, more output captured
~/.claude/skills/tmux-control/bin/tmux-exec mysession:0 \
    --cwd /path/to/project --wait 8 --lines 30 -- python -m mymodule
```

Options: `--cwd`, `--wait <sec>` (default 3), `--lines <n>` (default 20),
`--file <path>`, `--stdin`, `--bg` (don't capture after), `--no-clear`.

### 3. `tmux-restart-pane` — stop a process in one pane and restart it

The killer feature: restart one pane (scheduler, worker, dev server) while
all other panes — and their long-running jobs — keep running. Never kill the
whole session just to reload one pane.

```bash
# Dry run — see what would be sent and which panes would be verified
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    myproj:scheduler --dry-run -- \
    python scheduler.py --loop --interval 60

# Restart a worker in place
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    myproj:worker -- \
    python worker.py --loop

# Restart scheduler, verify other panes are still alive afterward
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    myproj:scheduler \
    --verify myproj:gpu0 \
    --verify myproj:gpu1 \
    --verify myproj:api -- \
    python scheduler.py --loop --interval 60
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
`tmux kill-window`, `tmux kill-pane`, plus any `EXTRA_DESTRUCTIVE_RE` patterns
from `config`, when protected processes are detected in any tmux session. To
install:

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
lists running protected processes and suggests `tmux-restart-pane` instead.

**Per-session descendant mapping** (v1.1): walks the process tree from each
tmux session's `#{pane_pid}` set, NOT global `ps`. Filters to protected
processes only (per `PROTECTED_RE` in `config`) so the dev shell session
containing Claude Code / node / MCP servers doesn't drown out the warning.
Identical worker processes (e.g. 4 dataloader workers per trainer) are
collapsed to `+N workers` suffixes. Typical runtime: 0.3-0.5s.

Direct invocation:
```bash
~/.claude/skills/tmux-control/bin/check-tmux-destructive "tmux kill-session -t myproj"
# Prints warning to stderr, exit 0
```

### 5. `tmux-kill-pane` — safe `kill-pane` with pre-flight check

Refuses to kill a pane whose process tree contains protected processes (per
`PROTECTED_RE` in `config`) unless you pass `--force`. Use this INSTEAD of
raw `tmux kill-pane -t ...`.

```bash
# Refuses if a protected process is detected
~/.claude/skills/tmux-control/bin/tmux-kill-pane myproj:gpu2

# Kill anyway
~/.claude/skills/tmux-control/bin/tmux-kill-pane myproj:gpu2 --force

# Allow specific pattern (e.g. only kill if process matches 'worker.py')
~/.claude/skills/tmux-control/bin/tmux-kill-pane myproj:worker --allow 'worker.py'

# Preview without killing
~/.claude/skills/tmux-control/bin/tmux-kill-pane myproj:gpu2 --dry-run
```

Exit codes: 0=killed (or dry-run), 2=usage, 3=target not found, 6=refused
(protected process detected, use --force).

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
│   (e.g. load new code in a worker while other panes keep running)
│   └─ tmux-restart-pane <target> [--verify <other_pane>] -- <command>
│
├─ ...stop the whole session
│   └─ ⚠️ STOP. Are there long-running jobs? Use tmux-status --procs.
│      If yes: do NOT kill the session. Use tmux-restart-pane on the
│      offending pane. The check-tmux-destructive hook should also catch this.
│
├─ ...kill a pane/window
│   └─ tmux-kill-pane <target>
│       Refuses if protected processes are in the pane.
│       Use --force to override, --allow '<pattern>' to allowlist.
│       Use --dry-run to preview.
│
└─ ...mass-stop the whole session / kill-server
    └─ ⚠️ STOP. Are there long-running jobs? Use tmux-status --procs.
       If yes: do NOT kill. Use tmux-restart-pane on the offending pane.
       The check-tmux-destructive hook should also catch this.
```

## Hard-won lessons (do not repeat)

1. **Never pipe `tmux capture-pane | tail -n N` directly.** Sometimes the
   Bash tool merges my multi-line command with the tmux output and feeds the
   result back to the target pane. Always write output to a file, then cat.

2. **Do NOT use `printf '%q'` to quote args for tmux send-keys / paste-buffer.**
   `printf '%q'` produces shell-escape sequences like `python\ foo.py`
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

5. **Whole-session stop scripts are sledgehammers.** Any project script that
   "stops everything" (kills the tmux session) will mark running jobs dead
   and lose hours of work. Only use when no long-running jobs are active.
   Always check with `tmux-status --procs` first, and register such scripts
   in `EXTRA_DESTRUCTIVE_RE` so the hook warns about them.

6. **Identifying "which pane runs X" requires capturing it.** Window names
   can lie (an `attach` followed by `cd` doesn't rename the window). Use
   `tmux-status <session> --tail 5` and look at the pane contents.

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

15. **Filter to protected processes when warning about kills.** A dev
    tmux session containing Claude Code / node / MCP servers easily has 30+
    descendants; warning about all of them drowns out the real concern.
    Filter via `PROTECTED_RE` in `config` and collapse identical worker
    commands via `+N workers` suffixes.

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
    A window contains panes, each of which may have long-running jobs.
    The original `DESTRUCTIVE_RE` only matched `kill-(session|server)` and
    `kill-pane`, missing `kill-window` entirely. This meant
    `tmux kill-window -t session:win0` would bypass the hook and silently
    kill running jobs. Fixed in v1.1.2: regex now matches
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

21. **PROTECTED_RE must be kept in sync across scripts.**
    `check-tmux-destructive` and `tmux-kill-pane` both need the
    protected-process pattern. Duplicated constants drift. Since v2.0.0,
    both scripts source the shared `../config` file — do NOT inline the
    pattern in scripts anymore. When two scripts share a constant, extract
    it to a shared file.

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

24. **Keep project specifics in `config`, not in script code.** v1.x
    hardcoded training patterns (`clsTrainer`, `palm_train.sh`,
    `tmux_scheduler.py --stop`) across four scripts. Every new project
    meant editing five files. v2.0.0 moved all project-specific regexes
    into `~/.claude/skills/tmux-control/config` (with env-var overrides).
    When a skill is generically useful, the ONLY thing that should change
    per project is configuration.

25. **A pane's job may BE the pane_pid, not a child.** v1.x process-tree
    walks started at tmux `#{pane_pid}` and printed only DESCENDANTS. But
    when a pane is created as `tmux new-session -d 'python train.py'`, the
    shell execs into python, so pane_pid IS the job and the "tree" is
    empty — `tmux-kill-pane` reported "idle shell" and killed a live job,
    and `check-tmux-destructive` warned "no protected processes". Fixed in
    v2.0.0: the kill-guard and destructive-guard walks now include the root
    pane_pid itself. `tmux-restart-pane` intentionally does NOT include it
    in its stop-wait loop (after C-c the shell must remain, or the loop
    would never see an "empty" pane), but its `--verify` stage now reports
    a vanished pane as a death instead of silently passing.

## Common workflows

### Workflow A: Restart one pane after a code change (others keep running)

```bash
# 1. Check current state
~/.claude/skills/tmux-control/bin/tmux-status myproj --tail 3 --procs

# 2. If other panes are busy, restart ONLY the pane you need
~/.claude/skills/tmux-control/bin/tmux-restart-pane \
    myproj:scheduler \
    --verify myproj:gpu0 \
    --verify myproj:gpu1 \
    --verify myproj:api -- \
    python scheduler.py --loop --interval 60

# 3. Confirm
~/.claude/skills/tmux-control/bin/tmux-status myproj --tail 8 --procs
```

### Workflow B: Stop everything cleanly

```bash
# 1. Any long-running jobs? List them.
~/.claude/skills/tmux-control/bin/tmux-status --procs

# 2. If nothing important is running, safe to stop:
tmux kill-session -t myproj

# 3. If yes — either wait, or explicitly decide to lose progress.
#    The check-tmux-destructive hook will intercept this and require
#    confirmation.

# 4. To kill ONE pane (refuses if a protected process is in it):
~/.claude/skills/tmux-control/bin/tmux-kill-pane myproj:worker
#   Add --force if you really mean it.
```

### Workflow C: Send a quick diagnostic command to a pane

```bash
# Read a config value, check a process, etc — without leaving the pane
# in a weird state
~/.claude/skills/tmux-control/bin/tmux-exec myproj:api \
    'tail -n 5 logs/app.log'
```

(Though usually you'd just `Read` the log file directly — this is for when
the pane has its own state you need to poke at.)
