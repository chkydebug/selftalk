---
description: Spawn a Claude, Gemini, or Codex CLI in a tmux session so the user can watch read-only while I drive it.
allowed-tools: Bash, Read
---

The user has invoked `/selftalk` to drive another Claude, Gemini, or Codex CLI in a detached tmux session, with read-only spectator access for themselves.

## Argument parsing

`$ARGUMENTS` contains flags and optional prompt. Flags may appear in any order.

- `-c` → target is `claude` (Claude Code CLI)
- `-g` → target is `gemini` (Google Gemini CLI)
- `-x` → target is `codex` (OpenAI Codex CLI)
- `-e` → **enable smart-route**: probe Claude's `/usage` before spawning; if Claude's *current session* % is **>80**, spawn the fallback instead. The fallback is whichever other target flag is also set (`-g` → Gemini, `-x` → Codex). Smart-route only applies to Claude — `-g` and `-x` alone never trigger a probe because they have no queryable usage endpoint.
- Anything after the flags (quoted or unquoted) is an initial prompt to send to the spawned CLI. Optional.
- If no target flag (`-c` / `-g` / `-x`) and no `-e`, stop and ask the user which they want — do not guess.
- If `-e` is passed with no other target flag, treat primary as Claude and fallback as Gemini.
- If `-e` is set without `-c`, Claude is still the primary (the probe target is always Claude).

Examples:
- `/selftalk -c` → spawn claude, idle, await instructions
- `/selftalk -g "what is your model"` → spawn gemini directly, no probe
- `/selftalk -x "explain this codebase"` → spawn codex directly, no probe
- `/selftalk -c -e -g "summarise this repo"` → probe Claude; ≤80% → Claude, >80% → Gemini
- `/selftalk -c -e -x "summarise this repo"` → probe Claude; ≤80% → Claude, >80% → Codex
- `/selftalk -e` → same as `-c -e -g` (Claude-preferred, Gemini fallback), idle after resolves

## Recursive use — intentional, no guardrails

Future use of this command is expected to be **recursive**: a spawned Claude session may itself invoke `/selftalk` to drive another CLI, which may in turn spawn another, etc. **Do not add anti-recursion checks** (PID-depth limits, parent-session sniffing, refuse-if-spawned-by-selftalk heuristics) unless the user explicitly asks. The user has accepted the runaway-quota risk. The fixed session name (`selftalk`) does mean a child invocation would collide with the parent; that's a feature for now, not a bug to patch — it forces the child to kill+replace, which keeps state observable.

## What to do

### 1. Pre-flight

- Check that the target binary exists with `which claude`, `which gemini`, or `which codex`. If missing, stop and tell the user.
- Generate a unique session name: `SESSION_NAME="selftalk_$(date +%s)"`
- Working dir: `/tmp/selftalk` (create if absent). Neutral so the spawned CLI doesn't load unrelated project context.

### 2. Smart-route (only if `-e` is set)

Before spawning the user's chosen target, probe Claude usage:

```bash
PROBE_SESSION="selftalk_probe_$(date +%s)"
tmux new-session -d -s $PROBE_SESSION -x 200 -y 50 -c /tmp/selftalk 'claude --dangerously-skip-permissions'
sleep 3
# Dismiss any trust prompt (option 1 pre-selected)
tmux send-keys -t $PROBE_SESSION Enter
sleep 2
tmux send-keys -t $PROBE_SESSION "/usage" Enter
sleep 4
tmux capture-pane -t $PROBE_SESSION -p > /tmp/selftalk_probe.out
tmux kill-session -t $PROBE_SESSION
```

Parse `/tmp/selftalk_probe.out` for the **"Current session"** line — it looks like `███... NN% used`. Extract the percentage.

- If session % > 80 → set effective target to the fallback (`-g` → gemini, `-x` → codex, default → gemini). Tell the user "Smart-route: Claude session at NN% — routing to [fallback]."
- Else → keep effective target as `claude`, tell the user "Smart-route: Claude session at NN% — using Claude."

