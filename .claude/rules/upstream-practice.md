# niri upstream development practice

Read at: a8562a1126922ed4c1c94d34c1dbb26633491d05 (2026-10-30)

Read on 2026-10-31 from `upstream/HEAD` (niri-wm/niri `main`), not from the `v26.04` base. Sources:
- docs: `CONTRIBUTING.md`, `docs/wiki/Development:-*.md`, `.github/pull_request_template.md`,
  `.github/ISSUE_TEMPLATE/*.md`;
- tooling: `.github/workflows/ci.yml` and `release.yml` `run:` steps, `rustfmt.toml`,
  `clippy.toml`, `typos.toml`, `[lints.clippy]` in `Cargo.toml`. There is no git hook config (no
  pre-commit, lefthook or husky; `docs/hooks/` holds mkdocs hooks), and no AGENTS.md or CLAUDE.md;
- history: the last 300 non-merge commits on `main` (2026-03-14 to 2026-10-30).

Where they disagree, what the tools enforce wins, then the docs, then the history.

## Policy

This gates everything below. niri is **human-only**. From `CONTRIBUTING.md#llm-policy` on `main`,
last changed 2026-10-19 (36e91606, "CONTRIBUTING: clarify that the LLM policy applies to test code
too"):

> You're not allowed to use LLMs to *create*.
> All pull request code (including tests), issue descriptions, comments, public reviews, etc. must
> be human-written.
> You can use LLMs privately (not like I can check or control that), but not to write from scratch.
> You may use LLMs for *translating* your human-written text; in this case you must also provide
> the original text to reduce the chance of miscommunication.

The templates carry the same rule as a checkbox, "I did not use an LLM to write this ...". The PR
template and the bug report got it on 2026-09-20 (7256ccf6). The blank-issue template `other.md`,
with its own checkbox, came on 2026-10-01 (ed22699d), along with `blank_issues_enabled: false`.

For agents in this fork:
- Contributing upstream is **off by default** (operator standing rule). Agents draft and file
  nothing for niri: no code, tests, PR or issue text, comments or reviews, and no `submit/*` branch.
- Any upstream submission is **operator-requested only** and must be **hand-written by the
  operator**. The rest of this file exists so fork work looks like upstream's, and so the operator
  has the practice at hand. It starts no contribution.
- A fork commit with a `Co-Authored-By: Claude` line is agent-written, so it can never be offered
  to niri as it stands.

## Code style

- **rustfmt, nightly** (tooling). `rustfmt.toml` sets `imports_granularity = "Module"`,
  `group_imports = "StdExternalCrate"`, `wrap_comments = true` and `comment_width = 100`, and all
  four are nightly-only options. CI runs `cargo fmt --all -- --check` on
  `dtolnay/rust-toolchain@nightly`. CONTRIBUTING says `cargo +nightly fmt --all`, and the PR
  template `cargo +nightly fmt`.
- **Upstream's dev shell differs by base.** On `main` (since 165a6d1e, "nix: remove `rust-overlay`
  input", 2026-06-17), `nix develop` gives nixpkgs' stable rustc, cargo and clippy, with nightly
  mode from `rustfmt.override { asNightly = true; }`. The `v26.04` shell this fork's base carries
  is older: a rust-overlay nightly frozen by its `flake.lock` (1.92.0-nightly, 2025-09-15). Both
  run `cargo fmt --all` with the nightly options; `.fork/gate` checks with its own pinned nightly.
- **clippy** (tooling). CI runs `cargo clippy --all --all-targets` on stable. `clippy.toml` sets
  `ignore-interior-mutability` for `smithay::desktop::Window`, `smithay::output::Output` and
  `wayland_server::backend::ClientId`. `Cargo.toml` allows `new_without_default` and
  `collapsible_match`. CI does not pass `-D warnings`, so only clippy errors fail it.
- **MSRV** (tooling). The msrv job runs `cargo check --all-targets` on the `rust-version` from
  `Cargo.toml`: 1.87 on `main`, 1.85 at `v26.04`. Don't use newer language or std features.
- **Everything else is convention.** CONTRIBUTING's review section: look at the surrounding code
  and at similar modules (e.g. other protocol implementations), and follow their style and
  structure. Design rules are in `Development:-Design-Principles.md`, e.g. actions apply
  immediately, disabled eye-candy costs nothing, consider all edge cases.
- **Logging levels** (`Development:-Developing-niri.md`): `error!` is a recoverable bug, used where
  you'd otherwise `unwrap()`, since a compositor crash takes the session down. `warn!` is something
  bad but possible, like a config error. `info!` must not make users want logging off. `trace!` is
  compiled out of release builds.

## Comments

- rustfmt wraps comments at 100 columns (`wrap_comments`, `comment_width`).
- In `src/` (history, not enforced), `//` comments are full sentences, capitalised, ending in a
  period. They say why, or name a constraint the code can't show ("Tile positions must be rounded
  to physical pixels."). Items get `///` doc comments.
- `typos.toml` configures the `typos` spell checker, but no CI job runs it, and `main` is not clean
  against it (typos-cli 1.46.3 reports 2 findings at a8562a11). It is not a gate.
- CONTRIBUTING asks to document unobvious design decisions: why one way over another, and which
  alternatives were tried.

## Commit messages

Nothing enforces the format: there is no hook and no CI check. From the history:
- **Subject shape `area: Summary`.** 188 of 300 subjects have a lowercase `area:` prefix (199
  counting others like `CONTRIBUTING:`): a module path or component (`input:` 32, `wiki:` 24,
  `pw_utils:` 18, `config:` 10, `tty:` 9, `tests/binds:` 9, `layout/scrolling:`, `backend/winit:`,
  `ci:`, `nix:`). A subject without one is a plain capitalised sentence ("Render IME popups at full
  opacity"; 107 of the other 112 start upper-case).
  The maintainer, who wrote 195 of the 300, prefixes 106 of his.
- **Summary** is imperative and usually capitalised after the prefix (126 upper-case, 62 lower).
  No trailing period (1 of 300 has one).
- **Conventional-commit types** (`feat:`, `fix:`, `fix(niri):`) appear in 21 of 300, all from
  contributors. The maintainer doesn't use them; prefer a module path.
- **Length:** median 44 characters, 90th percentile 62, max 110.
- **Body:** 115 of 300 have one. Wrapped at 72 (both p90 and p95 of line length are 72). It says
  why, or lists behaviour changes as `-` items.
- **Trailers are rare:** `Co-authored-by` 11 of 300, `Signed-off-by` 3 (no DCO). Issue references
  are rare and in prose ("Fixes #1584").

## Branching and history

- CONTRIBUTING, "Writing pull requests":
  - small, self-contained commits, each of which builds and passes tests;
  - squash review fixes into the commits they belong to;
  - rebase onto `main`, never merge it in, and force-push that rebase separately from content
    changes.
- History: `main` is linear, with no merge commits since at least 2026-03-14. Contributors'
  commits land rebased, with the maintainer as committer (289 of 300).

## Tests

- CI `test` job (stable): `cargo test --no-run --all --exclude niri-visual-tests`, then
  `cargo test --all --exclude niri-visual-tests -- --nocapture`.
- CI `randomized-tests` job: the same with `--release`, plus `RUST_BACKTRACE=1`,
  `RUN_SLOW_TESTS=1`, `PROPTEST_CASES=200000` and `PROPTEST_MAX_{LOCAL_REJECTS,GLOBAL_REJECTS,
  SHRINK_ITERS}=200000`. The wiki gives the maintainer's pre-push run as `env RUN_SLOW_TESTS=1
  PROPTEST_CASES=200000 PROPTEST_MAX_GLOBAL_REJECTS=200000 RUST_BACKTRACE=1 cargo test --release
  --all`.
- Where tests go (wiki, CONTRIBUTING):
  - new layout operations into the `Op` enum at the bottom of `src/layout/mod.rs`, which feeds the
    randomized tests, and into the `every_op` arrays where applicable;
  - new config options into the config parsing test;
  - weird Wayland behaviour into client-server tests in `src/tests/`. Snapshots use insta
    (`cargo-insta` is in the dev shell).
- `egl_*` tests need a working EGL display: a GPU, or Mesa's software renderer as on CI. nixpkgs'
  build skips them with `--skip=::egl`.
- The LLM policy names tests explicitly: "testing things that are trivially true, or testing
  internal states of the system rather than the outcomes" is what it objects to.
- Test by hand: as a nested window first, then on a TTY (`Development:-Developing-niri.md`).
  CONTRIBUTING's Testing section asks for edge cases, different input devices, and checking for
  performance drops.

## Docs

- New config options are documented on the wiki (`docs/wiki/`) with an example and a
  `<sup>Since: next release</sup>` annotation. The release replaces it with the version, and
  `release.yml` fails if one is left.
- Wiki pages (`Development:-Documenting-niri.md`): relative links with anchors, GitHub-style
  admonitions (`> [!NOTE]`), and code blocks annotated with a language. mkdocs builds them.
- Docs commits use the `wiki:` prefix (`wiki/<page>:` for one page).

## Review

From CONTRIBUTING, "Reviewing": reviewers check edge cases and missed scenarios, fit with niri's
design and code style, unrelated changes that belong in their own PR, and wiki updates (examples,
correct Since annotation). Authors mark addressed comments resolved.

## Submitting (operator-requested only)

Agents never submit, and for niri they produce no submission material at all (see Policy). This is
what upstream expects of the operator's own hand-written submission:
- Features match niri's design direction, ideally settled first in an issue or discussion.
- One feature or bug fix per PR, with no unrelated changes. Enable "Allow edits from maintainers".
- The description says what it does, what problem it solves, how to test it and how it was tested,
  with a config example for new syntax and the design decisions explained. Complete the PR
  template checklist.
- Cut it from upstream `main`, not the fork's base. Strip the fork's trailers and gate it against
  upstream's tip.

## Gate

`.fork/gate` mirrors `ci.yml` step by step on this host. Its header records which environment each
step uses and why. In short:
- tests, randomized tests and the feature-combination builds run in nixpkgs' niri build
  environment with its own rustc, where EGL works here;
- clippy, the visual-tests build and the Fedora `cargo build --all` run in the tree's own dev
  shell (for libadwaita) with a pinned stable toolchain, rustfmt with a pinned nightly, and msrv
  with the version the tree's `ci.yml` names. The gate pins rust-overlay itself, so none of this
  depends on the base's flake inputs;
- every step runs without the live session's display, compositor IPC and session-bus variables,
  with a private `XDG_RUNTIME_DIR` and `INSTA_UPDATE=no`;
- `nix flake check` is advisory, as in CI.

The `freebsd` job, the Alpine part of `build-musl`, and the deploy-only docs jobs aren't mirrored.
Run the gate before anything lands on `nixos` and on every pin.

## Where the fork deviates and why

- **The `local` docs commits** (FORK.md, CLAUDE.md, this file, `.fork/gate`) are fork-only and
  agent-written. Fork commits carry `Upstream:`, `Upstream-Status:` and `Drop-When:` trailers and a
  `Co-Authored-By: Claude` line, which upstream doesn't use. The tools read the trailers, and the
  attribution marks the commit as never submittable.
- **The base is a release** (`v26.04`, what nixpkgs ships), not `main`. The practice is still read
  from `main`.
- **Carried commits keep their own style.** The `bb38b7e14` backport keeps its `fix(niri):`
  subject, and RiversJin's commit from #4550 its unprefixed one.
- **RiversJin's #4550 fix is carried unchanged with its `egl_cleanup_*` tests.** The PR was closed
  unmerged under the LLM policy, so upstream won't take this code or these tests as they are. They
  stay unedited so the carried code matches the PR, and the tests are the fork's regression check
  for #3295.
- **The gate isn't CI's exact environment.** CI takes the newest `@stable` and `@nightly` on every
  run; the gate's toolchains are fixed until someone bumps them:
  - tests, randomized tests, the feature-combination builds and `build-musl`: rustc 1.95.0, from
    the nixpkgs ~/nixos locks. It is the toolchain the deployed fork is built with, and this
    environment is where the EGL tests run here;
  - clippy, the visual-tests build and the Fedora build: rustc 1.99.1, pinned stable;
  - rustfmt: `nightly-2026-10-31` (rustc 1.101.0-nightly, built 2026-10-29), pinned;
  - msrv: 1.85.0, from `v26.04`'s `ci.yml`.

  Fedora's job uses Fedora's own cargo, and the gate uses the pinned stable instead.
  `build-musl`'s command runs against glibc, and the `freebsd` job is not run.
