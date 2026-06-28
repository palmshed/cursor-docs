---
---

# Principal Engineering Review - Level 2 Planner

Date: 2026-06-27

Verdict: APPROVE WITH CONDITIONS

## Executive Summary

Level 2 moved the system in the right direction: the planner now has an explicit investigation loop, recovery is attempted before premature failure, and retrieval quality improved through deterministic `find`, `references`, `read`, and grep fallback paths. These are directionally correct decisions for the mission of evidence before opinion.

I would not treat the current architecture as stable for the next year without conditions. The implementation is still more of an execution state machine than a canonical investigation engine. `ExecutionEngine` owns classification, routing, recovery, evidence parsing, confidence updates, metrics, UI notifications, result synthesis, and `InvestigationSession` derivation in one loop. That coupling will become the dominant cost if the project doubles.

The most important architectural gap is that `InvestigationSession` is documented as the canonical model, but it is currently derived after execution from `ExecutionResult` rather than owning the investigation state. Replay, telemetry, diagnostics, and exporters still read multiple representations. As a result, the system can pass 43 regression scenarios while still lacking a durable, replayable, planner-owned model of "what was known, what remained unknown, why the next tool was selected, and why the planner stopped."

The long-term decision to add recovery is correct. The current shape of recovery is not yet correct enough to ship as the final Level 2 foundation.

## Strengths

1. Recovery exists before final failure.

   Evidence: `ExecutionEngine::execute()` tries recovery when completion is true but confidence is below 0.7, when the primary tool sequence is exhausted, and before a low-confidence stop (`src/services/execution_engine.cpp:1144-1192`, `1195-1207`, `1386-1433`). This directly addresses the Level 2 objective of reducing premature stops.

2. Retrieval has become more deterministic.

   Evidence: `CodebaseQuery` now prefers reference search for caller/reference questions, then directory-aware `find`, then grep fallback, then read (`src/services/execution_engine.cpp:589-635`). This matches the repo principle of deterministic retrieval before broad grep.

3. Evidence class gating is a useful abstraction.

   Evidence: `required_evidence()` maps goal types to evidence classes, and `check_completion()` refuses completion until required classes are present (`src/services/execution_engine.cpp:454-487`, `896-900`). This is a good foundation for completion rules.

4. Regression coverage exists and is cheap to extend.

   Evidence: scenario assertions cover outcomes, AI gating, minimum files, non-empty evidence, and required tools (`src/diagnostics/diagnostics.cpp:1077-1105` and following). This gives the team a practical harness for planner behavior.

5. The design intent is clearly documented.

   Evidence: `docs/engineering/planner_recovery.md` explains the recovery loop, strategy triggers, telemetry, and acceptance cases. This improves maintainability even where the implementation needs more separation.

## Weaknesses

1. `ExecutionEngine` is doing too much.

   Evidence: the same file and loop own goal classification (`src/services/execution_engine.cpp:300-398`), evidence requirements (`454-487`), tool selection (`493-690`), recovery strategy selection (`810-877`), execution/UI notifications (`1227-1233`), evidence parsing (`1235-1345`), confidence scoring (`1348-1384`), recovery execution (`1386-1433`), summary generation (`1442-1470`), metrics (`1499-1524`), outcome assignment (`1536-1546`), and `InvestigationSession` conversion (`1557-1615`).

   This is the main architectural risk. It will not scale cleanly when more goal types, tools, recovery strategies, or telemetry dimensions are added.

2. The planner is a hidden state machine encoded in string facts.

   Evidence: decisions depend on substring checks such as `find:noresults`, `grep:results`, `read`, `discovery`, `recovery:find_impl`, and path fragments like `.h"]` (`src/services/execution_engine.cpp:815-877`). This is brittle because facts are simultaneously user evidence, state flags, telemetry hints, and parser inputs.

