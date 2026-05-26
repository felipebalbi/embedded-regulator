# AGENTS.md

Operational guide for AI coding agents (GitHub Copilot, Claude Code, Cursor,
Aider, Codex, etc.) contributing to **`embedded-regulator`**.

This file complements—not replaces—`README.md`, `CONTRIBUTING.md`,
`CODE_OF_CONDUCT.md`, and `.github/copilot-instructions.md`. When this file
and `copilot-instructions.md` disagree, `copilot-instructions.md` wins for
commit-message and AI-attribution rules; this file wins for everything else.

---

## 1. Project at a Glance

- **Name:** `embedded-regulator`
- **Purpose:** A small, `no_std`-friendly Hardware Abstraction Layer (HAL)
  trait crate that describes how to enable and disable power regulators on
  embedded SoCs. Think `embedded-hal`-style: traits only, no drivers.
- **Crate type:** Single library crate (`src/lib.rs`). **Not a workspace.**
- **Edition:** `2021`
- **MSRV:** `1.75` (enforced by the `msrv` CI job; bump deliberately, not casually)
- **License:** MIT
- **Default target:** host (`cargo check`) plus `thumbv8m.main-none-eabihf`
  for `no_std` validation in CI.
- **Owner org:** `openDevicePartnership`
- **Upstream:** <https://github.com/openDevicePartnership/embedded-regulator>

### Public surface (today)

```text
pub trait  Error      : core::fmt::Debug { fn kind(&self) -> ErrorKind; }
pub enum   ErrorKind  { Other }           // #[non_exhaustive]
pub trait  ErrorType  { type Error: Error; }
pub trait  Regulator  : ErrorType {
    async fn enable(&mut self)  -> Result<(), Self::Error>;
    async fn disable(&mut self) -> Result<(), Self::Error>;
}
```

Blanket `&mut T` impls are provided for both `ErrorType` and `Regulator`.
`core::convert::Infallible` implements `Error` so drivers that cannot fail
can use it as their associated `Error` type.

### Feature flags

| Feature | Default | Effect |
| --- | --- | --- |
| `defmt` | off | Adds `dep:defmt`; derives `defmt::Format` on `ErrorKind`. |

There is exactly one optional dependency (`defmt`). `cargo hack
--feature-powerset check` is the source of truth for valid feature
combinations.

---

## 2. Repository Layout

```text
.
├── AGENTS.md                  <- you are here
├── CODEOWNERS
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── Cargo.lock
├── Cargo.toml
├── LICENSE                    <- MIT
├── README.md
├── SECURITY.md
├── deny.toml                  <- cargo-deny config (licenses, bans, sources)
├── rustfmt.toml               <- 120 cols, StdExternalCrate import grouping
├── src/
│   └── lib.rs                 <- the entire public API
└── .github/
    ├── copilot-instructions.md
    └── workflows/
        ├── check.yml          <- fmt, clippy, doc, hack, deny, msrv
        └── nostd.yml          <- cargo check on thumbv8m.main-none-eabihf
```

Everything an agent touches will almost always be `src/lib.rs`,
`Cargo.toml`, `deny.toml`, `README.md`, or one of the workflow files.

---

## 3. Golden Rules (read before editing anything)

1. **`no_std` always.** `#![cfg_attr(not(test), no_std)]` is non-negotiable.
   Do not pull in `std`, `alloc`, or anything that transitively forces them.
   Use `core::` paths exclusively (`core::fmt::Debug`, not `std::fmt::Debug`).
2. **Traits only.** This crate defines abstractions. Do not add concrete
   drivers, example boards, runtime executors, or peripheral access code.
   Driver crates live elsewhere and depend on this one.
3. **`async fn` in traits is intentional.** `#![allow(async_fn_in_trait)]`
   is set crate-wide. Do not introduce `async-trait`, `Box<dyn Future>`, or
   any allocation-based async machinery.
4. **Keep it tiny.** New dependencies require strong justification. Every
   dep must be `no_std`-compatible and license-compatible with `deny.toml`
   (`MIT`, `Apache-2.0`, `Unicode-DFS-2016`).
