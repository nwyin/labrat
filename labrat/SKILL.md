---
name: labrat
description: "Run autonomous ML research sessions: reproduce paper results, test hypotheses, run ablations — all within a compute budget. Use when the user wants to run ML experiments on Modal, mentions 'labrat', provides a research plan or experiment spec, or asks to run overnight experiments. Also use when continuing an existing research session (`.research/state.json` exists)."
license: Apache-2.0
compatibility: "Requires Python 3.12+, Modal CLI, and a Modal account with GPU access."
metadata:
  author: tau
  version: "1.3"
---

# Autonomous ML Research

You are an autonomous ML researcher. You design experiments, write training code, deploy to Modal, review results, and iterate — all within a budget. You maintain structured state so you can be interrupted and resume without losing progress.

## Quick start

Two modes:

1. **New session**: User provides a research goal + budget. You run a short interview, help narrow the scope, then initialize `.research/` and start working.
2. **Continuation**: `.research/state.json` exists. Read it, pick up where you left off.

Check which mode by reading `.research/state.json`. If it doesn't exist, ask the user for their goal and budget (if not already provided), then begin the interview/scoping flow below.

For overnight/recurring use: set up a recurring invocation (e.g. `/loop 5m /labrat` in Claude Code) and the skill keeps working autonomously, reading state each cycle. When using `treadmill`, run an idempotent state-advance worker or supervisor, not a passive status script.

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
     assets/
     experiments/
   ```
2. Ensure the project is in a git repository. If not, initialize one before continuing so all checkpoints are auditable.
3. Run the interview/scoping step and write the result to `.research/scope.md`
4. Copy or write the scoped research plan to `.research/plan.md`
5. Initialize `state.json` only after the scope is actionable (schema below)
6. Initialize `log.md` with the research goal and date
7. Git commit: `research: init <topic>`
8. Start the first experiment (always a baseline)

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

Optional fields worth tracking when you launch remote jobs:

```json
{
  "artifact_store": {
    "kind": "modal_volume",
    "volume_name": "research-vol",
    "base_path": "/labrat/<session-slug>"
  },
  "current_experiment": {
    "name": "00-baseline",
    "dir": ".research/experiments/00-baseline",
    "status": "running",
    "modal_app_id": "ap-1234567890",
    "remote_dir": "/labrat/<session-slug>/00-baseline",
    "remote_results_path": "/labrat/<session-slug>/00-baseline/results.json"
  }
}
```

**Status values:**
- `initializing` — setting up
- `running` — actively working (writing code, analyzing, etc.)
- `waiting` — Modal job in flight, checking each cycle
- `blocked` — needs human input (set `blocked_on` to explain why)
- `concluding` — writing final report
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
12. Write or refresh any user-facing artifacts affected by the work
13. Git commit if meaningful work was done
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

When using `treadmill`, the recurring command should be a state-transition worker such as `research-advance`, not a pure observer. The worker must:
- reconcile `.research/state.json` with remote volume artifacts first, then local `results.json`, and finally any tracked `modal_app_id`
- move `waiting -> running` when results are harvested
- mark `blocked` if the remote app stopped and no result artifact exists
- append to `.research/log.md` when state transitions happen

For Codex specifically, prefer the bundled supervisor:

```bash
python labrat/scripts/research-supervise
```

`research-supervise` runs `research-advance` first, then invokes `codex exec` when the session is actionable again. This gives Codex a `/loop`-like handoff: remote completion gets reconciled into state, then a fresh Codex worker picks up the next iteration.
**`budget_exhausted`** / **`concluding`** — Write `.research/report.html`, set status to `done`.

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
- `results.json` — local cache of the final result after harvest completes

For Modal app patterns, checkpointing, validation, and common failure patterns, see [references/modal-patterns.md](references/modal-patterns.md).

Run with: `cd .research/experiments/NN-name && modal run modal_app.py`

By default, write remote artifacts to a shared Modal Volume in a stable per-session layout:

```text
/labrat/<session-slug>/
  00-baseline/
    results.json
    metadata.json
    checkpoints/
  01-variant/
    results.json
    metadata.json
    checkpoints/
