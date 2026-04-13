# admin

Policy-as-code for the IntegratedDynamic GitHub org. All repository settings (branch protection, labels, merge strategies, team permissions) are declared here and enforced automatically via [safe-settings](https://github.com/github/safe-settings).

> **One rule:** never configure repos directly in the GitHub UI. Everything goes through this repo. If GitHub and this repo disagree, the next sync will overwrite GitHub.

---

## How it works

A GitHub Actions workflow runs safe-settings on every push to `main`, on a 4-hour schedule (drift prevention), and on manual trigger. safe-settings reads the config files in this repo and reconciles every repo in the org with what's declared.

```
push to main  ──►  safe-settings-sync workflow  ──►  GitHub API  ──►  repos updated
     ▲
     │
schedule (every 4h)
manual trigger (with optional dry-run)
```

The app code itself is checked out from the official `github/safe-settings@2.1.17` public repo — there is no fork to maintain.

---

## Config structure

```
admin/
├── deployment-settings.yml        # Configures the safe-settings app itself (not your repos)
├── .github/
│   ├── settings.yml               # Org-wide defaults — applied to ALL repos
│   ├── suborgs/
│   │   ├── backend.yml            # Overrides for *-api, *-service, *-worker, *-backend, *-server
│   │   ├── frontend.yml           # Overrides for *-app, *-web, *-ui, *-frontend, *-mobile, *-extension
│   │   └── infrastructure.yml     # Overrides for infrastructure, gitops, terraform-*, *-infra, platform-*
│   ├── repos/
│   │   └── examples/              # Example per-repo override files (not applied — just templates)
│   └── workflows/
│       └── safe-settings-sync.yml # The sync workflow
└── docs/
    └── suborg-examples/           # Example suborg files (not applied — just templates)
```

### Precedence (highest wins)

```
repos/<repo-name>.yml  >  suborgs/<pattern>.yml  >  settings.yml
```

You only need to declare what **changes** at each level. Everything else is inherited.

### What each level controls

| File | Scope | Typical use |
|---|---|---|
| `settings.yml` | Every repo in the org | Labels, default branch, merge strategies, baseline branch protection |
| `suborgs/*.yml` | Repos matching name patterns | Stricter reviews for infra, strict CI for backend/frontend |
| `repos/<name>.yml` | One specific repo | Custom description, extra labels, relaxed or tightened rules |
| `deployment-settings.yml` | The safe-settings app itself | Which repos to ignore, config validators |

---

## Making changes

### The safe workflow

```bash
# 1. Create a branch
git checkout -b my-change

# 2. Edit the relevant config file
# (see "What to edit" below)

# 3. Push and open a PR
git push -u origin my-change
gh pr create

# 4. Dry-run to preview what will change across the org
gh workflow run safe-settings-sync.yml --repo IntegratedDynamic/admin --ref my-change -f nop=true

# 5. Check the dry-run output
gh run list --repo IntegratedDynamic/admin --limit 3
gh run view <run-id> --repo IntegratedDynamic/admin --log | grep "There are changes"

# 6. Merge the PR — safe-settings applies for real on push to main
```

### Dry-run (NOP mode)

The workflow can be triggered in dry-run mode at any time — it shows every diff it *would* apply without touching anything:

```bash
gh workflow run safe-settings-sync.yml --repo IntegratedDynamic/admin -f nop=true
```

NOP runs use `LOG_LEVEL=debug` automatically. Look for `There are changes for branch` lines in the output for a diff summary.

### What to edit

| I want to… | Edit this file |
|---|---|
| Change a setting for **all repos** | `.github/settings.yml` |
| Change a setting for **all backend services** | `.github/suborgs/backend.yml` |
| Add a **new category** of repos with their own rules | Create `.github/suborgs/<name>.yml` |
| Override settings for **one specific repo** | Create `.github/repos/<repo-name>.yml` |
| Prevent safe-settings from touching a repo | Add it to `deployment-settings.yml` → `restrictedRepos.exclude` |
| Add a global validation rule | Edit `deployment-settings.yml` → `configvalidators` or `overridevalidators` |

---

## Forking / adapting this for another org

### 1. Create a GitHub App

Go to your org → Settings → Developer settings → GitHub Apps → New GitHub App.

Required permissions:
- Repository: `Administration` (read/write), `Contents` (read), `Issues` (read/write), `Metadata` (read), `Pull requests` (read/write)
- Organization: `Administration` (read/write), `Members` (read/write)

Subscribe to events: `Push`, `Repository`, `Pull request`, `Member`.

Install the app on your org (all repositories).

### 2. Configure repository secrets and variables

In the admin repo → Settings → Secrets and variables:

| Name | Type | Value |
|---|---|---|
| `SAFE_SETTINGS_PRIVATE_KEY` | Secret | Private key of the GitHub App (PEM) |
| `SAFE_SETTINGS_GITHUB_CLIENT_SECRET` | Secret | OAuth client secret of the GitHub App |
| `WEBHOOK_SECRET` | Secret | A random secret used to validate webhook payloads |
| `SAFE_SETTINGS_APP_ID` | Variable | App ID (shown on the GitHub App page) |
| `SAFE_SETTINGS_GH_ORG` | Variable | Your org name (e.g. `MyOrg`) |
| `SAFE_SETTINGS_GITHUB_CLIENT_ID` | Variable | OAuth client ID of the GitHub App |

### 3. Adapt the config

- Edit `.github/settings.yml` — replace org-wide defaults with yours
- Edit `suborgs/*.yml` — adjust patterns to match your repo naming conventions
- Edit `deployment-settings.yml` — update `restrictedRepos.exclude` to list repos that manage their own settings

### 4. Push to main

The workflow triggers automatically. Check the Actions tab for the first sync run.

---

## Caveats

- **No subdirectories inside `.github/suborgs/`** — safe-settings has a bug where a directory in that folder causes all alphabetically-subsequent `.yml` files to be silently skipped. Keep the suborgs folder flat.
- **`contexts: []` not `contexts: ["some-placeholder"]`** — specifying a non-existent check context causes NOP mode to crash.
- **`bypass_pull_request_allowances` only in `settings.yml`** — if you add it to both `settings.yml` and a suborg file, safe-settings' deep merge will concatenate the arrays and produce duplicates.
- safe-settings version is pinned to `2.1.17`. Don't upgrade to `2.1.19+` — probot v14 changed log initialization in a way that breaks the full-sync entrypoint.
