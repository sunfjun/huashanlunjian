# Agora

> Multi-AI role debate plugin for Claude Code — generate high-quality proposals through structured multi-perspective discussion.

## Overview

Agora assembles multiple AI roles to review your proposal from different perspectives, refining it through multiple rounds of debate until consensus is reached.

- 🎭 **Momus** — Devil's advocate: challenges assumptions, finds security risks and extreme scenarios
- 🔮 **Cassandra** — Detail insight: catches overlooked edge cases, boundary conditions and subtle flaws
- 🦉 **Athena** — Strategic architecture: reviews global design, scalability and long-term impact
- ⚒️ **Hephaestus** — Execution feasibility: evaluates implementability, resources and schedule risks

> *"Agora assembles four irreplaceable perspectives — the critic's eye, the prophet's voice, the strategist's judgment, the craftsman's hand."*

Discussion runs automatically until all roles reach consensus (up to N rounds), with the full process written to file for review.

## Installation

### From marketplace

In Claude Code, run the following commands:

```bash
# 1. Add marketplace
/plugin marketplace add ClaudeWorksHub/claude-plugins

# 2. Install plugin
/plugin install agora@claudeworkshub

# 3. Activate
/reload-plugins
```

To update to the latest version:

```bash
/plugin marketplace update claudeworkshub
/plugin install agora@claudeworkshub
/reload-plugins
```

### Local development

Copy the `agora/` directory to `~/.claude/plugins/`:

```bash
cp -r agora ~/.claude/plugins/
```

Or configure the plugin path in your project's `.claude/settings.local.json`.

## Usage

### Basic

```
/agora:proposal Design a user authentication system
```

### Select roles

```
/agora:proposal --roles athena,momus Design a caching system
```

### Exclude roles

```
/agora:proposal --exclude hephaestus Design a notification system
```

### Set max rounds

```
/agora:proposal --max-rounds 5 Design a notification system
```

### Combined

```
/agora:proposal --roles cassandra,athena --max-rounds 8 Design an API gateway
```

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--roles` | All 4 roles | Comma-separated role names (or legacy aliases) |
| `--exclude` | None | Exclude specific roles |
| `--max-rounds` | 10 | Max discussion rounds (1-20) |

### Role names

| Name | Legacy alias | Perspective |
|------|-------------|-------------|
| `momus` | `sunwukong` | Devil's advocate |
| `cassandra` | `lindaiyu` | Detail & edge cases |
| `athena` | `zhugeliang` | Strategy & architecture |
| `hephaestus` | `wangxifeng` | Execution & feasibility |

## Output

Each run generates two files in your project's `proposals/` directory:

| File | Content |
|------|---------|
| `proposal-<timestamp>.md` | Full discussion process |
| `proposal-<timestamp>-final.md` | Final proposal |

## How it works

```
Phase 0: Parse arguments, confirm with user
              ↓
Phase 1: Generate initial proposal
              ↓
Phase 2: Multi-round discussion (max N rounds)
         ┌─→ Each role reviews proposal (parallel recommended, serial fallback)
         │        ↓
         │   Main Claude responds to objections
         │        ↓
         │   Check convergence
         └── Continue if not converged
              ↓
Phase 3: Final proposal with self-review
```

### Convergence rules

- Rounds 1-3: Deep review encouraged — each role reviews from at least 3 dimensions (can be no-objection with dimension list)
- Round 4+: Converge when all active roles have no objections for 2 consecutive rounds (structural rule, main Claude cannot override)
- Grep cross-check: Before convergence, verify status from file matches parsed results
- Hard limit: Stop at `--max-rounds` with best current proposal + unresolved issues listed

## Git handling

The plugin does not modify your `.gitignore`. Suggestions:

- Discussion files can be large — consider adding `proposals/proposal-*.md` to `.gitignore`
- Final proposals (`*-final.md`) are worth committing as design decision documents
- Or keep everything for full audit trail

## Extensibility

Ships with 4 built-in roles. The format is open for future custom roles:

- Create a role file following the same format in your project's `.claude/agents/`
- Use `--roles` to include it in discussions
- The role must follow the same status/summary protocol

## Discussion language

Discussions follow the language of your task description — English task gives English discussion, etc. Role names are Greek mythology characters (Momus/Cassandra/Athena/Hephaestus); legacy aliases (sunwukong/lindaiyu/zhugeliang/wangxifeng) remain supported.

## License

MIT
