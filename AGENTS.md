# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

Skills library for V-Sekai multiplayer-fabric agents. Each skill is a
self-contained, reusable behaviour that agents (zone_console, zone backend,
Godot zone processes) can compose to accomplish tasks.

## Repo layout

| Path | Purpose |
|------|---------|
| `skills/` | Individual skill definitions |
| `AGENTS.md` | This file |

## Rules

- No `type(scope):` commit prefixes. Sentence case only.
- Each skill in `skills/` is one file with a clear docstring describing
  inputs, outputs, and side effects.
- Skills must not import from each other to avoid dependency cycles.
