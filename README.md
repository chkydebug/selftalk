# selftalk

`selftalk` is a cross-platform orchestration skill for **Claude Code**, **Gemini CLI**, and **Codex CLI**. It allows one agent to drive another through a detached `tmux` session, turning agent-to-agent delegation into a first-class workflow.

The project was born out of a practical necessity: **overcoming the low usage limits of Claude Code.** 

While Claude is often the preferred agent for complex strategy and surgical edits, its session limits can be restrictive. `selftalk` provides a "Model Offloading" architecture, allowing you to stay in your primary Claude session while transparently offloading minor tasks, boilerplate generation, or exploratory research to the higher-quota environments of Gemini or Codex.

## Why This Exists: The Offloading Strategy

Modern CLI agents have vastly different "resource envelopes." `selftalk` treats these agents as schedulable runtimes that can be swapped based on the task at hand:

- **Primary Driver (Claude):** Best for high-level reasoning, project architecture, and final verification.
- **Worker Offloading (Gemini/Codex):** Ideal for "heavy lifting" tasks like writing unit tests, refactoring multiple files, or scaffolding new modules where you don't want to burn through your Claude quota.

By using the **Smart-Routing** feature (`-e`), `selftalk` automatically monitors your Claude session usage. If you're nearing your limit, it transparently shifts the work to a fallback worker, ensuring your development flow never hits a hard stop.

## Agent Hypervisor Concept

`selftalk` acts as an **Agent Hypervisor**. Instead of treating models as isolated chat windows, it treats them as background processes that can be launched, monitored, and managed.

- **Unified Control:** One agent stays in charge; others run in the background.
- **Spectator Mode:** Watch the worker live (read-only) with `tmux attach -r -t selftalk`.
- **Quota-Aware:** Routing decisions are based on real-time usage data.
- **Isolated Execution:** Workers run in `/tmp/selftalk` to prevent polluting your main project state until you're ready to merge.

## Smart Routing

The standout feature is **smart-route**.

When invoked with `-e`, `selftalk` probes Claude’s current session usage via `/usage` before deciding what to launch. If Claude is above the threshold, `selftalk` routes to a fallback target instead:

- `-c -e -g`: prefer Claude, fall back to Gemini
- `-c -e -x`: prefer Claude, fall back to Codex
- `-e`: prefer Claude, fall back to Gemini by default

That makes `selftalk` practical as a quota-management layer, not just a launcher. It lets users preserve Claude capacity for the work that actually needs it, while shifting overflow execution to Gemini or Codex automatically.

If the probe is inconclusive, the behavior is explicit: default back to Claude and report that the probe could not be parsed. No silent routing surprises.

## Current Behavior

`selftalk` currently supports:

- `claude`
- `gemini`
- `codex`

It handles the repetitive operational details that make agent-driving awkward by hand:

- pre-flight binary checks
- stale `tmux` session replacement
- first-run trust prompt handling
- required permission-bypass flags for unattended driving
- prompt submission through `tmux load-buffer` and `paste-buffer`
- idle detection based on each CLI’s running indicator
- clean session teardown

The user-facing result is a single command: spawn the worker, watch it live, and keep the orchestration logic in one place.

## Installation

This repo is a Codex skill. Installation is just placing it under your Codex skills directory.

### Prerequisites

Before installing, make sure the machine has:

- `tmux`
- at least one supported CLI: `claude`, `gemini`, or `codex`

### Option 1: Manual install

```bash
mkdir -p ~/.codex/skills/selftalk
cp SKILL.md ~/.codex/skills/selftalk/SKILL.md
```

If you are installing from GitHub:

```bash
git clone https://github.com/OWNER/REPO.git
mkdir -p ~/.codex/skills/selftalk
cp REPO/SKILL.md ~/.codex/skills/selftalk/SKILL.md
```

Restart Codex after installation so the new skill is picked up.

### Option 2: Repo-local install

If you already cloned the repository locally:

```bash
mkdir -p ~/.codex/skills/selftalk
cp /path/to/selftalk-skill/SKILL.md ~/.codex/skills/selftalk/SKILL.md
```

Restart Codex after installation.

Note: Codex’s GitHub skill installer expects a skill to live in a subdirectory inside a repository. Since this project is a single-skill repo with `SKILL.md` at the root, manual copy is the correct installation path today.

## Usage

Once installed, invoke the skill through `/selftalk` with the target flags you want:

```text
/selftalk -c
/selftalk -g "summarize this repository"
/selftalk -x "inspect this bug"
/selftalk -c -e -g "continue the implementation"
```

After spawn, the user can watch the delegated agent live in a separate terminal:

```bash
tmux attach -r -t selftalk
```

Detach with `Ctrl-b d`.

## Safety Model

`selftalk` is intentionally operational, not restrictive.

- It uses bypass flags required for unattended terminal driving.
- It assumes the user understands the quota and execution implications.
- It does not try to block recursive use.
- It avoids mutating agent configuration and treats the worker session as ephemeral.

That makes the skill powerful, but it also means contributors should preserve clarity around cost, permissions, and routing behavior.

## Contributing

Contributions are welcome, especially from people interested in agent orchestration, quota-aware routing, and terminal automation.

High-value areas for contribution:

- support for additional agent CLIs
- more robust smart-route probing and heuristics
- better observability around spawn, idle, and teardown states
- install helpers and packaging improvements
- tests or fixtures for parsing and session-state detection
- documentation and examples for real orchestration workflows

Good contributions here are not just code changes. They should improve the discipline of the hypervisor model: predictable spawning, explicit routing, visible control flow, and minimal operator surprise.

## Design Standard

If you contribute, optimize for:

- explicit behavior over hidden magic
- operational reliability over cleverness
- transparent quota decisions
- reproducible terminal behavior
- user visibility into what the spawned agent is doing

`selftalk` is trying to make agent orchestration feel like systems engineering, not prompt theater.
