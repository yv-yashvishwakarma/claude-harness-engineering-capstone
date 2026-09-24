# Harness Engineering with Claude and Claude Code

A four-system capstone exploring practical harness engineering patterns for reliable AI-assisted software development.

The project covers agentic control loops, long-context management, Claude Code repository configuration, and multi-shift orchestration with deterministic state and recovery controls.

## Systems

### 1. Agentic Loop — Stop-Reason-Driven Control

Built a claims-intake agent around the model's `stop_reason` rather than a fixed iteration limit.

Key patterns:
- `tool_use` drives continued tool execution.
- `end_turn` terminates the loop.
- Structured tool errors distinguish retryable and non-retryable failures.
- Trace artifacts capture the agent trajectory.

**Tests:** 29/29 passed.

### 2. Long-Conversation Context Strategy

Built a context assembly strategy for a retail support copilot.

Observed result:
- Baseline context: 38,708 tokens
- Assembled context: 16,858 tokens
- Reduction: 56.45%

**Tests:** 28 passed, 2 skipped.

Main evaluation: **6/6 passed.**

### 3. Claude Code Multi-Surface Configuration

Configured a multi-surface repository using Claude Code project instructions, path-scoped rules, shared standards, and a forked read-only deployment-check skill.

Key patterns:
- Project-level `CLAUDE.md`
- Path-scoped `.claude/rules/`
- Shared `.claude/standards/`
- Forked `deploy-check` skill
- Read-only tool allowlist
- Repository validation

**Tests:** 35/35 passed.

**Validator:** OK.

### 4. Multi-Shift Quality Monitoring Orchestration

Built a multi-shift monitoring workflow using layered orchestration, warm-tier historical storage, bounded hot state, and crash-recovery logic.

Key patterns:
- SQL pre-filtering before model invocation
- Bounded hot-state storage
- Atomic state updates
- 30-minute recovery threshold
- Resumed invocations with prior partial findings
- Deterministic state and recovery controls around model execution

**Tests:** 33/33 passed.

**Warm-tier defects:** 40

**Hot-state size:** 643 bytes
**Configured maximum:** 5,120 bytes

## Engineering Themes

| System | Main problem | Harness pattern |
|---|---|---|
| 1 | Agent control | Stop-reason-driven loop |
| 2 | Context growth | Context assembly and token budgeting |
| 3 | Repository guidance | Scoped rules and forked skills |
| 4 | Multi-step orchestration | State, recovery, and bounded memory |

## Evidence

The repository includes test results, live-run outputs, traces, evaluation artifacts, context-budget measurements, configuration files, and orchestration evidence.

The detailed project reflection is available in [reflection-brief.md](reflection-brief.md).

## Repository Structure

```text
.
├── system1-agentic-loop/
├── system2-context-strategy/
├── system3-claude-code-config/
├── system4-orchestration/
└── reflection-brief.md
```

## Technologies and Concepts

- Python
- Anthropic Claude API
- Claude Code
- Agentic tool-use loops
- Long-context engineering
- Token budgeting
- Structured tool errors
- Repository-level AI instructions
- Path-scoped development rules
- Forked sub-agent workflows
- SQL pre-filtering
- Persistent state
- Crash recovery
- Deterministic orchestration

## Notes

This repository contains the capstone implementation and supporting evidence. API credentials are supplied through environment variables and are not committed to the repository.
