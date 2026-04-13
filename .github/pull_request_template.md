## What changed and why

## Dry-run result

- [ ] Triggered:
      `gh workflow run safe-settings-sync.yml --repo IntegratedDynamic/admin --ref $BRANCH -f nop=true`
- [ ] Output reviewed — no unexpected diffs
- [ ] Known safe-settings bugs not triggered:
  - `bypass_pull_request_allowances` not added to any suborg file
  - `contexts:` uses `[]`, not a placeholder string
  - No subdirectory added to `.github/suborgs/`
