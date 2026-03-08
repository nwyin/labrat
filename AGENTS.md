# Labrat — Autonomous ML Research Agent

This repository contains a skill for AI coding agents. See `labrat/SKILL.md` for the full instructions.

## What this does

Labrat turns your AI agent into an autonomous ML researcher. Given a research goal and compute budget, it:

1. Starts with a short interview to clarify goals, metrics, and constraints
2. Creates a structured `.research/` directory with state tracking
3. Designs and runs a baseline experiment on Modal GPUs
4. Iterates with controlled ablations (one variable at a time)
5. Tracks spending against your budget
6. Writes a final summary with results table and conclusions

## Prerequisites

- Python 3.12+
- [Modal](https://modal.com) CLI installed and authenticated (`uv tool install modal && modal setup`)
- A Modal account with GPU access

## Quick start

Tell your agent:

> Run a labrat session: test whether label smoothing improves MNIST accuracy. Budget: $5.

The agent will first ask a few clarifying questions and write a scoped brief in `.research/scope.md`, then initialize `.research/`, write experiment code, deploy to Modal, collect results, and iterate.

## Key files

- `labrat/SKILL.md` — Full agent instructions (Agent Skills format)
- `labrat/scripts/research-status` — CLI status checker (`python labrat/scripts/research-status`)
- `labrat/references/modal-patterns.md` — Modal deployment templates and patterns
