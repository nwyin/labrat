# labrat

Autonomous ML research agent skill. Designs experiments, deploys to Modal GPUs, tracks results, and iterates — all within a compute budget.

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

Continue an existing session (if `.research/state.json` exists):

> Continue the research session.

Check status without an agent:

```bash
python labrat/scripts/research-status
```

## How it works

1. **Initialize** — Creates `.research/` with `state.json`, `plan.md`, `log.md`
2. **Baseline** — Always runs a baseline experiment first
3. **Iterate** — Each experiment changes one variable from baseline
4. **Track** — Logs results, tracks spend against budget
5. **Conclude** — Writes `summary.md` with results table and findings

Experiments run on Modal GPUs (defaults to T4, the cheapest option). Each experiment produces:
- `config.json` — hyperparameters
- `train.py` — training script
- `modal_app.py` — Modal deployment wrapper
- `results.json` — collected after run

## Structure

```
labrat/                           # ML research skill
  SKILL.md                        # Agent instructions
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
