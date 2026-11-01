# intendednull/niri: downstream fork

This repository is a downstream fork of [niri-wm/niri](https://github.com/niri-wm/niri). It carries
a small patch queue on an upstream release, for the `ms7c91-nixos` desktop. The `~/nixos` flake
builds the `nixos` branch with nixpkgs' own niri recipe and only the source swapped (`forks/niri/`
in that repo).

It exists because niri 26.04 keeps every client buffer it imports while the monitors are off
([#3295](https://github.com/niri-wm/niri/issues/3295)). On this machine that reached about 37 GiB of
VRAM and GTT, and once ended in a global OOM. The fix,
[#4550](https://github.com/niri-wm/niri/pull/4550), was closed unmerged under niri's LLM policy, and
no release fixes it.

## Upstream contribution policy

niri is **human-only**. Its
[LLM policy](https://github.com/niri-wm/niri/blob/main/CONTRIBUTING.md#llm-policy) on `main` (read
2026-10-31 at `a8562a11`; the policy last changed 2026-10-19) requires all pull request code,
tests, issue text, comments and reviews to be human-written.

**Agents may not write anything that goes to niri:** no code, tests, PR or issue text, comments or
reviews.

Contributing is **off by default** for this fork. A submission happens only when the operator asks
for a specific change, and the operator writes it by hand. Upstream's development practice, which
fork work follows, is in [`.claude/rules/upstream-practice.md`](.claude/rules/upstream-practice.md).

## Patch queue

**Base:** `v26.04`, the release nixpkgs builds, so `Cargo.lock` is unchanged.

Bottom to top:

1. **`fork: Document the downstream fork`**: this file, `CLAUDE.md`,
   `.claude/rules/upstream-practice.md` and `.fork/gate`. `local`.
2. **`fix(niri): Skip fallback frame callbacks without active monitors or outputs`**, Etienne
   Robert's upstream commit `bb38b7e14` (2026-10-11), which is on `main` but not in `v26.04`.
   `backport`, `Drop-When: base contains bb38b7e14`.
3. **`Clean up renderer caches from the fallback timer`**, RiversJin's `4da66421` from #4550, with
   its `egl_cleanup_*` regression tests, carried unchanged. It fixes #3295. `closed (policy)`,
   `Drop-When: an upstream release fixes niri-wm/niri#3295`.
4. **`fork: Isolate the gate and correct the practice record`**: changes to the four files from
   commit 1, made after `nixos` was published. It isolates the gate from the live session, pins its
   toolchains, and corrects the practice file and these docs. `local`.

Every commit carries trailers for the `fork` tool in `~/nixos`. They survive rebase and
`format-patch`:

```
Upstream: <url>              # one or more: PR, issue, commit
Upstream-Status: <state>
Drop-When: <condition>       # optional, free text
```

| `Upstream-Status` | meaning | droppable when |
|---|---|---|
| `backport` | on upstream's default branch, not in our base | the base contains it (rebase then skips it) |
| `submitted` | an open upstream PR or issue carries it (anyone's) | the PR merges (it becomes `backport`), or the issue closes as completed |
| `closed (<reason>)` | the upstream PR closed unmerged: `policy`, `rejected`, `superseded` | the linked issue closes as completed by another change; re-read the patch on every base bump |
| `unsent` | not offered upstream | informational only, since contributing is off by default |
| `local` | never goes upstream (fork docs, machine-specific) | never |

Fork-authored commits follow upstream's commit style and also carry a `Co-Authored-By: Claude`
line, which marks them as agent-written. Cherry-picks keep their original author and message, plus
the `(cherry picked from commit …)` line and our trailers.

## Branches and tags

| ref | role |
|---|---|
| `nixos` (default branch) | **Integration branch**: an upstream base plus a linear patch queue. It is what `~/nixos` builds. It is rewritten (a rebase onto a new base, or a queue edit on the same base) only through the guarded path below. |
| `main` | Mirror of upstream `main`. Never committed to. Sync it with `gh repo sync intendednull/niri -b main`. |
| `patch/<slug>` | Topic branch for one change, cut from `nixos` and fast-forwarded onto it. |
| `submit/<slug>` | **Operator-requested only.** A branch shaped for upstream submission. Agents create none for niri, which is human-only. |
| `pin/<YYYY-MM-DD>-<short8>` (tags) | Created by `fork pin` for every pinned rev. Never moved or deleted. They keep every old pin fetchable after `nixos` is rewritten. |

## Guards

- **The `upstream` push URL is `no_push`.** It has no colon or slash, so git treats it as a local
  path, finds nothing there, and fails without touching the network. Don't "fix" it.
- **`gh repo set-default intendednull/niri`** is set in this clone. Without it, `gh` in a
  non-interactive shell picks the `upstream` remote, and a routine `gh pr create` would post to
  niri-wm/niri under the operator's account. The push URL only guards `git push`, so every `gh`
  call also names its repo: `gh … -R intendednull/niri`.
- **Clone config:** `rerere.enabled` and `rerere.autoupdate` are on, `remote.upstream.tagOpt` is
  `--tags`, and `upstream/HEAD` is set (`git remote set-head upstream --auto`). Tools use
  `upstream/HEAD`, never a hard-coded `main`.
- **Push permission:** sessions may push `patch/*` branches, `nixos` (a fast-forward, or the
  pinned-lease push after a rewrite) and `pin/*` tags to `origin` without asking. Nothing is ever
  pushed to an upstream.

## Landing a patch

1. Work on `patch/<slug>`, cut from `nixos`, following the upstream practice, with the trailers.
2. Run `.fork/gate`.
3. Fast-forward `nixos` onto it and push: `git push origin nixos`.
4. Pin and deploy from `~/nixos` (see Deploying).

## Rewriting `nixos` (the guarded path)

This is for both a rebase onto a new base and a queue edit on the same base (drop, replace,
reorder, reword). `fork rebase niri <new-base>` and `fork rebase niri --edit` run it.

1. Start from a clean checkout on `nixos`. After `git fetch origin`, `nixos` must equal
   `origin/nixos`, so commits pushed from elsewhere aren't discarded.
2. Save the pre-rewrite ref: `git branch nixos-pre-<label> nixos`. Never overwrite an existing one.
3. Rebase:
   - When the old base is an ancestor of the new one, use plain `git rebase <new-base> nixos`. It
     skips every commit whose patch-id is already upstream. Release tags are normally on `main`,
     but a point release can be cut on a branch (`v0.1.10.1` is on `niri-0-1-10`), so always check
     with `git merge-base --is-ancestor <old-base> <new-base>`.
   - Don't use `--onto <new-base> <old-base>` there. It compares against the old base, so it
     conflicts on a backport that upstream later edited instead of dropping it.
   - Otherwise, use `--onto` with a todo that drops every commit
     `git cherry <new-base> nixos <old-base>` marks `-`.
   - A queue edit is `git rebase -i <base>`.
   - Preview the drops with `git cherry <new-base> nixos`. rerere replays recorded conflict
     resolutions.
4. Review: `git range-diff <old-base>..nixos-pre-<label> <new-base>..nixos`, then run the full
   `.fork/gate`. On a base bump, re-read the upstream practice too.
5. Push with a lease pinned to the saved ref:
   `git push --force-with-lease=nixos:nixos-pre-<label> origin nixos`. A bare `--force-with-lease`
   is unsafe, because `fork status` refreshes `origin/nixos`. Adding `--force-if-includes` doesn't
   help, since git ignores it when the lease names an explicit expected value, so the pinned lease
   is the only guard.
6. Pin it with `fork pin niri`.

## The gate

`.fork/gate` runs upstream's CI checks on this host against the working tree. Its header says
which dev environment each step uses and why. **Every pin runs the full gate.** When a third-party
patch carried unchanged fails a step, the failure is recorded under "Where the fork deviates" in
the practice file, and the pin records it as an accepted gate failure. The patch isn't edited to
fix it.

## Deploying

Deploying is a `~/nixos` change. The runbook is `docs/forks.md` in that repo. The tool runs from a
`~/nixos` worktree as `nix run .#fork -- <command>` (`status`, `gate`, `rebase`, `pin`,
`practice`). `fork pin` writes `forks/niri/fork.json`, and the human reviews the code delta before
it is applied.
