# Verification and release

"Done" has four layers, checked in order. Each layer is cheaper than the next; never advance past a failing layer.

## 1. Convergence diff (the acceptance test)

Materialize the generator's output for the demo fixture:

```bash
dhall to-directory-tree --file tests/Exhaustive.dhall --output demo-output --allow-path-separators
```

`tests/Exhaustive.dhall` applies the generator to `Sdk.Fixtures.Exhaustive` (the pre-parsed demo project shipped with the pinned SDK) - see java.gen's for the shape. Then:

```bash
diff -r demo-output <design-artifact> # minus the agreed exclusion list
```

Required: **byte-identical**, modulo only the exclusion list agreed in grilling category 1. "Close enough" is not convergence - whitespace and ordering differences are real divergences in generated code.

Commit `demo-output/` to the generator repo so future generator changes review as output diffs.

## 2. Toolchain build

The materialized artifact builds with the target toolchain (`mvn compile`, `cargo build`, `dotnet build`, …). If layer 1 passes and the design artifact was validated up front, this passes by construction - a failure here means the spec-validation phase was skipped or the exclusion list hides a load-bearing file.

## 3. Tests

Run the artifact's test suite (`mvn verify`, `cargo test`, …). Integration tests need Docker + Postgres with the demo migrations applied (the reference designs use Testcontainers, which handles this given a Docker daemon). If Docker is unavailable locally, say so explicitly and rely on layer 4 - never claim tests passed without running them.

## 4. CI

Adapt java.gen's `.github/workflows/` unchanged in structure:

- **`ci.yml`** - on every push/PR: matrix over `tests/*`, regenerate output, fail on dirty diff against the committed `demo-output/`, then build and test the artifact with the target toolchain (swap the Java/Maven steps for the target's).
- **`release.yml`** - manual dispatch with a SemVer version: validates `CHANGELOG.md` (must contain a `# Upcoming` section), runs CI, freezes the generator into a single `resolved.dhall`, tags, and attaches it to a GitHub release.
- **`bump.yml`** - computes the next patch/minor/major version from tags and calls release.

Also maintain `CHANGELOG.md` (top section `# Upcoming`, promoted to `# vX.Y.Z` on release).

## Dhall hygiene (every task, not just the end)

- `dhall type --file gen/Gen.dhall` - the whole tree type-checks.
- `dhall format --transitive --file gen/Gen.dhall` - canonical formatting.
- Every `Interpreters/` module ends with `Algebra.module Input Output run`; every `Templates/` module ends with `Algebra.module Params run`. No exceptions.
- Interpolation over concatenation: `"Optional<${t}>"`, not `++`.

## Release

Users consume the generator by URL in their pgenie project config:

```yaml
artifacts:
  <lang>:
    gen: https://github.com/<owner>/<lang>.gen/releases/download/vX.Y.Z/resolved.dhall
```

Releasing = running the `bump` (or `release`) workflow. Before the first release, confirm with the user: repo owner/name, license, and that CI is green on the default branch.
