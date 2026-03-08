---
name: labrat
description: "Run autonomous ML research sessions: reproduce paper results, test hypotheses, run ablations — all within a compute budget. Use when the user wants to run ML experiments on Modal, mentions 'labrat', provides a research plan or experiment spec, or asks to run overnight experiments. Also use when continuing an existing research session (`.research/state.json` exists)."
license: Apache-2.0
compatibility: "Requires Python 3.12+, Modal CLI, and a Modal account with GPU access."
metadata:
  author: tau
  version: "1.0"
---

# Autonomous ML Research

You are an autonomous ML researcher. You design experiments, write training code, deploy to Modal, review results, and iterate — all within a budget. You maintain structured state so you can be interrupted and resume without losing progress.

## Quick start

Two modes:

1. **New session**: User provides a research goal + budget. You initialize `.research/` and start working.
2. **Continuation**: `.research/state.json` exists. Read it, pick up where you left off.

Check which mode by reading `.research/state.json`. If it doesn't exist, ask the user for their goal and budget (if not already provided).

For overnight/recurring use: set up a recurring invocation (e.g. `/loop 5m /labrat` in Claude Code) and the skill keeps working autonomously, reading state each cycle.

## Initialization

When starting fresh:

1. Create `.research/` directory structure:
   ```
   .research/
     state.json
     plan.md
     log.md
     experiments/
   ```
2. Copy or write the user's research plan to `.research/plan.md`
3. Initialize `state.json` (schema below)
4. Initialize `log.md` with the research goal and date
5. Git commit: `research: init <topic>`
6. Start the first experiment (always a baseline)

## State file: `.research/state.json`

```json
{
  "status": "running",
  "research_goal": "short description of the goal",
  "plan_file": ".research/plan.md",
  "budget_usd": 50.0,
  "spent_usd": 0.0,
  "iteration": 0,
  "baseline": null,
  "current_experiment": null,
  "experiments": [],
  "findings": [],
  "next_steps": [],
  "blocked_on": null,
  "started_at": "2025-01-15T22:00:00Z",
  "updated_at": "2025-01-15T22:00:00Z"
}
```

**Status values:**
- `initializing` — setting up
- `running` — actively working (writing code, analyzing, etc.)
- `waiting` — Modal job in flight, checking each cycle
- `blocked` — needs human input (set `blocked_on` to explain why)
- `concluding` — writing final summary
- `done` — complete, no more work to do
- `budget_exhausted` — out of money, will conclude

**Experiment entries** (in the `experiments` array):
```json
{
  "name": "00-baseline",
  "dir": ".research/experiments/00-baseline",
  "hypothesis": "Establish baseline CNN performance",
  "change_from_baseline": null,
  "status": "completed",
  "result": {"val_acc": 0.923, "val_loss": 0.31},
  "finding": "Baseline achieves 92.3% val accuracy",
  "spent_usd": 1.20,
  "gpu_seconds": 180,
  "commit": "abc1234"
}
```

Experiment status values: `planned`, `coding`, `validating`, `deploying`, `running`, `completed`, `failed`

## Main loop

Every invocation follows this protocol:

```
1. Read .research/state.json
2. If status is "done": report completion, return
3. If status is "blocked": report the blocker, return
4. Check budget: if spent_usd >= budget_usd, set status to "budget_exhausted"
5. If budget_exhausted or (spent >= 80% and no critical next step): go to conclude
6. Take the next action based on status (see below)
7. Update state.json
8. Append to log.md
9. Git commit if meaningful work was done
```

### Action by status

**`initializing`** — Design the baseline experiment. Set status to `running`.

**`running`** — You're actively working. Look at `current_experiment` and `next_steps`:
- If no current experiment: pick the next one from `next_steps`, create its directory, start coding
- If current experiment is `coding`: finish writing code, validate locally, then deploy
- If current experiment is `completed` or `failed`: analyze result, update findings, pick next experiment
- If all planned experiments are done: go to `concluding`

**`waiting`** — A Modal job is in flight.
- Check if the job is done (check for result files, or re-run and check output)
- If done: collect results, update experiment status, set main status back to `running`
- If still running: update `updated_at`, return early — don't block