3. `InvestigationSession` is not yet canonical.

   Evidence: the struct claims every representation is a view over it (`include/core/investigation_session.h:31-35`), but it is built from `ExecutionResult` after the loop completes (`src/services/execution_engine.cpp:1557-1615`). Diagnostics still export timeline and evidence from `TraceEvent` and `ExecutionResult` rather than the session (`src/diagnostics/diagnostics.cpp:605-778`). Replay serializes `last_investigation`, but separately logs recovery, trust, retrieval, confidence, outcome, execution path, and state (`src/services/replay_service.cpp:68-153`, `180-250`).

4. Recovery logic is duplicated inside the execution loop.

   Evidence: recovery execution and evidence recording are implemented once in the post-completion confidence branch (`src/services/execution_engine.cpp:1151-1187`) and again in the low-confidence branch (`1391-1426`), separate from the normal tool execution path (`1227-1345`). These paths can diverge silently.

5. Confidence thresholds are hard-coded and not calibrated to query class.

   Evidence: post-completion recovery uses `< 0.7` (`src/services/execution_engine.cpp:1150`), low-confidence stopping uses `0.2` (`1388`), and recovery tools add a fixed `0.5` confidence entry (`1183-1186`). These numbers may be acceptable for small regressions, but they are not yet justified as production stop conditions.

6. Current scenarios do not prove recovery behavior.

   Evidence: `scenarios/regressions/recovery_find_noresults.json` asserts only `insufficient_evidence`, `ai_called: false`, and non-empty evidence. `recovery_header_to_implementation.json` asserts only success and AI called. The runner supports `tools_used`, but does not assert recovery count, recovery strategy, confidence path, selected file, or stop reason (`src/diagnostics/diagnostics.cpp:1077-1105` and following).

## Risks

1. Scale risk: repeated full scans and broad grep will become expensive.

   On a 10 million line repository, the current find/reference/grep/read sequence needs stronger indexing, budgets, and cancellation semantics. The architecture currently encodes tool choice but not cost models, result limits, or repository-scale adaptive behavior in the planner state.

2. Correctness risk: string facts can create false positives.

   `has_fact_containing()` treats evidence as an untyped string bag. A tool output containing `read`, `grep:results`, or a path-looking fragment can affect planner state. As more tools are added, accidental state transitions become likely.

3. Replay risk: replay cannot reconstruct planner reasoning.

   `InvestigationSession` has `reasoning_steps`, but `from_result()` does not populate them (`include/core/investigation_session.h:47`, `src/services/execution_engine.cpp:1557-1615`). Replay can show what happened, but not reliably why the planner chose the next tool or why it stopped.

4. Telemetry risk: metrics are derived late and inconsistently.

   Recovery metrics are computed after execution from facts and history (`src/services/execution_engine.cpp:1499-1524`). Replay logs recovery, retrieval, trust, confidence, and session data as parallel structures. That makes metric drift likely.

5. Product risk: UI and diagnostics observe different investigation models.

   JSON output uses `InvestigationSession` when present (`src/diagnostics/diagnostics.cpp:557-577`), but timeline and evidence export use `TraceEvent` and `ExecutionResult` (`605-778`). A user can see different evidence surfaces for the same run.

## Blind Spots

1. Unknown tracking is not represented.

   The AGENTS invariant requires known facts, unknown facts, active hypotheses, rejected hypotheses, evidence required next, and confidence. `InvestigationSession` has goal, conclusion, tools, files, symbols, reasoning, and evidence summary, but no explicit unknowns, hypotheses, rejected hypotheses, next evidence requirement, or per-step confidence (`include/core/investigation_session.h:37-51`).

2. Recovery stop conditions are not explained in product terms.

   `MAX_RECOVERY = 3` and `MAX_ITERATIONS = 20` are hard-coded (`src/services/execution_engine.cpp:1137-1138`). There is no documented relationship to repo size, tool cost, query class, or telemetry saturation.

3. No first-class planner step model exists.

   A durable step should contain: known state before selection, unknown being reduced, selected tool, selection reason, expected evidence, actual evidence, confidence delta, and stop/recovery decision. Today these are scattered across facts, trace events, tool history, and metrics.