```

Track the volume in `.research/state.json` under `artifact_store`, and track each experiment's `remote_dir` or `remote_results_path`. The recurring worker should treat the remote `results.json` as the source of truth, then write a local `.research/experiments/NN-name/results.json` cache after harvesting.

If you are running unattended, prefer this pattern:
- run detached on Modal
- record `modal_app_id`, `remote_dir`, and `remote_results_path` in state
- have the remote function write `results.json` and `metadata.json` into the Modal Volume before exit
- have the recurring worker poll the volume artifact first and only fall back to app-state inspection for recovery

Only use a durable local waiter process as a fallback when you cannot rely on a volume-backed artifact path.

Do not treat a local PID or a passive status script as the source of truth for completion.

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

Git is mandatory for auditability. Work inside a git repository and keep checkpoint commits throughout the session.

If the directory is not already a git repo:
- run `git init` before starting research artifacts
- make the initialization commit once `.research/` scaffolding, scope, and plan exist

Commit at meaningful boundaries (not every tiny edit):
- `research: init <topic>` — after initialization
- `research: scope <topic>` — after major scope revisions, if the project definition changed materially
- `research: <NN-name> code` — after writing experiment code
- `research: <NN-name> results — <one-line finding>` — after collecting results
- `research: conclude — <verdict>` — after writing the final HTML report

Do not go multiple major phases without a commit. At minimum there should be auditable snapshots for:
- scoped project definition
- each experiment's planned code
- each experiment's collected result
- the final report

Stage only `.research/` files unless the user explicitly asked for broader repo changes.

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

Write `.research/report.html` when concluding. This is the primary deliverable for the user, and it should be easy to skim quickly. Use semantic HTML with clear section headings and compact tables.

The report must include:
- title and one-sentence verdict near the top
- a short executive summary
- the goal and scoped hypothesis
- the approach in 2-3 sentences
- a results table comparing every experiment to baseline
- graphs and/or diagrams when they materially improve understanding
- findings backed by data
- caveats and confidence level
- budget and compute usage
- links or references to important local artifacts in `.research/`
- a "How to Read the Code" section
- a "How to Sanity-Check the Results" section

Create report visuals as local files under `.research/assets/` and embed them into `report.html` with relative paths. Prefer simple static PNG or SVG artifacts that are easy to inspect in git diffs and in a browser.

Useful visuals include:
- metric-over-time plots for training or validation curves
- bar charts for baseline vs variant comparisons
- small workflow diagrams showing the execution path or experiment pipeline
- confusion matrices or similar diagnostic plots when they clarify behavior

Do not add decorative charts. Every diagram or graph should answer a concrete review question. Label axes, units, seeds, and experiment names clearly enough that a reviewer can sanity-check the figure quickly.

The "How to Read the Code" section should help a reviewer skim quickly. Include:
- which files matter most and why
- the execution path from config to training to evaluation to result serialization
- where the baseline/variant difference is implemented
- where to look for likely mistakes first

The "How to Sanity-Check the Results" section should help a reviewer audit quickly. Include:
- the exact metric to inspect first
- expected behavior if the hypothesis is true vs false
- obvious failure modes or confounders
- whether the result is strong, weak, noisy, or inconclusive
- the minimum checks needed to decide whether the run is credible

Suggested structure:

```html
<html>
  <head>
    <title>Research Report: <topic></title>
  </head>
  <body>
    <h1>Research Report: <topic></h1>

    <section>
      <h2>Executive Summary</h2>
      <p>...</p>
    </section>

    <section>
      <h2>Goal</h2>
      <p>...</p>
    </section>

    <section>
      <h2>Approach</h2>
      <p>...</p>
    </section>

    <section>
      <h2>Results</h2>
      <table>
        <thead>
          <tr>
            <th>#</th><th>Experiment</th><th>Change</th><th>Key Metric</th><th>vs Baseline</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>0</td><td>baseline</td><td>—</td><td>92.3%</td><td>—</td>
          </tr>
          <tr>
            <td>1</td><td>label-smoothing</td><td>smoothing=0.1</td><td>93.1%</td><td>+0.8pp</td>
          </tr>
        </tbody>
      </table>
      <figure>
        <img src="assets/results-overview.png" alt="Overview chart comparing key experiment metrics">
        <figcaption>...</figcaption>
      </figure>
    </section>

    <section>
      <h2>Findings</h2>
      <ol>
        <li>...</li>
      </ol>
    </section>

    <section>
      <h2>How to Read the Code</h2>
      <ul>
        <li>...</li>
      </ul>
    </section>

    <section>
      <h2>How to Sanity-Check the Results</h2>
      <ul>
        <li>...</li>
      </ul>
    </section>

    <section>
      <h2>Conclusion</h2>
      <p>supported | weakly_supported | inconclusive | contradicted | failed_to_test</p>
      <p>...</p>
    </section>

    <section>
      <h2>Caveats</h2>
      <p>...</p>
    </section>

    <section>
      <h2>Budget</h2>
      <ul>
        <li>Allocated: $X</li>
        <li>Spent: $Y</li>
        <li>GPU hours: Z</li>
      </ul>
    </section>

    <section>
      <h2>Artifacts</h2>
      <ul>
        <li>...</li>
      </ul>
    </section>
  </body>
</html>
```

After writing the HTML report, set `status: "done"` and commit: `research: conclude — <verdict>`

## Quick status check

Run the bundled `research-status` script from the project root for a quick progress summary without invoking an agent.
