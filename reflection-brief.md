# Reflection Brief — Harness Engineering Capstone

**Name:** Yash Vishwakarma
**Date:** 2026-09-24

## Environment

- **Model(s):** System 1 used `claude-haiku-4-5-20251001`. System 2 used its configured Anthropic model. Systems 3 and 4 did not require a comparable live model run for verification.
- **OS / Python:** Linux / Python 3.13.0
- **Approx. API spend:** System 1 reported an estimated total cost of $0.1214. System 2's recorded artifact contains model-authoritative token counts rather than a preserved dollar-cost figure.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The `claim_02_stolen_bike` trace shows `tool_use → tool_use → tool_use → tool_use → end_turn`. In `claims_intake/loop.py`, the `run()` function continues when `response.stop_reason == "tool_use"` and executes the requested tools; when it receives `end_turn`, it returns `FinalState`. Any other value raises `UnexpectedStopReason`. This makes `stop_reason` the control signal rather than model text. Artifact: `runs/20260924_170558/traces/claim_02_stolen_bike.jsonl` and `claims_intake/loop.py`.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_antipatterns.py` checks for an integer-literal iteration cap such as `for _ in range(5)` or `while turn < 5`. A fixed cap could terminate a claim before the model reaches `end_turn`, even when additional tool calls are required. The test also checks that string-membership tests against assistant text do not drive control flow. Artifact: `tests/test_antipatterns.py`.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `classify_claim` and `assess_severity` both accept a rationale and a categorical decision, but they validate against different allowed sets: `CLAIM_TYPES` for claim classification and `SEVERITIES` for severity. Their tool contracts distinguish the two decisions instead of treating them as arbitrary text. The tool layer also returns structured errors containing `is_error`, `error_category`, `is_retryable`, and `message`, allowing the agent to distinguish a permanent validation problem from a retryable failure. Artifact: `claims_intake/tools.py`.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → `claim_02_stolen_bike` took 5 turns and ended with `end_turn`. The complete System 1 run reported an estimated total cost of $0.1214, while the README gives an approximate reference cost of about $0.05 on Haiku 4.5. The README value is an estimate; actual model tool-use trajectories and token consumption can vary between runs. Artifacts: `runs/20260924_170558/traces/claim_02_stolen_bike.jsonl`, `system1-run-output.txt`, and the exercise README.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → The baseline was 38,708 tokens and the assembled context was 16,858 tokens, a 56.45% reduction. The `active` section dominates at 15,789 tokens, compared with 204 for `case_facts`, 430 for `resolved_refund`, and 453 for `resolved_subscription`. The active section is kept as the primary working context because it represents the current issue that the copilot must reason about. Artifact: `runs/20260924-172321/budget.json`.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Completed historical segments are compressed into summaries, while the active segment and compact case facts are preserved for current reasoning. The resulting sections were 204 `case_facts`, 430 `resolved_refund`, 453 `resolved_subscription`, and 15,789 `active` tokens. This keeps the largest amount of context focused on the current issue while substantially reducing historical context. Artifact: `runs/20260924-172321/budget.json`.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The main evaluation passed 6/6 questions. In the control evaluation, Q1 unexpectedly passed while Q6 failed: the control answered `Active issue` instead of the exact expected `in_progress`. This demonstrates that structured preservation of facts can matter for exact state retrieval even when a less-structured context can still produce a semantically related answer. Artifacts: `runs/20260924-172321/eval.jsonl` and `runs/20260924-172321/eval_control.jsonl`.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The API rule uses the glob `src/api/**/*` in its YAML `paths` frontmatter. This allows the convention to activate for matching API files without requiring a `CLAUDE.md` in every relevant directory. The project's `CLAUDE.md` explicitly describes `.claude/rules/` with path-scoped globs as the preferred mechanism for cross-cutting conventions. Artifact: `.claude/rules/api.md` and `CLAUDE.md`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The deploy-check skill declares `context: fork` and a read-only `allowed-tools` list containing `Read`, `Grep`, `Glob`, and restricted Git/GitHub commands. Forking keeps verbose discovery output out of the main session, while the allowlist prevents the skill from modifying files, pushing, or deploying. Without these controls, discovery could consume the main context and the validation workflow would have a larger mutation surface. Artifact: `.claude/skills/deploy-check/SKILL.md`.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator returned `OK`. A project-level example is `./CLAUDE.md` or `.claude/rules/api.md`, which is part of the repository configuration. A user-level example is `~/.claude/CLAUDE.md` or a user-level command/skill, which is personal and is not shared through the project repository. Artifact: validator output `OK` and `CLAUDE.md`.

