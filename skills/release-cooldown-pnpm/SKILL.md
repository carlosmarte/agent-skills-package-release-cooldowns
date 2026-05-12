---
name: release-cooldown-pnpm
description: Configure pnpm release cooldown via the minimumReleaseAge setting (camelCase; unit is minutes, not days) in pnpm-workspace.yaml to quarantine newly published packages and defend against supply-chain attacks. Use when hardening a pnpm-using repo, editing pnpm-workspace.yaml, or auditing whether a project enforces a minimum package age. Supports an internal-package exclude list. Requires pnpm v10.16.0+.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# pnpm Release Cooldown (`minimumReleaseAge`)

Configure pnpm to refuse to install packages whose latest publish is younger than N minutes.

## Why

Same threat model as the npm cooldown: malicious versions (compromised maintainer tokens, supply-chain injection) are typically discovered and unpublished within days. A 7-day quarantine prevents pnpm from ever resolving a poisoned tarball during that window.

## Version requirement

- **pnpm `>= 10.16.0`.** Earlier versions silently ignore the setting.
- Verify: `pnpm --version`.

## Configure

### `pnpm-workspace.yaml` (the canonical location)

```yaml
minimumReleaseAge: 10080
minimumReleaseAgeExclude:
  - "@my-company/internal-package"
  - "@my-company/*"
```

`pnpm-workspace.yaml` is read even when the repo is not a workspace. If the file does not exist, create it at the repo root.

### Verify

```bash
pnpm config get minimumReleaseAge
```

## Unit

**Minutes, as an integer.** Not days, not seconds, not a duration string.

| Goal | Value |
| ---- | ----- |
| 1 day | `1440` |
| 7 days | `10080` |
| 14 days | `20160` |
| 30 days | `43200` |

⚠️ A common bug: writing `7` here means **7 minutes**, not 7 days. The npm/pnpm units are deliberately different — do not copy values between them.

## Exclude trusted internal packages

`minimumReleaseAgeExclude` accepts package names and `@scope/*` globs. Internal monorepo packages published to a private registry should usually be excluded so a `1.2.3 → 1.2.4` bump from your own team is not blocked by the cooldown.

```yaml
minimumReleaseAgeExclude:
  - "@acme/*"          # any internal scope
  - "@acme-tools/cli"  # specific package
```

Do **not** add third-party packages to the exclude list to "work around" the cooldown. Use a per-install override (below) and document why.

## Override for an urgent CVE patch

There is no documented `--minimum-release-age=0` CLI flag in pnpm 10.16. The supported escape hatches are:

1. Temporarily lower `minimumReleaseAge` to `0` in `pnpm-workspace.yaml`, install, then restore the value in the same PR.
2. Or add the specific package to `minimumReleaseAgeExclude` with a comment explaining the CVE, and remove it once the registry-wide cooldown window has passed.

Either way, the change should be visible in `git diff` — that is the audit trail.

## What it does NOT do

- Does not gate `git:` / `file:` / `link:` dependencies — only npm-registry resolutions.
- Does not retroactively quarantine entries already in `pnpm-lock.yaml`.
- Does not detect malicious code; it only buys the ecosystem time to detect and yank a poisoned version.

## Auditing a repo

```bash
grep -RIn 'minimumReleaseAge' . --include='pnpm-workspace.yaml' --include='.npmrc' 2>/dev/null
```

If absent and the repo uses pnpm (presence of `pnpm-lock.yaml` or `packageManager: pnpm@*` in `package.json`), recommend adding the block above to `pnpm-workspace.yaml`.

## Do not confuse with the npm syntax

| Tool | Key | Unit |
| ---- | --- | ---- |
| npm | `min-release-age` (kebab-case, in `.npmrc`) | days |
| pnpm | `minimumReleaseAge` (camelCase, in `pnpm-workspace.yaml`) | minutes |

A repo that uses both for compatibility must set both, not just one.
