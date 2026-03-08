---
name: treadmill
description: "Run a command or prompt on a recurring interval, like a lab rat on a wheel. Use when the user wants to poll, repeat a task periodically, set up a recurring check, or keep a long-running process supervised. Pairs with /labrat for overnight research sessions."
license: Apache-2.0
compatibility: "Requires bash. Works on macOS and Linux."
metadata:
  author: tau
  version: "1.0"
allowed-tools: Bash
---

# Treadmill

Run a command on a recurring interval using a background shell loop. Portable across all agent harnesses.

## How to use

The bundled `treadmill` script handles start/stop/status. Set it up from the skill directory:

```bash
SKILL_DIR="$(cd "$(dirname "$0")" && pwd)"
export PATH="${SKILL_DIR}/scripts:$PATH"
```

Or reference it directly: `bash ${CLAUDE_SKILL_DIR}/scripts/treadmill`

### Start a treadmill

```bash
treadmill start <interval> <command...>
```

Interval formats: `30s`, `5m`, `1h`, or plain seconds (e.g. `300`).

Examples:
```bash
# Check research status every 5 minutes
treadmill start 5m python .research/experiments/00-baseline/modal_app.py

# Poll a deployment
treadmill start 2m curl -s https://api.example.com/health

# Run a script every hour
treadmill start 1h python check_metrics.py
```

### Stop

```bash
treadmill stop
```

### Check status

```bash
treadmill status
```

### View logs

```bash
treadmill log      # last 30 lines
treadmill log 100  # last 100 lines
```

## State

Treadmill keeps its state in `.treadmill/` in the current directory:
- `pid` — PID of the background loop
- `config` — interval, command, start time
- `log` — stdout/stderr from each run

## Pairing with labrat

For overnight ML research, start a treadmill that re-runs the labrat check:

```bash
# Have the agent check in every 5 minutes
treadmill start 5m python -c "
import json, sys
state = json.load(open('.research/state.json'))
print(f'Status: {state[\"status\"]}')
if state['status'] == 'waiting':
    print('Job still running...')
elif state['status'] == 'done':
    print('Research complete.')
    sys.exit(0)
"
```

The agent can read treadmill logs to see what happened between invocations.

## When to use this vs built-in loop

Some agent tools (like Claude Code) have a built-in `/loop` command. Use that when available. Use `/treadmill` when:
- Your agent harness doesn't have a built-in loop
- You want a detached background process that survives agent restarts
- You need to run shell commands on a timer independent of the agent