4. Architecture review mode is hard-coded to this repository.

   Evidence: `ArchitectureReview` searches for project-specific terms such as `AgentMode`, `MODE_`, `AuthProvider`, and `provider_label` (`src/services/execution_engine.cpp:665-690`). This is useful for current audits but should not be confused with a general architecture review planner.

5. Evidence quality is mostly class-based, not semantic.

   The planner can know it has `FileSearch` and `FileContent`, but not whether the read file actually answers the user's question. This is acceptable for Level 2 only if the limitation is explicit and measured.

## Recommendations

### Critical

1. Make `InvestigationSession` the source of truth or rename it.

   If it remains a derived summary, call it `InvestigationReport` or `InvestigationSummary`. If it is intended to be canonical, move authoritative investigation state into it and have `ExecutionResult`, replay, diagnostics, telemetry, and UI consume views derived from it.

   Future cost if ignored: high. Every new planner capability will add another adapter and another place where replay, UI, and telemetry disagree.

2. Introduce typed planner facts and planner steps.

   Replace string-only state transitions with typed records: tool attempt, tool result, evidence class, selected candidate, recovery attempt, stop reason, and confidence observation. Keep human-readable summaries as renderers, not planner state.

   Future cost if ignored: high. String-fact coupling will become the main source of non-deterministic planner bugs.

3. Extract recovery execution into one path.

   `select_recovery_tool()` can remain separate, but executing a recovery tool should use the same code path as executing a primary tool. Evidence recording and confidence updates should not be duplicated across branches.

   Future cost if ignored: medium to high. Duplicated recovery paths will drift as tools and metrics evolve.

### High

4. Split planner responsibilities out of `ExecutionEngine`.

   Keep `ExecutionEngine` as orchestration. Move goal classification, tool policy, completion policy, recovery policy, evidence ingestion, and confidence policy into separable components with explicit interfaces.

   Future cost if ignored: high. The current file will become the planner monolith and resist safe change.

5. Calibrate confidence and recovery thresholds with telemetry.

   Treat `0.7`, `0.2`, `3 recovery attempts`, and `20 iterations` as provisional. Track false success, false insufficient evidence, average tools, recovery success by strategy, and confidence delta by query class.

   Future cost if ignored: medium. Planner behavior will look stable in regressions but fail unpredictably on larger repositories.

6. Strengthen regression assertions for recovery.

   Add scenario expectations for recovery attempts, selected strategy, selected files, stop reason, confidence range, and tool order where behavior matters. The current recovery scenarios are too coarse to protect the design.

   Future cost if ignored: medium. Refactors can break recovery while preserving final outcome.

### Medium

7. Add explicit unknowns and evidence requirements to the session model.

   The planner invariant requires these fields. They should be visible in replay and diagnostics.

   Future cost if ignored: medium. The system will explain evidence after the fact but will not demonstrate investigation discipline.

8. Separate product-specific architecture review heuristics from general planner logic.

   Keep repository-specific review checks as a capability or profile, not a goal-type branch in the core engine.

   Future cost if ignored: medium. The planner will accumulate special-case audits that do not generalize.

9. Make replay a planner replay, not just event replay.

   Replay should reconstruct planner decisions, state transitions, and stop conditions. Current replay captures useful data, but not enough to answer why a specific decision was made.

   Future cost if ignored: medium. Debugging production behavior will require reading logs and inferring planner state manually.

### Low

10. Populate `reasoning_steps` or remove it until supported.

   An empty canonical field is misleading. Either fill it at every selection point or mark it as future work outside the serialized contract.

   Future cost if ignored: low, but it weakens trust in the model.

11. Normalize diagnostics to one investigation view.

   JSON, timeline, trace, and evidence export should not each reconstruct tools/files differently.

   Future cost if ignored: low to medium. User-facing inconsistencies will grow over time.

## Technical Debt

### Critical Debt

- Canonical model mismatch: `InvestigationSession` is documented as authoritative but derived from `ExecutionResult`.
  Estimated future cost: 2-4 weeks to unwind after more UI/replay/telemetry surfaces depend on divergent representations.

