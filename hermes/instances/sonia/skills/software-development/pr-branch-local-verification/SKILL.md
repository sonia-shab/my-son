---
name: pr-branch-local-verification
description: "Verify a PR branch locally: worktree, tests, falsify."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [code-review, pull-requests, git-worktree, testing, falsification]
    category: software-development
    related_skills: [github, requesting-code-review, systematic-debugging]
---

# Verifying a PR branch locally

## When to Use

- Reviewing/approving a PR where CI green is not enough evidence.
- A PR claims a fix (bug, perf, validation) and you must show the tests pin it.
- You need the branch's real tests to run without disturbing the user's clone.

## Workflow

Use when a review needs more than reading the diff: run the branch's own tests in
isolation, then try to break the claimed fix. Reading CI status is necessary but
not sufficient — CI proves the branch is green, not that the tests pin the change.

## Isolation rules

1. Bring the PR head in as a **detached worktree**, not a checkout of the user's
   clone (their tree may be dirty and a running session may own it):
   `git fetch <base-repo> refs/pull/N/head:pr-N` then
   `git worktree add --detach <scratch>/wt-N pr-N`. GitHub exposes fork PR heads
   at `refs/pull/N/head` on the **base** repo, so no extra remote is needed.
2. **Never symlink `vendor/` (or `node_modules/`) into the worktree.** Composer's
   `vendor/composer/autoload_*.php` derives `$vendorDir = dirname(__DIR__)` from
   the *real* file path, so a symlink makes the autoloader load `app/`, models and
   test traits from the ORIGINAL clone — you silently test the base branch and
   get confusing reds. Use a hardlink copy instead: `cp -al <clone>/vendor <wt>/vendor`
   (fast, and paths resolve to the worktree).
3. Copy every env/config file the test runner loads. Laravel-style repos track
   `.env.testing` with *placeholder* secrets: if a connection fails with
   "password authentication failed" while the app's own `.env` credentials work,
   the runner is loading the tracked file, not yours. Also run the repo's app-key
   bootstrap script when it has one (blank `APP_KEY` in the tracked env file
   surfaces as `MissingAppKeyException` several tests in).
   **Re-run the bootstrap script after any `git checkout -- .env.testing`.** That
   hygiene step restores the tracked placeholder (empty `APP_KEY` by design, so
   no key ever leaks into git) and silently breaks the whole suite — a mass
   `MissingAppKeyException` failure is an env artefact, not a code regression.
   Check `git status --short` after regenerating: the key lands in a *tracked*
   file, so it must never be staged or committed. Repos that forbid committed
   keys ship a `scripts/check-app-key-leak.php` guard — run it before pushing.
4. Copy or link generated frontend assets the views need (`public/build/`,
   `manifest.json`) or every test that renders a layout dies with
   "Vite manifest not found".
5. Create `storage/framework/{cache/data,sessions,views}`, `storage/logs` and
   `bootstrap/cache` in the worktree; run `config:clear` + `route:clear` before
   the first run when the repo's docs say cached config/routes break the suite.
6. **Serialize runs.** If all worktrees share one test database, concurrent
   `artisan test` / `pest` processes collide. One run at a time.
7. Probe with NEW test files inside the worktree, then delete them. Never edit
   the PR's own tests to make a probe pass.

## Verification beyond "tests pass"

- **Falsify each claimed fix.** Revert the single line the PR hinges on (a
  validation rule, an ORDER BY tie-breaker, a by-reference capture) and confirm
  the new test goes RED, then restore. A test that also passes on the reverted
  code pins nothing — say so in the review instead of counting it as evidence.
- **Read the linked issue's comments, not only its body.** Maintainer "gate"
  comments routinely override the body's plan (e.g. rejecting a busy-503 lock in
  favour of stale-while-revalidate). Review the code against the *comment*
  decision, and flag body/plan items that are now stale rather than treating
  them as missing work.
- **Check data-backed claims against the real database** (column names and
  non-conforming counts, not the PR's prose — bodies misname columns), and check
  every caller of a changed endpoint for values the new validation would now
  reject (a new numeric regex breaks any widget passing a composite id).
- **For cache/async mechanics, verify the framework source**, in the version the
  lock file pins — which path is lock-guarded, where a lock is released, whether
  failures are swallowed is exactly what such PRs get wrong and what memory gets
  wrong. A small probe test that prints state after the request beats reasoning
  about `defer()`: served-stale + upstream call count + post-request cache value
  together prove the background refresh actually ran.
- Re-check CI on the exact head SHA (`gh pr view N --json headRefOid`) rather
  than trusting an earlier read; blocking jobs can still be running.
- **Separate local-toolchain noise from branch regressions.** A hardlinked
  `vendor/` can drift from `composer.lock` (e.g. installed laravel/framework
  v13.38.0 vs a lock pinning v13.35.0), which makes local PHPStan report
  errors CI never sees — typically "Ignored error pattern ... was not matched"
  in files the branch never touched. Confirm by running the same tool on the
  base branch: identical output there means the branch is clean and the local
  environment is stale, not that the PR broke something. Say this explicitly
  in the review instead of reporting the branch as failing.

## Hygiene

- Record `git status --short` before and after: the user's clone must come back
  unchanged (no stray branches, no modified tracked files).
- Tear down: `git worktree remove --force`, `git branch -D pr-N`,
  `git worktree prune`, delete the scratch dir.
- Adding a review is `gh pr review N --approve --body-file <file>`; when the user
  says "don't merge", never touch `gh pr merge` and re-read `state,mergedAt`
  afterwards to confirm nothing merged.
- Review bodies: what was read, what was *executed* (files + counts), the
  falsification result, the CI state with SHA — then non-blocking notes.
  Distinguish "this is broken" from "the title claims more than the code does".
