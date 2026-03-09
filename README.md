# labrat

Autonomous ML research agent skill. Interviews the user, scopes an ML project into a tractable plan, then designs experiments, deploys to Modal GPUs, tracks results, and iterates within a compute budget.

Also includes **treadmill** — a portable recurring-command skill for agent harnesses that don't have a built-in loop.

Built on the [Agent Skills](https://agentskills.io) open standard. Works with Claude Code, Codex, OpenCode, Cursor, Gemini CLI, and any other compatible tool.

## Install

### Claude Code

Copy the skill directories to your personal skills folder:

```bash
cp -r labrat/ ~/.claude/skills/labrat
cp -r treadmill/ ~/.claude/skills/treadmill
```

Or install as a plugin from GitHub:

```bash
# In Claude Code
/install-plugin gh:tnguyen21/labrat
```

Then invoke with `/labrat` or just describe a research goal.

### Codex

Copy the skill into your project:

```bash
cp -r labrat/ .codex/skills/labrat
```

Or place the `AGENTS.md` in your project root for basic instructions without the full skill system.

### OpenCode

Copy the skill to any supported location:

```bash
# Project-local
cp -r labrat/ .opencode/skills/labrat

# Or global
cp -r labrat/ ~/.config/opencode/skills/labrat
```

OpenCode also reads from `.claude/skills/` and `.agents/skills/`.

### Other Agent Skills-compatible tools

Copy the `labrat/` directory to wherever your tool discovers skills. The format follows the [Agent Skills spec](https://agentskills.io/specification).

## Prerequisites

- Python 3.12+
- [Modal](https://modal.com) CLI (`uv tool install modal && modal setup`)
- Modal account with GPU access

## Usage

Start a new research session:

> Run a labrat session: test whether dropout rate affects convergence on CIFAR-10. Budget: $10.

For a new session, the agent first asks a few clarifying questions and writes a scoped project brief in `.research/scope.md` before writing code.

Continue an existing session (if `.research/state.json` exists):

> Continue the research session.

Check status without an agent:

```bash
python labrat/scripts/research-status
```

Advance state without an agent:

```bash
python labrat/scripts/research-advance
```

For Codex-style unattended progress, run the supervisor instead:

```bash
python labrat/scripts/research-supervise
```

## How it works

1. **Interview** — Clarifies goals, metrics, constraints, and writes a scoped project brief
2. **Initialize** — Creates `.research/` with `state.json`, `scope.md`, `plan.md`, `log.md`
3. **Baseline** — Always runs a baseline experiment first
4. **Iterate** — Each experiment changes one variable from baseline
5. **Track** — Logs results, tracks spend against budget, and reconciles finished artifacts back into state
6. **Conclude** — Writes `summary.md` with results table and findings

`research-advance` only reconciles state. `research-supervise` adds the next handoff for Codex by invoking `codex exec` when the session is actionable again, which is the closest equivalent to Claude Code's recurring `/loop`.

Experiments run on Modal GPUs (defaults to T4, the cheapest option). Each experiment produces:
- `config.json` — hyperparameters
- `train.py` — training script
- `modal_app.py` — Modal deployment wrapper
- `results.json` — collected after run

## Structure

```
labrat/                           # ML research skill
  SKILL.md                        # Agent instructions
  scripts/research-advance        # State reconciliation worker
  scripts/research-supervise      # Reconcile state, then wake Codex if needed
  scripts/research-status         # CLI status checker
  references/modal-patterns.md    # Modal deployment patterns
treadmill/                        # Recurring command skill
  SKILL.md                        # Agent instructions
  scripts/treadmill               # Background loop manager
AGENTS.md                         # Codex/fallback instructions
README.md
LICENSE
```

## License

Apache-2.0