5. **Stable + MSRV both build.** Any feature you use must compile on Rust
   `1.75`. Don't reach for newer language features without bumping MSRV in
   `Cargo.toml`, `.github/workflows/check.yml` (the `msrv` matrix), and
   documenting *why* in the workflow comment.
6. **Preserve blanket `&mut T` impls.** When adding methods to `Regulator`
   (or any new trait), forward them through the existing `impl<T: …>` block
   so downstream `&mut T` users keep working.
7. **`#[non_exhaustive]` on public enums.** `ErrorKind` is `#[non_exhaustive]`
   on purpose so new variants are not SemVer-breaking. New error enums must
   follow suit.
8. **Defmt is opt-in.** Any new public type that derives `Debug` should also
   gate `#[cfg_attr(feature = "defmt", derive(defmt::Format))]` so embedded
   users get logging without paying for it by default.
9. **No `unwrap()` / `expect()` / `panic!()` in library code.** Return
   `Result<_, Self::Error>` instead. Tests may panic.
10. **No I/O, no clocks, no global state.** This is a pure abstraction layer.

---

## 4. Coding Conventions

### Formatting

`rustfmt.toml` says:

```toml
group_imports     = "StdExternalCrate"
imports_granularity = "Module"
max_width         = 120
```

That means imports are grouped into three blocks (std/core, external
crates, this crate) with blank lines between, collapsed at the module
level (`use core::fmt::{Debug, Display};`, not one line per item), and
lines wrap at 120. Always run `cargo fmt` before committing. Note that
the `group_imports` and `imports_granularity` keys require a nightly
`rustfmt`; on stable they emit a warning and are ignored, which is
acceptable. CI runs `cargo fmt --check` on stable, so the warning is
informational only.

### Documentation

- Every public item gets a `///` doc comment. The CI `doc` job runs
  `cargo doc --no-deps --all-features` with `RUSTDOCFLAGS=--cfg docsrs`,
  so broken intra-doc links **fail the build**.
- Prefer first-sentence summary + blank line + details (rustdoc convention).
- Use `[Type]` intra-doc links for types defined in this crate.
- When adding new feature-gated items, annotate them with
  `#[cfg_attr(docsrs, doc(cfg(feature = "x")))]` so docs.rs renders the
  feature badge.

### Naming

- Trait methods: lowercase verbs (`enable`, `disable`, `set_voltage`).
- Error kinds: nouns describing the failure category (`Overcurrent`,
  `Undervoltage`), not the operation that triggered them.
- Avoid `get_` prefixes (Rust API guidelines).

### Async

- All I/O-like methods are `async fn` returning `Result<…, Self::Error>`.
- Do not introduce executors, timers, or `core::task` plumbing. If a
  method needs a delay, take a `&mut impl embedded_hal_async::delay::DelayNs`
  (and add it as a dependency only if truly needed — currently not present).

### Errors

- Follow the `embedded-hal` error model already present:
  `Error` trait → `ErrorKind` enum → `ErrorType` associated type → concrete
  traits bound on `ErrorType`. New trait additions should reuse this
  pattern, not invent a parallel one.

---

## 5. The Workflow Loop for Agents

Before declaring a task done, run **all** of the following from the repo
root and ensure each passes:

```powershell
cargo fmt --check
cargo clippy --all-features --all-targets -- -D warnings
cargo check
cargo check --target thumbv8m.main-none-eabihf --no-default-features
cargo hack --feature-powerset check
cargo doc --no-deps --all-features
cargo test
```

Notes:

- `cargo hack` requires `cargo install cargo-hack` once. If unavailable in
  the agent environment, document the omission in the PR description.
- The `thumbv8m.main-none-eabihf` target needs `rustup target add
  thumbv8m.main-none-eabihf` once.
- `cargo test` currently has no `#[test]` functions, but the invocation
  still validates that `cfg(test)` compiles with `std` available.
- `cargo deny check --all-features` mirrors CI; run it whenever
  `Cargo.toml`, `Cargo.lock`, or `deny.toml` changes.