---

## Part 2 — Synthesis

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm tier contained 40 defects, while the shift pipeline queries only the relevant slice through `WarmStore.defects_since(...)`. The SQL-side query filters defects by timestamp and applies a limit before the model invocation, so the model receives only the selected slice rather than the full warm-tier history. Artifact: `shift_monitor/warm.py` and `shift_monitor/pipeline.py`.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.py` uses a 30-minute threshold: a recent partial run can resume, while a stale partial run starts fresh with its captured findings injected as a summary. A fresh start avoids relying on an old execution context that may no longer reflect the current shift state. The design therefore preserves useful findings without blindly continuing stale work. Artifact: `shift_monitor/recovery.py`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The recorded `hot_state.json` was 643 bytes, well below the 5,120-byte budget enforced by `state.py`. Keeping the state small limits the amount of cross-shift context carried into future invocations and prevents persistent state from growing without bound. This matters for an indefinitely running shift-monitoring system because every future run needs a predictable, bounded working state. Artifact: System 4 `data/hot_state.json` and `shift_monitor/state.py`.

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model:** System 1's agentic execution uses the model response and tool calls to determine the next claim-processing action; evidence is in `claims_intake/loop.py` and the claim traces.
    → **Harness:** System 1's `claims_intake/loop.py` deterministically enforces the `stop_reason` control flow, while `tools.py` validates and executes tool calls.
    → **Orchestration:** System 4's `shift_monitor/pipeline.py`, `warm.py`, `recovery.py`, and `state.py` manage database selection, invocation, persistence, and cross-shift recovery.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code and one guided by prompt. When is each right?
    → System 4's 5,120-byte hot-state limit is deterministic: `state.py` rejects state that exceeds the configured budget. By contrast, the rich invocation in `invocation.py` guides the model to produce a shift report and updated hot-state JSON. Deterministic enforcement is appropriate for hard resource and safety boundaries, while prompts are useful for guiding model-generated reasoning and output within those boundaries. Artifacts: `shift_monitor/state.py` and `shift_monitor/invocation.py`.

16. **Context, two faces.** Compare context management in System 2 and System 4 with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages context within a session, reducing 38,708 baseline tokens to 16,858, a 56.45% reduction. System 4 manages context across shifts, keeping the hot state at 643 bytes and querying the 40-defect warm tier for only relevant records. Both preserve information needed for the current task while avoiding unnecessary historical context, but System 2 uses summarization/compression whereas System 4 uses tiered persistence and SQL pre-filtering. Artifacts: System 2 `budget.json`; System 4 `hot_state.json`, `warm.py`, and `state.py`.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → System 4's tests verify the crash-recovery truth table, including incomplete, complete, and missing manifests and the 30-minute resume threshold. A normal successful shift cannot demonstrate whether the system behaves correctly after a partial crash or stale execution. This matters before shipping because failure recovery is part of the system's behavior even though it is not visible during a successful path. Artifact: System 4 test suite, which completed with 33 passed.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 4, an orchestration failure can affect the processing and persisted state of a shift because `pipeline.py` coordinates the database query, model invocation, and hot-state update. The implementation limits that risk through SQL pre-filtering, the 5,120-byte state budget, atomic state writes, and a single model invocation per shift. The provided implementation does not define a dedicated named kill switch, so the explicit enforcement boundaries are the pipeline invocation and the deterministic state/recovery controls. Artifacts: `shift_monitor/pipeline.py`, `shift_monitor/state.py`, and `shift_monitor/recovery.py`.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it.
    → The first System 1 environment encountered an Anthropic SDK/HTTPX compatibility problem: `anthropic 0.39.0` attempted to use the `proxies` argument that the installed `httpx 0.28.1` client did not accept. I fixed the environment by installing a compatible HTTPX version with `pip install "httpx<0.28"`. The environment also initially required configuring the provided Anthropic/Vocareum credentials before the live model run could execute. Artifact: System 1 environment setup/run history.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would make the historical replay timestamp explicit in System 4 instead of relying on the runtime clock when replaying a recorded response. The recorded fixture was for 2026-04-30, while the live execution used the current runtime date of 2026-09-24 and consequently reported `new=0`. Making the reference timestamp an explicit replay input would make historical verification deterministic while leaving production behavior unchanged. Artifacts: `shift_monitor/pipeline.py`, `fixtures/recorded_responses/shift_C_2026-04-30.json`, and `system4-run-output.txt`.
