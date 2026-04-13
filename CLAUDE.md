# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this
repository.

## What this repo is

Policy-as-code for the **IntegratedDynamic** GitHub org. All repo settings (branch protection,
labels, merge strategies, team permissions) are declared here and enforced by a GitHub Actions
workflow that runs [safe-settings](https://github.com/github/safe-settings) on every push to `main`.

**One invariant:** never touch GitHub settings directly in the UI — this repo always wins on the
next sync.

## Common commands

```bash
# Format all files (requires mise + npm install first)
mise run fmt

# Check formatting without writing
mise run fmt:check

# Bootstrap after clone
mise install && npm install
```

## How to make and validate a change

```bash
# 1. Branch off main
git checkout -b my-change

# 2. Edit config (see structure below)

# 3. Dry-run from your branch — mandatory before merge
gh workflow run safe-settings-sync.yml --repo IntegratedDynamic/admin --ref my-change -f nop=true

# 4. Check the dry-run output
gh run list --repo IntegratedDynamic/admin --limit 3
gh run view <run-id> --repo IntegratedDynamic/admin --log | grep "There are changes"

# 5. Open PR, merge — real sync fires on push to main
```

## Config structure and precedence

```
.github/settings.yml          # org-wide defaults (ALL repos)
.github/suborgs/*.yml         # name-pattern groups (merged on top of settings.yml)
.github/repos/<name>.yml      # single-repo overrides (highest precedence)
deployment-settings.yml       # safe-settings app config (not repo settings)
```

Precedence (highest wins): `repos/<name>.yml > suborgs/*.yml > settings.yml`

Only declare what **changes** at each level — everything else is inherited via deep merge.

### suborgs name patterns

| File                 | Matches                                                            |
| -------------------- | ------------------------------------------------------------------ |
| `backend.yml`        | `*-api`, `*-service`, `*-worker`, `*-backend`, `*-server`          |
| `frontend.yml`       | `*-app`, `*-web`, `*-ui`, `*-frontend`, `*-mobile`, `*-extension`  |
| `infrastructure.yml` | `infrastructure`, `gitops`, `terraform-*`, `*-infra`, `platform-*` |

### deployment-settings.yml

Controls the safe-settings **process** (not individual repos):

- `restrictedRepos.exclude` — repos safe-settings will never touch (currently: `admin`, `.github`)
- `configvalidators` — validate a single setting value (e.g. block admin collaborator permission)
- `overridevalidators` — validate when a suborg/repo overrides an org setting (e.g. block lowering
  `required_approving_review_count` below org baseline)

## Known safe-settings bugs and workarounds (version 2.1.17 → 2.1.19)

These are **already worked around** in this repo — do not undo them:

1. **Suborgs directory early-exit** — never add subdirectories inside `.github/suborgs/`. A dir
   causes all alphabetically-later `.yml` files to be silently skipped (`return` instead of
   `continue` bug in `getSubOrgConfigs()`). Example/template files live in `docs/suborg-examples/`
   instead.

2. **`contexts: []` not a placeholder string** — `contexts: ["placeholder"]` with a non-existent
   check name crashes NOP mode. Always use `contexts: []`.

3. **`bypass_pull_request_allowances` only in `settings.yml`** — safe-settings deep-merges arrays
   (concatenates, not replaces). If set in both `settings.yml` and a suborg file, `nbrieussel` ends
   up listed twice and the API rejects it. Set bypass **only** in `settings.yml`.

4. **probot v14 full-sync break** — fixed in 2.1.19+ via
   [PR #949](https://github.com/github/safe-settings/pull/949). The version is now running `2.1.19`
   in `.github/workflows/safe-settings-sync.yml` (`SAFE_SETTINGS_VERSION`).

## Open hygiene issues (tracked in this repo's GitHub Issues)

| #                                                         | Priority | Title                                              |
| --------------------------------------------------------- | -------- | -------------------------------------------------- |
| [#3](https://github.com/IntegratedDynamic/admin/issues/3) | P1       | Add `permissions: contents: read` to sync workflow |
| [#4](https://github.com/IntegratedDynamic/admin/issues/4) | P2       | Harden sync workflow (npm ci, ubuntu pin, timeout) |
| [#5](https://github.com/IntegratedDynamic/admin/issues/5) | P2       | Upgrade safe-settings 2.1.17 → 2.1.19              |
| [#6](https://github.com/IntegratedDynamic/admin/issues/6) | P2       | Add Renovate for automated dependency updates      |
| [#7](https://github.com/IntegratedDynamic/admin/issues/7) | P2       | Establish GitHub App private key rotation policy   |
| [#8](https://github.com/IntegratedDynamic/admin/issues/8) | P3       | SHA-pin the safe-settings checkout ref             |
| [#9](https://github.com/IntegratedDynamic/admin/issues/9) | P3       | Add PR template with dry-run checklist             |

## Deep-merge behaviour — what to watch for

Safe-settings **merges** configs recursively. Arrays are **concatenated**, not replaced. This means:

- Adding a label at suborg level adds it on top of org labels (good)
- Adding `bypass_pull_request_allowances` at suborg level duplicates users already in `settings.yml`
  (bad — see bug #3 above)
- Adding `teams:` at suborg level is additive (fine, but teams must exist in the org first)

## Workflow — how the sync runs

The GitHub Actions workflow (`.github/workflows/safe-settings-sync.yml`) checks out the official
`github/safe-settings` public repo at a pinned version tag, then runs `npm run full-sync`. It uses:

- `APP_ID` + `PRIVATE_KEY` + `GITHUB_CLIENT_ID/SECRET` + `WEBHOOK_SECRET` — GitHub App credentials
  stored as repo secrets/variables
- `DEPLOYMENT_CONFIG_FILE` pointing to `deployment-settings.yml` at repo root
- `FULL_SYNC_NOP=true` in dry-run mode (enables `LOG_LEVEL=debug` automatically)

The app code is **not forked** — it is fetched fresh from `github/safe-settings` on every run.