- The `clippy` CI job uses `giraffate/clippy-action`, which effectively
  runs `cargo clippy` on the PR diff. Treat all clippy warnings as errors
  locally (`-D warnings`).

### Quick "did I break anything" matrix

| Change touched… | Re-run at minimum |
| --- | --- |
| `src/**` | fmt, clippy, check, nostd-check, doc, test |
| `Cargo.toml` | the above **plus** `cargo hack` and `cargo deny check` |
| `deny.toml` | `cargo deny check --all-features` |
| `.github/workflows/**` | `actionlint` if available; otherwise eyeball |
| `*.md` only | none required; do not commit unrelated reformatting |

---

## 6. Git, Commits, and Branching

### Commit message format (from `copilot-instructions.md`)

- Subject line: capitalized, ≤ 50 chars, imperative mood
  (`Add voltage trait`, not `Added voltage trait`).
- Blank line between subject and body.
- Body wrapped at 72 chars, explaining **what** and **why**, never **how**.
- One logical change per commit.

### Required trailers

Every AI-assisted commit **must** include:

```text
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

For example:

```text
Assisted-by: GitHub Copilot:claude-opus-4.7
```

- Verify the *actual* model you are running before composing the trailer.
  Do not copy a model name from a previous session.
- Basic tools (git, cargo, your editor) are **not** listed as trailing
  tools — only specialized analyzers (e.g., `clippy`, `coccinelle`,
  `cargo-deny`) when they materially shaped the change.
- AI agents **must not** add `Signed-off-by:` trailers. Only humans
  certify the Developer Certificate of Origin.

### Identity

Set author identity per-invocation, never globally:

```powershell
git -c user.name="Felipe Balbi" `
    -c user.email="felipe.balbi@microsoft.com" `
    commit -m "..."
```

### Branching

- Work on a topic branch off `upstream/main`
  (`git checkout -B my-topic upstream/main`).
- Push to your fork (`origin`), never to `upstream`.
- **Never force-push** to a shared branch. If history needs rewriting
  on a personal branch, do so before pushing and flag it in the PR.

---

## 7. CI Reference

Two workflows live under `.github/workflows/`:

### `check.yml`

| Job | Toolchain | What it runs |
| --- | --- | --- |
| `fmt` | stable | `cargo fmt --check` |
| `clippy` | stable, beta | `cargo clippy` via `giraffate/clippy-action@v1` |
| `doc` | nightly | `cargo doc --no-deps --all-features` with `RUSTDOCFLAGS=--cfg docsrs` |
| `hack` | stable + `thumbv8m.main-none-eabihf` target | `cargo hack --feature-powerset check` |
| `deny` | stable | `EmbarkStudios/cargo-deny-action@v2`, `check --all-features` |
| `msrv` | matrix `["1.75"]` | `cargo check` |

The `semver` job is commented out until the crate has a published
release; do not re-enable it speculatively.

### `nostd.yml`

Single job that runs
`cargo check --target thumbv8m.main-none-eabihf --no-default-features`.
If you add a feature that *requires* `std`, gate it with `#[cfg(feature
= "std")]` *and* update the no-std workflow to skip that feature.

---

## 8. Pilot-Specific Notes

Different AI coding assistants integrate with this repo differently.
The rules in section 3 apply to all of them; below are integration
specifics.

### GitHub Copilot / Copilot Chat / Copilot CLI

- Authoritative config: `.github/copilot-instructions.md`. Read it first.
- The CLI is the recommended way to run multi-step refactors; prefer it
  over the chat side-panel for changes touching more than one file.
- Use the `Assisted-by:` trailer documented in section 6.
- For PR reviews requested via `@copilot`, Copilot may only suggest;
  humans merge.

### Copilot Coding Agent (cloud)

- Runs in an ephemeral container. `.github/workflows/copilot-setup-steps.yml`
  is not currently present; if you add one, keep the install step minimal
  (`rustup target add thumbv8m.main-none-eabihf`, `cargo install
  cargo-hack cargo-deny`) and pin action versions.
- The agent should run the full verification matrix from section 5 before
  proposing a PR.

### Claude Code / Claude Desktop

- Honors this `AGENTS.md` as its primary spec.
- When running `cargo` commands, allow long timeouts (clippy + doc cold
  builds can exceed 2 minutes on a fresh container).

### Cursor / Windsurf / Continue

- Treat `AGENTS.md` as project rules. If the tool has a separate
  `.cursorrules` / `.windsurfrules` file, keep it as a thin pointer to
  this document; do not duplicate content.

### Aider

- Add `AGENTS.md`, `.github/copilot-instructions.md`, `Cargo.toml`, and
  `src/lib.rs` to the chat context for any non-trivial change.
- Use `/test cargo test` and `/lint cargo clippy --all-targets -- -D warnings`
  to wire Aider's autotest loop to the right commands.

### OpenAI Codex / `codex` CLI

- Same rules. The model is expected to run `cargo fmt` and `cargo clippy`
  itself before declaring a turn complete.

### Generic LLM agents (LangGraph, AutoGen, custom)

- The minimum viable verification gate is `cargo fmt --check && cargo
  clippy --all-targets --all-features -- -D warnings && cargo check
  --target thumbv8m.main-none-eabihf --no-default-features`.
- Anything that fails CI is "not done", regardless of how confident the
  model is.

---

## 9. Common Tasks — Worked Examples

### Adding a new trait method

1. Add the `async fn` signature to `Regulator` (or define a new trait
   following the `ErrorType`-bound pattern).
2. Add a forwarding `#[inline]` impl inside the existing
   `impl<T: Regulator + ?Sized> Regulator for &mut T` block.
