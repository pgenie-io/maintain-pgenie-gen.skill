# Grilling the generator author

Interview the user before planning. The design artifact is one sample of a function; the grilling pins down the function.

## Protocol

- One question at a time. Provide your recommended answer with every question, with reasoning.
- Never ask what you can answer yourself by reading the design artifact, the architecture doc, or the reference generators. Explore first, ask second.
- Record every decision as you go; the accumulated decisions become constraints in the plan. In the update flow, re-ask only categories the design change touches.
- Close with an open question: any constraints, non-goals, or past failure modes the categories missed.

## Mandatory categories

Ask in this order - later categories depend on earlier answers.

### 1. Scope of generation

Which files in the design artifact are generated, which are static scaffolding the generator emits verbatim, and which are excluded from the convergence diff entirely (README, `.gitignore`, license)? The agreed exclusion list is used verbatim by verification.

### 2. Config surface

What belongs in `Config.dhall` (package/namespace prefix, dependency versions, feature toggles like java.gen's `useOptional`) versus hardcoded? Existing generators keep this minimal - recommend minimal, and recommend that every config option have a default exercised by the fixture test.

### 3. Name handling

Casing convention per identifier position (types, functions, fields, files, modules), reserved-word escaping, and collision policy. All of it lands in `Interpreters/Name.dhall` - nowhere else. Derive the visible cases from the design artifact; ask only about positions the demo doesn't exercise.

### 4. Type mapping

The full table: pgenie primitive → target type, including nullability representation, arrays, composites, enums, ranges/multiranges - and the explicitly **unsupported** set. Derive what the demo shows; grill on the rest of the primitive set the SDK model defines.

### 5. Unsupported-construct policy

Confirm the architecture doc's skip-with-warning protocol (errors at leaf interpreters propagate to skippable units - queries, custom types - and become warnings). Ask whether any construct should hard-fail the whole run instead. Unsupported input must never produce partial output.

### 6. Runtime dependencies

Which driver/libraries the generated code depends on, pinned versions, minimum language/toolchain version. These appear in the generated manifest (pom.xml, Cargo.toml, …) and constrain the templates.

### 7. Extrapolation rules - the critical category

The demo doesn't exercise every feature. For each pattern in the design artifact, establish how it generalizes: what is per-query, per-parameter, per-result-column, per-custom-type, per-enum-variant? What does a query with zero parameters produce? Zero result columns? Twenty? Push until every template's parameter space is defined, not just the sample point the demo shows. This is the category agents most often skip and the one that determines whether the generator works beyond the demo.

### 8. Release

Repo name (`<lang>.gen` is the ecosystem convention), license, SemVer scheme, and whether releases follow the java.gen workflow set (CHANGELOG with `# Upcoming` section, `bump.yml`, `release.yml` attaching `resolved.dhall`). Recommend copying the java.gen setup unchanged.
