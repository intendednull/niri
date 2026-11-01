# CLAUDE.md: intendednull/niri (downstream fork)

This is a fork of niri-wm/niri, built by the `~/nixos` flake. Read these first:
- [`FORK.md`](FORK.md): why the fork exists, the patch queue, branches, guards and procedures;
- [`.claude/rules/upstream-practice.md`](.claude/rules/upstream-practice.md): niri's development
  practice and where the fork deviates (loaded automatically);
- [`.fork/gate`](.fork/gate): upstream's CI checks for this host. Its header explains the
  environments.

Rules for agents:

- **Upstream contributions are off by default.** niri is human-only (its LLM policy). Never draft
  or file code, PRs, issues, comments or reviews for niri-wm/niri, and create no `submit/*` branch.
  A submission happens only when the operator asks, and the operator writes it by hand.
- **Never push to niri-wm, and never point `gh` at it** beyond a read you really need. Every `gh`
  call names its repo: `-R intendednull/niri`. The `upstream` push URL `no_push` is deliberate; do
  not "fix" it.
- **Pushing to `origin` is pre-authorised:** `patch/*` branches, and `nixos` with the pinned-lease
  push after a rewrite. `pin/*` tags come from `fork pin`; never move or delete one.
- **Never commit to `main`.** It mirrors upstream.
- **Do patch work on `patch/<slug>` off `nixos`.** One concern per patch, in niri's style
  (`area: Summary`, rustfmt nightly), with the `Upstream:`, `Upstream-Status:` and optional
  `Drop-When:` trailers and a `Co-Authored-By: Claude` line.
- **Cherry-picks keep their original author and message**, plus the `-x` line and our trailers.
  Never edit carried third-party code; record its gate failures under "Where the fork deviates".
- **Rewrite `nixos` only through the guarded path** in `FORK.md`: plain `git rebase <new-base>`
  when the old base is an ancestor (never `--onto` then), `git range-diff`, the gate, and
  `git push --force-with-lease=nixos:nixos-pre-<label> --force-if-includes origin nixos`.
- **Run `.fork/gate` before landing on `nixos`.** Every pin runs the full gate.
- **Deploying is a `~/nixos` change** (`docs/forks.md`, `nix run .#fork`), not something done from
  this repo.
