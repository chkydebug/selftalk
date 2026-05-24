---
name: selftalk
description: Spawn a Claude, Gemini, or Codex CLI in a tmux session so the user can watch read-only while Codex drives it.
metadata:
  short-description: Drive another CLI in a detached tmux session
---

# selftalk

Ported from: `/home/koustav/.claude/commands/selftalk.md`.

The user has invoked `/selftalk` to drive another Claude, Gemini, or Codex CLI in a detached tmux session, with read-only spectator access for themselves.

This skill is designed for use from Codex by running shell commands via the available terminal tool (e.g. `functions.exec_command`).

## Argument parsing

`$ARGUMENTS` contains flags and optional prompt. Flags may appear in any order.

- `-c` -> target is `claude` (Claude Code CLI)
- `-g` -> target is `gemini` (Google Gemini CLI)
- `-x` -> target is `codex` (OpenAI Codex CLI)
- `-e` -> enable smart-route: probe Claude's `/usage` before spawning; if Claude's current session % is >80, spawn the fallback instead.
  Smart-route only applies to Claude: `-g` and `-x` alone never trigger a probe because they have no queryable usage endpoint.
- Anything after the flags (quoted or unquoted) is an initial prompt to send to the spawned CLI. Optional.
- If no target flag (`-c` / `-g` / `-x`) and no `-e`, stop and ask the user which they want. Do not guess.
- If `-e` is passed with no other target flag, treat primary as Claude and fallback as Gemini.
- If `-e` is set without `-c`, Claude is still the primary (the probe target is always Claude).

Examples:
- `/selftalk -c` -> spawn claude, idle, await instructions
- `/selftalk -g "what is your model"` -> spawn gemini directly, no probe
- `/selftalk -x "explain this codebase"` -> spawn codex directly, no probe
- `/selftalk -c -e -g "summarise this repo"` -> probe Claude; <=80% -> Claude, >80% -> Gemini
- `/selftalk -c -e -x "summarise this repo"` -> probe Claude; <=80% -> Claude, >80% -> Codex
- `/selftalk -e` -> same as `-c -e -g` (Claude-preferred, Gemini fallback), idle after resolves

## Recursive use (intentional)

Future use of this command is expected to be recursive: a spawned session may itself invoke `/selftalk` to drive another CLI.

Do not add anti-recursion checks unless the user explicitly asks. The fixed session name (`selftalk`) means a child invocation collides with the parent; that's intentional (kill+replace keeps state observable).

## What to do

### 1. Pre-flight

- Check that the target binary exists with `which claude`, `which gemini`, or `which codex`. If missing, stop and tell the user.
- Session name is fixed: `selftalk`. If a session with that name already exists (`tmux has-session -t selftalk 2>/dev/null`), kill it first.
- Working dir: `/tmp/selftalk` (create if absent). Keeps the spawned CLI from loading unrelated project context.

### 2. Smart-route (only if `-e` is set)

Before spawning the user's chosen target, probe Claude usage:

```bash
tmux new-session -d -s selftalk_probe -x 200 -y 50 -c /tmp/selftalk 'claude --dangerously-skip-permissions'
sleep 3
tmux send-keys -t selftalk_probe Enter
sleep 2
tmux send-keys -t selftalk_probe "/usage" Enter
sleep 4
tmux capture-pane -t selftalk_probe -p > /tmp/selftalk_probe.out
tmux kill-session -t selftalk_probe
```

Parse `/tmp/selftalk_probe.out` for the "Current session" line (looks like `... NN% used`) and extract the percentage.

- If session % > 80: set effective target to the fallback (`-g` -> gemini, `-x` -> codex, default -> gemini). Tell the user: "Smart-route: Claude session at NN% - routing to [fallback]."
- Else: keep effective target as Claude. Tell the user: "Smart-route: Claude session at NN% - using Claude."
- If `/usage` output is unparseable: default to Claude and tell the user the probe was inconclusive. Do not silently flip.

### 3. Spawn (with permissions bypassed)

```bash
# Claude
tmux new-session -d -s selftalk -x 200 -y 50 -c /tmp/selftalk 'claude --dangerously-skip-permissions'

# Gemini
tmux new-session -d -s selftalk -x 200 -y 50 -c /tmp/selftalk 'gemini -y --skip-trust'

# Codex
tmux new-session -d -s selftalk -x 200 -y 50 -c /tmp/selftalk 'codex --dangerously-bypass-approvals-and-sandbox'
```

The bypass flags are required; without them, each shell call inside the spawned CLI triggers a permission dialog that this driving session can't reliably handle.

Wait ~3s for the CLI to come up, then `tmux capture-pane -t selftalk -p` to confirm initial state.

### 4. Handle first-run prompts

Even with bypass flags, workspace-trust prompts may appear on first use in a new directory. Typically option 1 is pre-selected; send `Enter`.

### 5. Report the attach command to the user

Verbatim:

> Session up. To watch live (read-only) open a terminal and run: `tmux attach -r -t selftalk` - detach with `Ctrl-b d`.

### 6. Drive the session

Always use `tmux load-buffer` / `tmux paste-buffer` for prompts, then send Enter separately. Avoid the combined `send-keys "text" Enter` form; some CLIs treat that Enter as a newline in the input box.

```bash
cat > /tmp/selftalk_prompt.txt <<'EOF'
<prompt body here>
EOF
tmux load-buffer -t selftalk /tmp/selftalk_prompt.txt
tmux paste-buffer -t selftalk
sleep 1
tmux send-keys -t selftalk Enter
```

Poll for idle:

- Claude / Gemini: look for `esc to cancel`
- Codex: look for `esc to interrupt`

```bash
# Claude / Gemini
until ! tmux capture-pane -t selftalk -p | grep -qE "esc to cancel"; do sleep 3; done
tmux capture-pane -t selftalk -p | tail -60

# Codex
until ! tmux capture-pane -t selftalk -p | grep -qE "esc to interrupt"; do sleep 3; done
tmux capture-pane -t selftalk -p | tail -60
```

### 7. Teardown

```bash
tmux send-keys -t selftalk Escape
sleep 1
tmux send-keys -t selftalk "/quit" Enter
sleep 2
tmux kill-session -t selftalk 2>/dev/null
```

Confirm with `tmux list-sessions 2>&1` ("no server running" if selftalk was the only one).

## Cost warning

Spawned sessions draw on the corresponding provider quotas. The `-e` smart-route probe itself spawns a temporary Claude session and runs `/usage`. Mention this on first spawn if the user hasn't already acknowledged it.

## Do not

- Do not run destructive commands in the spawned CLI (rm, git push, etc.) unless the user explicitly asks.
- Do not strip the bypass flags from spawn commands; without them the spawn is unusable from this driving session.

