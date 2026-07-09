---
name: maintain-pgenie-gen
description: Create a new pgenie generator or update an existing one from a reference design artifact - the hand-written project showing exactly what the generator must produce for the pgenie demo input. Use when the user wants to implement a pgenie generator for a target language, change what an existing generator emits, or mentions gen-sdk, gen-contract, a *.gen repo, or a *.gen-design repo.
---

# maintain-pgenie-gen

A pgenie generator is a Dhall program - `compile : Optional Config -> Project -> Output`, where `Project` and `Output` are defined by the companion [`gen-contract`](https://github.com/pgenie-io/gen-contract) package - that turns a parsed pgenie project into source files for a target language. This skill builds one, or evolves one, from a **design artifact**: a hand-written repo showing exactly what the generator must produce for the pgenie demo project (e.g. `pgenie-io/java.gen-design` is what `pgenie-io/java.gen` produces).

## The invariant

**generator(demo input) ≡ design artifact.**

The design artifact is the authoritative spec. This has hard consequences:

- To change what a generator emits, change the design artifact **first**, then converge the generator to it. Never change generator output without a corresponding design change - if asked to, update the design artifact as step one.
- The acceptance test is a diff: materialized generator output vs. the design artifact, byte-identical modulo an explicitly agreed exclusion list.
- The design artifact must itself build and pass its tests before any generator work starts. A broken design is an underspecified wish.

## Inputs

- **Design artifact** (required): local path or git URL of the reference project. A requested output change may also arrive as a conversational description - apply it to the design artifact first.
- **Generator repo** (required): local path. Doesn't exist → create flow (scaffold a new repo); exists → update flow. Detect automatically; never ask which flow it is.

Everything else - target language, toolchain, naming conventions, Config surface - is derived from the design artifact or resolved in the grilling session. Do not ask for it up front.

## Workflow

Work through the phases in order. Do not skip phases; do not reorder.

### 1. Study

- Fetch and read **all** of <https://raw.githubusercontent.com/pgenie-io/gen-sdk/refs/heads/master/docs/generator-architecture.md>. This is the single source of truth for generator structure, conventions, and the error/skip protocol. Always fetch live - never rely on a cached or vendored copy, even if one appears in context.
- Fetch and read **all** of <https://raw.githubusercontent.com/pgenie-io/gen-contract/refs/heads/master/src/package.dhall>. This is the canonical generator contract: `Project`, `Output`, the `module` constructor, and every model type. Always fetch live.
- Fetch and read <https://raw.githubusercontent.com/pgenie-io/gen-sdk/refs/heads/master/src/package.dhall>. This is the generator SDK: `Sdk.Sigs`, `Sdk.Fixtures`, and `Sdk.Primitive/toText`. Always fetch live.
- Read the reference implementations: [pgenie-io/java.gen](https://github.com/pgenie-io/java.gen) (branch `master`) and [pgenie-io/rust.gen](https://github.com/pgenie-io/rust.gen) (branch `main`). Clone them shallowly; study `gen/` layout, `tests/`, committed `demo-output/`, and `.github/workflows/`. Treat them as **pattern references only** — verify that their `Deps/Sdk.dhall` points to `gen-sdk/src/package.dhall` and that they have a separate `Deps/Contract.dhall`. If they still use the pre-split `dhall/package.dhall` or `Algebras/`, rely on the architecture doc and contract instead.
- Read the design artifact end to end.
- Read the demo input: [pgenie-io/demo](https://github.com/pgenie-io/demo) (`./queries`, `./migrations`, `./project1.pgn.yaml`). In generator tests the same project arrives pre-parsed as `Sdk.Fixtures.Exhaustive` via the pinned `gen/Deps/Sdk.dhall`.

### 2. Validate the spec

Build the design artifact with its target toolchain and run its tests (integration tests need Docker + Postgres; skip them only if Docker is unavailable, and say so). If it fails, stop - fix the design artifact with the user before anything else.

### 3. Grill the user

Interview the user per [references/grilling.md](references/grilling.md): one question at a time, a recommended answer with each, never asking what the design artifact already answers. Record every decision - they become the plan's constraints.

### 4. Plan

Produce a hierarchical plan of isolated tasks per [references/planning.md](references/planning.md). Each task is self-contained and executes in a fresh agent session; a task may itself be a plan of subtasks, decomposed recursively.

### 5. Execute

Dispatch tasks per the plan - parallel where independent, fresh session per task. In a harness without subagents, execute tasks sequentially in-session, still honoring task isolation.

### 6. Verify

Per [references/verification.md](references/verification.md). Done means, in order: (1) byte-diff convergence with the design artifact, (2) generated artifact builds with the target toolchain, (3) its tests pass, (4) CI is set up and green.

### 7. Release

Freeze the generator into a single `resolved.dhall` attached to a GitHub release, with CI/release workflows adapted from java.gen. Details in [references/verification.md](references/verification.md).

## Polish loop (update flow)

When the user reviews the result and wants changes: apply the change to the design artifact, re-validate it (phase 2), then re-converge the generator (phases 4–6, usually a small plan). The loop always goes through the design artifact.