- String-fact planner state: planner decisions rely on substring searches over human-readable evidence.
  Estimated future cost: 2-3 weeks plus recurring regression risk as tools are added.

### High Debt

- `ExecutionEngine` monolith: too many planner, execution, telemetry, and rendering-adjacent responsibilities.
  Estimated future cost: 3-6 weeks to split safely once more capabilities land.

- Duplicated recovery execution and evidence ingestion.
  Estimated future cost: 1-2 weeks and likely behavioral drift.

- Under-calibrated confidence and recovery thresholds.
  Estimated future cost: ongoing production tuning, plus difficult incident analysis.

### Medium Debt

- Scenarios assert final outcomes more than planner decisions.
  Estimated future cost: 1 week to expand the harness; more if delayed until after regressions accumulate.

- Replay lacks full planner decision state.
  Estimated future cost: 1-3 weeks depending on how much telemetry schema changes.

- Product-specific architecture review heuristics live in core planner selection.
  Estimated future cost: 1 week to extract while still small.

### Low Debt

- Empty or underused session fields such as `reasoning_steps` and `symbols_found`.
  Estimated future cost: days if addressed soon.

- Multiple diagnostics renderers reconstruct tools/files independently.
  Estimated future cost: days to one week.

## Scalability

The first bottlenecks are likely to appear in this order:

1. Planner complexity.

   More goal types and tools will make the switch-based planner harder to reason about before raw runtime becomes the limiting factor.

2. Retrieval and ranking.

   Repository-wide find/reference/grep scans are acceptable now, but a 10 million line repository needs indexing, result caps, and cost-aware tool selection.

3. Evidence volume.

   Evidence facts store truncated tool output and string flags together. Large outputs will either lose important evidence or pollute planner state.

4. Replay and telemetry schema drift.

   Parallel representations will diverge as soon as new planner fields are added.

5. UX consistency.

   Different consumers already render from different sources. This will become visible once users rely on investigation timelines.

## Missing Capabilities Required By The Current Architecture

These are not new product features. They are capabilities the current architecture needs to remain coherent:

1. Typed investigation step recording.

   Required because current recovery, confidence, and evidence decisions cannot be replayed or audited from `InvestigationSession`.

2. Cost-aware retrieval budgets.

   Required because the planner already sequences multiple repository-wide tools, and fixed iteration counts do not represent actual cost.

3. Recovery strategy telemetry by strategy ID.

   Required because recovery exists, but metrics do not yet answer which strategy fired, why, and whether that strategy improved confidence or evidence completeness.

4. Semantic evidence validation.

   Required because evidence classes prove that a file was searched/read, not that the read content answered the user's question.

5. Unified investigation rendering.

   Required because UI, JSON, evidence export, replay, and timeline cannot remain independent interpretations of the same run.

## Long-Term Outlook

The direction is right. Level 2 should continue to prioritize planner behavior over adding capabilities. However, the current implementation should be treated as a successful prototype of planner recovery, not the final planner architecture.

If the team makes `InvestigationSession` authoritative, introduces typed planner steps, and extracts policy components from `ExecutionEngine`, this architecture can scale to a larger product. If the team continues adding strategies and telemetry into the current loop, the system will become harder to trust precisely when it needs to explain itself most.

The one-year-correct decision is not "add recovery into `ExecutionEngine`." The one-year-correct decision is "make recovery a typed, replayable planner transition in a canonical investigation model." The current code is a useful bridge to that design, but not the destination.

## Final Verdict

APPROVE WITH CONDITIONS

Conditions before treating Level 2 as production-ready:

1. Establish a truly canonical investigation model or rename the current session summary.
2. Replace string-fact planner state with typed planner events/facts.
3. Unify normal and recovery tool execution paths.
4. Add recovery-specific regression assertions.
5. Calibrate confidence and recovery thresholds against telemetry.

I would approve continued internal development on this foundation. I would not approve shipping it as a production engineering product until those conditions are addressed.