3. Document the method with `///`, including units (volts, microamps,
   etc.) when relevant.
4. If the method introduces a new failure mode, add a variant to
   `ErrorKind` (remember it's `#[non_exhaustive]`).
5. Run the full section-5 matrix.

### Adding a feature flag

1. Declare it under `[features]` in `Cargo.toml`. Use `dep:foo` syntax to
   keep the namespace clean.
2. Gate the code with `#[cfg(feature = "foo")]` and the docs with
   `#[cfg_attr(docsrs, doc(cfg(feature = "foo")))]`.
3. Verify with `cargo hack --feature-powerset check` that every
   combination still compiles, including with `--no-default-features`
   on the embedded target.
4. Update the feature table in section 1 of this file.

### Adding a dependency

1. Confirm it is `no_std` (check its `Cargo.toml` for `default-features
   = false` opt-out or a `#![no_std]` attribute).
2. Confirm its license is in `deny.toml`'s `allow` list.
3. Run `cargo deny check --all-features`.
4. Justify the dep in the commit body.

### Bumping MSRV

1. Update `rust-version` in `Cargo.toml`.
2. Update the `msrv` matrix in `.github/workflows/check.yml`.
3. Update the inline comment in that workflow explaining *why* the bump
   was needed (existing comments reference `namespaced-features`, `fixed`,
   and `embedded-hal-async` — follow that style).
4. Mention the bump prominently in the PR description; it is a
   semver-minor break for downstream users on older toolchains.

---

## 10. Things Not To Do

- ❌ Add `std`, `alloc`, `tokio`, `async-trait`, `futures` (the trait
  crate has no need for any of them).
- ❌ Introduce concrete regulator drivers in this crate.
- ❌ Add `#[derive(Serialize, Deserialize)]` to public types without an
  opt-in feature flag and strong justification.
- ❌ Rename or remove public items without a SemVer-major version bump
  and a deprecation period.
- ❌ Force-push to shared branches.
- ❌ Add `Signed-off-by:` from an AI agent.
- ❌ Commit `Cargo.lock` changes without re-running `cargo deny check`.
- ❌ Globally configure `git config user.email`; always pass identity
  per-invocation with `git -c user.email=…`.
- ❌ Leave clippy warnings in place — fix them or `#[allow]` with a
  comment explaining why.
- ❌ Add CI matrix entries for OSes other than Linux unless we have a
  concrete need; this is a `no_std` library, the cross-platform surface
  is `rustc` itself.

---

## 11. Pointers

- `README.md` — user-facing summary.
- `CONTRIBUTING.md` — Open Device Partnership contribution policy and
  MIT licensing of contributions.
- `CODE_OF_CONDUCT.md` — community standards.
- `SECURITY.md` — how to report vulnerabilities.
- `.github/copilot-instructions.md` — canonical commit-message + AI
  attribution rules.
- `deny.toml` — license and dependency policy.
- `rustfmt.toml` — formatting policy.

When in doubt, prefer the smallest change that passes CI and document
your reasoning in the commit body. The crate's value is in being
boring, stable, and tiny.

## Model selection & cost discipline

Premium models (Opus, GPT-5 family, "high"/"xhigh" reasoning variants)
cost an order of magnitude more than standard models (Sonnet, Haiku,
mini). Most steps in a typical task do not need premium reasoning,
and over-using premium models wastes credits without improving
outcomes. The rules below apply to *all* model selection: your own
session, sub-agents launched via the `task` tool, and parallel work
launched via `/fleet`.

### Default posture

- **Default to the cheapest model that can do the job.** Reach for a
  premium model only when one of the escalation triggers below is hit.
- **Plan with premium, execute with cheap.** Spend at most one or two
  premium turns on design / planning, then downshift to a cheaper
  model for mechanical execution of the plan.
- **Never bump the model "just in case."** If you cannot articulate
  *why* a cheaper model would fail, use the cheaper model.

### Escalation triggers (use a premium model)

Reach for a premium model when *any* of these are true:

- Cross-module refactor, architectural design, or API design from
  scratch.
- Subtle correctness reasoning: concurrency, lifetimes, `unsafe`,
  FFI ABI, cryptography, safety-critical control paths.
- Debugging a failure that survived one prior cheap-model attempt.
- Reviewing code on a safety-, security-, or money-critical path.
- The diff cannot be predicted in advance — i.e. there is genuine
  creative or design work to do, not just typing.

### De-escalation triggers (use a cheap model)

Use the cheapest available model when *any* of these are true:

- Searching, reading, summarising files or docs.
- Single-file mechanical edits: rename, format, lint fix, dependency
  bump, boilerplate, scaffolding from a known template.
- Generating tests for code that already works.
- Running builds, tests, linters, or other commands where the model
  only needs to report success/failure.
- Routine commits, PR descriptions, changelog entries.
- The diff is essentially predictable before generation.

### Sub-agent routing (the `task` tool)

When delegating with the `task` tool, set `model:` explicitly. Do not
let sub-agents inherit a premium default for cheap work.

| Sub-agent type    | Default model             | Override to                                     |
|-------------------|---------------------------|-------------------------------------------------|
| `explore`         | cheap                     | keep cheap (`claude-haiku-4.5` or `gpt-5-mini`) |
| `task` (run cmd)  | cheap                     | keep cheap                                      |
| `research`        | cheap for breadth         | premium only for the final synthesis            |
| `general-purpose` | match task                | cheap for mechanical work; premium for design   |
| `rubber-duck`     | premium                   | keep premium — this is where reasoning pays off |
| `code-review`     | premium on critical paths | cheap on cosmetic / mechanical diffs            |

### `/fleet` (parallel sub-agents) rules

- Fleet mode multiplies cost by the fleet width. Apply the rules
  above *per worker*, not in aggregate.
- Split a fleet job along complexity lines: route the cheap,
  parallelisable workers (file edits, test runs, doc updates) to a
  cheap model; reserve premium models for the small number of
  workers that need real reasoning.
- If every worker in a fleet would need a premium model, the work is
  probably not a good fit for fleet mode — reconsider the
  decomposition before paying N× premium.

### Session hygiene

- Keep sessions short and focused. Long premium sessions are the
  single largest source of waste because every turn re-processes the
  full history.
- Use `/compact` when the conversation grows long, and `/new` for
  unrelated work.
- Prefer `/ask` for one-off side questions so they don't extend the
  main session.

### When in doubt

Ask: *"If a cheaper model produced the wrong answer here, would I
catch it in seconds (compiler, tests, my own review) or in
weeks (production incident)?"* If the former, use the cheap model
and let the feedback loop do its job.
