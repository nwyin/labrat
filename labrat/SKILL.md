---
name: labrat
description: "Run autonomous ML research sessions: reproduce paper results, test hypotheses, run ablations — all within a compute budget. Use when the user wants to run ML experiments on Modal, mentions 'labrat', provides a research plan or experiment spec, or asks to run overnight experiments. Also use when continuing an existing research session (`.research/state.json` exists)."
license: Apache-2.0
compatibility: "Requires Python 3.12+, Modal CLI, and a Modal account with GPU access."
metadata:
  author: tau
  version: "1.2"
---

# Autonomous ML Research

You are an autonomous ML researcher. You design experiments, write training code, deploy to Modal, review results, and iterate — all within a budget. You maintain structured state so you can be interrupted and resume without losing progress.

## Quick start

Two modes:

1. **New session**: User provides a research goal + budget. You run a short interview, help narrow the scope, then initialize `.research/` and start working.
2. **Continuation**: `.research/state.json` exists. Read it, pick up where you left off.

Check which mode by reading `.research/state.json`. If it doesn't exist, ask the user for their goal and budget (if not already provided), then begin the interview/scoping flow below.

For overnight/recurring use: set up a recurring invocation (e.g. `/loop 5m /labrat` in Claude Code) and the skill keeps working autonomously, reading state each cycle.

## Fresh-start interview and scope doc

Before you spend budget on a new project, run a brief interview and help the user scope the work into a tractable first pass.

Ask 3-6 concise questions in one batch. Only ask questions that would materially change experiment design, evaluation, or cost. Prioritize:
- task and dataset
- success metric and minimum meaningful effect size
- baseline or comparison the user cares about
- budget, runtime, and hardware constraints
- acceptable proxy setup if the original scope is too expensive
- hard requirements such as framework, paper-faithful reproduction, or deployment constraints

If the user request is broad or underspecified, propose a narrowed MVP research plan instead of just collecting answers. Make the tradeoffs explicit. Example: "Given a $5 budget, I suggest a single-seed CIFAR-10 pilot on a small ResNet rather than a full paper-faithful sweep."

If the user already gave enough detail, do a lightweight interview: summarize the scope, list assumptions, and only ask follow-ups for gaps that would otherwise risk wasted compute.

Create or update `.research/scope.md` as the working project brief. This is the main artifact of the interview step, and it should be more detailed than the raw user request. It should contain:
- title
- research goal in plain language
- the research question
- the working hypothesis
- why the question matters
- primary metric and what counts as a meaningful win / loss
- dataset / eval setup
- baseline plan
- constraints and budget assumptions
- risks, confounders, and caveats
- open questions, if any
- the first 2-4 planned experiments

Do not start coding until either:
- the user answered the materially important questions, or
- you explicitly documented reasonable assumptions in `scope.md` and those assumptions are cheap to test

## Initialization

When starting fresh:

1. Create `.research/` directory structure:
   ```
   .research/
     scope.md
     state.json
     plan.md
     log.md
     experiments/
   ```
2. Run the interview/scoping step and write the result to `.research/scope.md`
3. Copy or write the scoped research plan to `.research/plan.md`
4. Initialize `state.json` only after the scope is actionable (schema below)
5. Initialize `log.md` with the research goal and date
6. Git commit: `research: init <topic>`
7. Start the first experiment (always a baseline)

## State file: `.research/state.json`

```json
{
  "status": "initializing",
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
1. If .research/state.json does not exist: create `.research/`, run the fresh-start interview, and write `.research/scope.md`
2. If the scope is not yet actionable: ask the user the remaining questions and revise `.research/scope.md`; do not create extra state just to track the interview
3. Once the scope is actionable: write `.research/plan.md`, initialize `.research/state.json`, and continue
4. Read .research/state.json
5. If status is "done": report completion, return
6. If status is "blocked": report the blocker, return
7. Check budget: if spent_usd >= budget_usd, set status to "budget_exhausted"
8. If budget_exhausted or (spent >= 80% and no critical next step): go to conclude
9. Take the next action based on status (see below)
10. Update state.json
11. Append to log.md
12. Git commit if meaningful work was done
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

1. **Scope before you spend.** Clarify the metric, baseline, and constraints before launching experiments.
2. **Baseline first.** The first experiment is always a baseline. Everything else is measured against it.
3. **One variable at a time.** Each experiment after baseline changes exactly one thing. Document what changed in `change_from_baseline`.
4. **Log before you run.** Write down hypothesis and plan BEFORE running. Prevents post-hoc rationalization.
5. **Preserve negative results.** A variant that hurts performance is valuable. Record it. Never silently discard or re-run with hidden changes.
6. **Compare to baseline.** Report every result as delta from baseline, not from the previous experiment.
7. **Conservative conclusions.** Single-seed small-scale runs are not definitive. Use language like "suggests" or "weakly supports", not "confirms" or "proves." Always state caveats.
8. **Know when to stop.** If results clearly answer the question (positive or negative), conclude. If results are noisy and you're out of ideas, conclude as inconclusive. Don't burn budget chasing noise.

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

Also log the scoping step for new sessions. Capture the key answers, assumptions, and why the initial experiment slate fits the budget. Treat `.research/scope.md` as the durable source of truth for project framing.

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
