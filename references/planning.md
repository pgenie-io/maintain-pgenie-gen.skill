# Planning and execution

Produce a hierarchical plan of **isolated tasks**, each executed in a **fresh agent session**. A task may itself be a plan consisting of subtasks, decomposed and dispatched the same way, recursively.

## Why the architecture makes this safe

The generator architecture is designed for module independence:

- Templates are pure `Params -> Text`, cannot import the model or other templates, and are independently type-checkable with `dhall type`.
- Interpreters have a fixed shape (`Sdk.Sigs.Interpreter.module InterpreterConfig.Type Input Output run`) and communicate only via primitives and Structures.
- `Interpreters/Name.dhall` is the single owner of identifier handling.

So module boundaries are task boundaries. One template or one interpreter per task is the natural grain.

## Task isolation rules

- **Self-contained spec.** Each task's prompt must carry everything the fresh session needs: the exact file(s) to produce, their Input/Output or Params types, the relevant grilling decisions, the relevant excerpt of the design artifact (the target text the template must reproduce), and the architecture conventions that apply. The subagent must not need to re-derive context or re-interview the user.
- **Declared interfaces first.** Before dispatching, fix the type signatures at every task boundary (each interpreter's Input/Output, each template's Params). Interface changes require re-planning, not silent drift inside a task.
- **Verifiable in isolation.** Every task ends by running `dhall type` and `dhall format --transitive` on its files. A task that cannot be checked without its siblings is cut wrong - merge or re-split.
- **No shared mutable state.** Tasks touching the same file are the same task or are sequenced.

## Suggested decomposition (create flow)

Dependency order; tasks within a step are independent and run in parallel:

1. **Scaffold**: repo layout, `gen/Deps/` (pinned Sdk, Prelude, Lude, Typeclasses - copy pins from java.gen), `gen/Config.dhall`, `gen/InterpreterConfig.dhall`, empty `gen/Gen.dhall` wiring per architecture doc.
2. **`Interpreters/Name.dhall`**: all casing/escaping per grilling category 3.
3. **Type mapping interpreters**: `Primitive`, `Scalar`, `Value` per grilling category 4.
4. **Templates**: one task per template (statement module, type modules, test modules, manifest, README, …). Each reproduces its target text from the design artifact, flush-left per the unindented-output rule.
5. **Interpreters, bottom-up**: leaf interpreters, then `Query`/`CustomType`, then `Project` (which owns cross-file aggregation). Wire `Compiled.nest` labels and the skip-with-warning protocol at skippable units.
6. **Integration**: `gen/compile.dhall`, `gen/Gen.dhall`, `tests/Exhaustive.dhall` applying the generator to `Sdk.Fixtures.Exhaustive`, first materialization.
7. **Convergence loop**: diff-driven fixes until verification passes (see verification.md). Convergence fixes are dispatched as fresh tasks, one per divergent module cluster.

In the update flow, plan only the affected subtree: diff the design artifact change, map divergent files back to owning templates/interpreters, one task per affected module.

## Execution

- Dispatch each task to a fresh agent session (in Claude Code: the Agent tool, general-purpose). Run independent tasks in parallel.
- A task that is itself a plan is decomposed by the orchestrator (or by its own session if the harness allows nested agents) and its subtasks dispatched under the same rules.
- In a harness without subagents, execute tasks sequentially in-session - still one at a time, still self-contained specs, so any task can be re-run cold.
- The orchestrating session owns integration: after each wave of tasks, type-check the whole tree (`dhall type` on `gen/Gen.dhall`) before dispatching the next wave.