If `/usage` output is unparseable (TUI didn't render in time, network hiccup, etc.) — default to Claude and tell the user the probe was inconclusive. Do not silently flip.

### 3. Spawn (with permissions bypassed)

```bash
SESSION_NAME="selftalk_$(date +%s)"

# Claude
tmux new-session -d -s $SESSION_NAME -x 200 -y 50 -c /tmp/selftalk 'claude --dangerously-skip-permissions'
tmux set-option -t $SESSION_NAME mouse on

# Gemini
tmux new-session -d -s $SESSION_NAME -x 200 -y 50 -c /tmp/selftalk 'gemini -y --skip-trust'
tmux set-option -t $SESSION_NAME mouse on

# Codex
tmux new-session -d -s $SESSION_NAME -x 200 -y 50 -c /tmp/selftalk 'codex --dangerously-bypass-approvals-and-sandbox'
tmux set-option -t $SESSION_NAME mouse on
```

The `--dangerously-skip-permissions` / `-y --skip-trust` / `--dangerously-bypass-approvals-and-sandbox` flags are *required*, not optional — without them, each shell call inside the spawned CLI pops a permission dialog that this driving session can't easily click through. We learned this the hard way; do not strip the flags to be "safe."

Wait ~3 s for the CLI to come up, then `tmux capture-pane -t $SESSION_NAME -p -S - -E -` to see initial state.

### 4. Handle first-run prompts

Even with the bypass flags, the workspace-trust prompt still appears on first use of a new directory.

- **Claude**: "Do you trust this folder?" — option 1 pre-selected. Send `Enter`.
- **Gemini**: trust prompt with three options — option 1 pre-selected. Send `Enter`.
- **Codex**: may show a terms/trust prompt on first run — option 1 pre-selected. Send `Enter`. If it asks about sandbox policy, the bypass flag already overrides it.

### 5. Report the attach command to the user

Verbatim:

> Session up. To watch live (read-only) open a terminal and run: `tmux attach -r -t <SESSION_NAME>` — detach with `Ctrl-b d`.

### 6. Drive the session

**Always** use `tmux load-buffer` / `tmux paste-buffer` for every prompt — even single words or short moves. Never use `tmux send-keys -t $SESSION_NAME "text" Enter` as a combined call: both Claude and Gemini CLI treat the Enter in that form as a newline inside the input box, not a submission. The standalone `tmux send-keys -t $SESSION_NAME Enter` that follows is what actually submits. Pattern:

```bash
cat > /tmp/selftalk_prompt.txt <<'EOF'
<prompt body here>
EOF
tmux load-buffer -t $SESSION_NAME /tmp/selftalk_prompt.txt
tmux paste-buffer -t $SESSION_NAME
sleep 1
tmux send-keys -t $SESSION_NAME Enter
```

Then poll for idle using a background until-loop. The busy indicator differs by CLI:

- **Claude / Gemini**: grep for `esc to cancel`
- **Codex**: grep for `esc to interrupt` (codex shows this while the agent is running)

```bash
# Claude / Gemini
until ! tmux capture-pane -t $SESSION_NAME -p | grep -qE "esc to cancel"; do sleep 3; done
echo "IDLE"; tmux capture-pane -t $SESSION_NAME -p | tail -60

# Codex
until ! tmux capture-pane -t $SESSION_NAME -p | grep -qE "esc to interrupt"; do sleep 3; done
echo "IDLE"; tmux capture-pane -t $SESSION_NAME -p | tail -60
```

Run that in the background with `run_in_background: true`; you'll get a notification when the spawn goes idle. Do not chain manual `sleep` calls — they're blocked beyond a small budget.

### 7. Teardown

```bash
tmux send-keys -t $SESSION_NAME Escape
sleep 1
tmux send-keys -t $SESSION_NAME "/quit" Enter
sleep 2
tmux kill-session -t $SESSION_NAME 2>/dev/null
```

Confirm with `tmux list-sessions 2>&1` ("no server running" if selftalk was the only one).

## Cost warning

A spawned `claude` session draws on the user's Claude quota — every prompt sent counts. Gemini draws on Gemini's free-tier quota. Codex draws on the user's OpenAI API quota (model defaults to o3 unless overridden with `-c model="..."` via codex's own `-c` config flag). The `-e` smart-route probe itself spawns a temporary Claude session and runs `/usage` (cheap — `/usage` is a slash command, not a model call, but it still counts as a session). Mention this on first spawn in a conversation if the user hasn't acknowledged it.

## Do NOT

- Do not spawn more than one `selftalk` session in parallel (recursion is OK because the child kills the parent's session by name — see "Recursive use" above).
- Do not send destructive commands (rm, git push, etc.) to the spawned CLI on the user's behalf — even though it lives in `/tmp/selftalk`, the spawned CLI has Bash access to the whole filesystem and now runs with permissions bypassed.
- Do not strip the `--dangerously-skip-permissions` / `-y --skip-trust` / `--dangerously-bypass-approvals-and-sandbox` flags from the spawn command "to be safe" — without them the spawn is unusable from this driving session.
- Do not edit the spawned CLI's settings.json or any config — the spawn is meant to be ephemeral.