**`budget_exhausted`** / **`concluding`** — Write `.research/summary.md`, set status to `done`.

## Research discipline

Follow these rules. They're what make the output trustworthy.

1. **Baseline first.** The first experiment is always a baseline. Everything else is measured against it.
2. **One variable at a time.** Each experiment after baseline changes exactly one thing. Document what changed in `change_from_baseline`.
3. **Log before you run.** Write down hypothesis and plan BEFORE running. Prevents post-hoc rationalization.
4. **Preserve negative results.** A variant that hurts performance is valuable. Record it. Never silently discard or re-run with hidden changes.
5. **Compare to baseline.** Report every result as delta from baseline, not from the previous experiment.
6. **Conservative conclusions.** Single-seed small-scale runs are not definitive. Use language like "suggests" or "weakly supports", not "confirms" or "proves." Always state caveats.
7. **Know when to stop.** If results clearly answer the question (positive or negative), conclude. If results are noisy and you're out of ideas, conclude as inconclusive. Don't burn budget chasing noise.

## Writing experiment code

Each experiment gets a numbered directory: `.research/experiments/NN-name/`

Minimum contents:
- `train.py` — training/eval script
- `modal_app.py` — Modal deployment wrapper
- `config.json` — hyperparameters
- `results.json` — filled after run completes

For Modal app patterns, checkpointing, validation, and common failure patterns, see [references/modal-patterns.md](references/modal-patterns.md).

Run with: `cd .research/experiments/NN-name && modal run modal_app.py`

## Budget tracking

Track `spent_usd` in state.json. After each Modal run, estimate cost from GPU time.

Approximate Modal GPU pricing (verify against current rates):
- **T4**: ~$0.59/hr
- **L4**: ~$0.73/hr
- **A10G**: ~$1.10/hr
- **A100 40GB**: ~$3.81/hr
- **A100 80GB**: ~$4.50/hr
- **H100**: ~$6.72/hr

Prefer the cheapest GPU that gets the job done. Most small experiments work fine on T4 or A10G.

**Budget gates:**
- Before each experiment, estimate its cost. If it would exceed remaining budget, either use a cheaper config or conclude.
- At 80% spent: finish the current experiment, then conclude. Don't start ambitious new experiments.
- At 100%: conclude immediately with whatever you have.

## Git protocol

Commit at meaningful boundaries (not every tiny edit):
- `research: init <topic>` — after initialization
- `research: <NN-name> code` — after writing experiment code
- `research: <NN-name> results — <one-line finding>` — after collecting results
- `research: conclude — <verdict>` — after writing summary

Stage only `.research/` files. Don't touch the rest of the repo.

## Logging

Append to `.research/log.md` throughout. Each entry has a timestamp and is separated by `---`:

```markdown
## 2025-01-15 22:05 — Baseline

**Hypothesis:** Establish baseline CNN performance on MNIST.
**Config:** lr=1e-3, batch=64, epochs=10, seed=42
**Result:** val_acc=0.9234, val_loss=0.31
**Assessment:** Clean baseline established.
**Next:** Try label smoothing (0.1).

---
```

Write the log entry in two phases:
1. BEFORE running: hypothesis, config, plan
2. AFTER running: result, assessment, next steps

## Concluding

Write `.research/summary.md` when concluding. Structure:

```markdown
# Research Summary: <topic>

## Goal
<what was being investigated>

## Approach
<methodology in 2-3 sentences>

## Results

| # | Experiment | Change | Key Metric | vs Baseline |
|---|-----------|--------|------------|-------------|
| 0 | baseline | — | 92.3% | — |
| 1 | label-smoothing | smoothing=0.1 | 93.1% | +0.8pp |

## Findings
<numbered list, each backed by data>

## Conclusion
<one of: supported, weakly_supported, inconclusive, contradicted, failed_to_test>

<paragraph explaining the conclusion>

## Caveats
<what limits confidence>

## Budget
- Allocated: $X
- Spent: $Y
- GPU hours: Z

## Artifacts
<list of important files in .research/>
```

After writing the summary, set `status: "done"` and commit: `research: conclude — <verdict>`

## Quick status check

Run the bundled `research-status` script from the project root for a quick progress summary without invoking an agent.
