# HANDOFF-M3

Written 2026-08-24 on the MacBook, before migrating this checkout to the Mac Studio M3 Ultra
(Orca server). Chat history does not migrate. Everything the next session needs is here.

---

## Where we were

The goal was to become a recognised contributor to the upstream Orca project at `stablyai/orca`,
working from the fork `webdevtodayjason/orca`. That meant learning the repo's contribution norms,
finding genuinely unclaimed issues, and landing small, well-tested PRs rather than large ones.
One PR is merged and three more are open awaiting maintainer CI approval.

---

## Done

All four commits are pushed. Nothing exists only on the laptop.

| Commit | Branch | Upstream PR | State |
|---|---|---|---|
| `b63292bee` | (merged upstream) | [#15213](https://github.com/stablyai/orca/pull/15213) | **MERGED** 2026-08-22 by `nwparker` |
| `2148a91fe` | `fix/liquid-syntax-highlighting` | #15213 | local commit that became the merge above |
| `b58e82a23` | `fix/document-terminal-read-screen` | [#15893](https://github.com/stablyai/orca/pull/15893) | open |
| `aa9c0f6fc4` | `fix/pi-status-dead-owner-pid` | [#16118](https://github.com/stablyai/orca/pull/16118) | open |
| `e0cb1f999b` | `fix/coalesce-scroll-intent-sync` | [#16207](https://github.com/stablyai/orca/pull/16207) | open |

What each one is:

- **#15213 (merged)** — `.liquid` files rendered as uncolored plaintext. Monaco already bundles the
  Liquid grammar; `EXT_TO_LANGUAGE` in `src/renderer/src/lib/language-detect.ts` just had no
  `.liquid` entry, so `detectLanguage()` fell through to `plaintext`. One map entry plus a
  regression test. Fixes upstream issue #15128 (P0).
- **#15893** — documented `terminal read --screen` in `skill-guides/orca-cli.md`, and realigned two
  usage lines in `src/cli/help.ts` that had drifted from `src/cli/specs/core.ts`
  (`terminal read` missing `--screen`, `terminal split` missing `--command <text>`).
  Fixes upstream issue #15856.
- **#16118** — the pi/omp/prime status extension suppressed reporting forever when it inherited a
  dead owner PID. `src/main/pi/agent-status-handler-source.ts` now emits an `isOwnerAlive` probe;
  a live foreign owner still suppresses. Fixes upstream issue #16109.
- **#16207** — `syncTerminalScrollIntentSoon` scheduled four callbacks per call with no dedup, and
  the wheel listener calls it per wheel event. Now coalesced per terminal, settle debounced.
  Linked as `Refs #16137`, deliberately **not** `Fixes` (see Landmines).

Also posted: an intro in the Orca Discord `#introductions`, and PR #15213's URL in `#prs`.

---

## In flight

**Nothing is half-written.** There is no partial edit, no stash, no uncommitted work. The three
open PRs are complete and pushed; they are waiting on other people, not on us.

Next concrete step: when a maintainer approves the CI runs, check the results and respond to any
review. If a PR merges, post its **bare URL** in the Orca Discord `#prs` channel.

---

## Blocked

- **All three open PRs are blocked on a maintainer approving the GitHub Actions workflow run.**
  Fork PRs from a first-time contributor do not run CI until someone with write access approves.
  Only `track-community-pr` has run on each. This is not actionable from our side.
  For reference, #15213 sat in this state for five days before `nwparker` merged it, without us
  chasing it. Do not ping about it; it resolves on its own.
- **#16207 may need triage before it can merge.** It is scoped as `Refs`, not `Fixes` (below).

---

## Landmines

Read this section before running anything.

1. **The test suite is NOT green on `main`.** About 27 files under `src/main/`
   (`claude-accounts`, `skills`, `updater`, `ssh`) fail on a pristine `upstream/main` checkout,
   roughly 238 tests. Separately `src/renderer/src/components/terminal-pane/terminal-ime-xterm-consumed-key-commit.test.ts`
   fails on main too. **The bar is "no NEW failures", not "all green."** Before blaming your own
   change, check out pristine `upstream/main` and re-run the same suites to compare. Verified twice.

2. **Always pass the config to vitest:** `npx vitest run --config config/vitest.config.ts <paths>`.
   Without `--config`, renderer component tests fail en masse with bogus module-resolution errors
   (558 fake failures in one observed run). `pnpm test` sets this itself; ad-hoc `npx vitest` does not.

3. **Never capture an exit code through a pipe.** `pnpm test 2>&1 | tail -40` returns *tail's*
   status (0) even when the suite failed, and truncates the log so you cannot see what broke.
   Redirect to a file and read `$?` directly. This produced a false "all tests pass" report once.

4. **This file is a root-level entry and CI rejects new ones.**
   `.github/scripts/check-root-directory-entries.mjs` compares root entries between base and head
   and blocks anything new. `HANDOFF-M3.md` therefore lives **only** on the
   `webdevtodayjason/Orca-Dev` branch, which is never PR'd upstream. **Do not merge or cherry-pick
   this file onto any `fix/*` branch** — it would both fail CI and pollute an open PR.

5. **Never disable `max-lines`.** AGENTS.md forbids it outright, and CI runs a ratchet. Limits are
   `.ts` 300, `.tsx` 400, `.mjs` 600, test files 800 (blank lines and comments skipped). When a test
   file overflows, extract the harness into a `*-test-harness.ts` module — that is the established
   pattern (`src/shared/agent-hook-listener-test-harness.ts`, `src/relay/pty-handler-test-harness.ts`,
   and now `src/main/pi/agent-status-extension-test-harness.ts`).

6. **In vitest, fake `clearTimeout` alongside `setTimeout`.**
   `vi.useFakeTimers({ toFake: ['setTimeout'] })` leaves the real `clearTimeout`, which cannot
   cancel a fake timer — a working debounce then looks broken. Use
   `toFake: ['setTimeout', 'clearTimeout']`. This cost an hour.

7. **Editing a bundled skill guide requires regenerating two artifacts.** After changing anything in
   `skill-guides/`, run `pnpm run generate:bundled-skill-guides` and
   `pnpm run generate:skill-bundle-manifest`. CI runs the `verify:` counterparts and fails otherwise.

8. **Check for existing PRs before starting any issue.** This repo moves extremely fast (issue
   numbers went 15137 → 16170 in about six days) and every `good first issue` already had 1-6 open
   PRs stacked on it. Two issues that were unclaimed one day were assigned with six competing PRs
   two days later. Use:
   `gh api "repos/stablyai/orca/issues/<n>/timeline?per_page=60" --jq '.[] | select(.event=="cross-referenced")'`
   and check `.assignee`.

9. **#16207 is scoped `Refs #16137`, not `Fixes`, on purpose.** That issue also reports that TUI
   Scroll Speed has no effect on trackpads. That is *intended* behavior, not a bug:
   `resolveTrackpadPixelWheelReportCount` in
   `src/renderer/src/lib/pane-manager/pane-terminal-tui-wheel-reports.ts` returns before the
   multiplier is applied, and its comment says trackpad pixel streams are meant to map 1:1 to
   physical distance. Do not "fix" that half without a maintainer asking for it.

10. **Orca's own application state is not in this repo and git will not migrate it.**
    `~/Library/Application Support/orca` (~22M: `terminal-history/`, `orchestration.db`, `profiles/`,
    paired-device keys) and `~/.orca` (~24K). If the goal is to reproduce the working Orca *app*
    environment on the M3, that data has to be copied separately or re-paired.

11. **`.orca/` in the repo root is untracked and deliberately not committed.** It is Orca's local
    workspace scratch (`.orca/drops/` holds ~3.2M of screenshots dropped into the session). It is
    local scratch state, not source. It is **not** gitignored, so it will keep showing as untracked.

---

## Running on the M3

### Env files required

**None.** There is no `.env` at the repo root and none anywhere in the tree outside `node_modules`.
This project needs no secrets to build, test, or run. Nothing to copy across.

### Services expected

**None.** No database, no redis, no docker-compose, no tunnels, no ports to reserve. This is an
Electron desktop application, not a client/server stack. `orca serve` exists as a headless runtime
mode but is not required for development.

### Toolchain

- **Node 24** — `engines: { node: "24" }`. Local machine ran v24.11.1. CI runs 24 and 26.
  A different major will make pnpm print an engine warning on every command.
- **pnpm 10.24.0** — pinned via `packageManager` in `package.json`. Use corepack or match exactly.
- `.npmrc` sets `shamefully-hoist=true` and `minimum-release-age=4320`. The latter refuses packages
  published in the last three days, so a fresh install can resolve differently than expected.

### Commands that actually work today

```bash
pnpm install                 # first
pnpm dev                     # ensure:electron-runtime + electron-vite dev
pnpm build                   # build:desktop + build:native
pnpm test                    # ensure native runtime, then vitest with the right config
pnpm lint                    # oxlint + ~11 audit/verify gates
pnpm typecheck               # parallel tsc projects
```

Verified passing on the laptop as of this handoff: `pnpm lint` (exit 0), `pnpm typecheck` (exit 0),
`pnpm build` (exit 0). `pnpm test` passes except the pre-existing failures in Landmine 1.

The repo's own `orca.yaml` setup hook is:
```
node config/scripts/run-internal-dev-setup.mjs
pnpm install
```

### Native modules

`native/` contains `computer-use-macos`, `keyboard-layout-macos`, `notification-status-macos`
(plus linux/windows variants that are not built here). `pnpm build` runs `build:native`. The M3 is
macOS arm64 like this laptop, so no cross-compilation concern — but a first build there will
compile these from source and needs Xcode command line tools.

### Pinned to this laptop

- **GitHub CLI auth.** `gh` is authenticated as `webdevtodayjason` via
  `/Users/sem/.config/gh/hosts.yml`, token scopes `gist, read:org, repo, workflow`. **This does not
  migrate.** Run `gh auth login` on the M3 before any `gh` command, or every PR/issue lookup fails.
- **Absolute paths.** Two worktrees live at `/Users/sem/orca/workspaces/orca/Orca-Dev` (primary,
  this one) and `/Users/sem/code/orca` (on `main`). Worktree links are recorded by absolute path,
  so moving the checkout without re-creating them will break `git worktree list`.
- **Git remotes** (these do migrate with the clone, listed for confirmation):
  `origin` = `https://github.com/webdevtodayjason/orca.git`,
  `upstream` = `https://github.com/stablyai/orca.git`.
- No local certs, no `/etc/hosts` entries, no licence files, no keychain items beyond the `gh` token.

---

## Processes running right now that die when this laptop sleeps

**None.** Verified at handoff time:

- No dev server (`pnpm dev` is not running).
- No watchers, no background test or build processes (`ps` shows no vitest / pnpm / electron-vite / esbuild).
- No background agents or sub-workers (`ListAgents` reports none; none were ever spawned).
- No tunnels.
- The PR-status watcher loop was explicitly stopped and has no pending wakeups.

Nothing needs to be shut down before sleeping, and nothing will be lost by sleeping.
