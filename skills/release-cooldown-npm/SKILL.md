---
name: release-cooldown-npm
description: Configure npm release cooldown via the min-release-age setting (unit is days, integer only) to quarantine newly published packages and defend against supply-chain attacks. Use when hardening an npm-using repo, editing .npmrc, setting up CI policy, or auditing whether a project enforces a minimum package age. Requires npm v11.10.0+.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# npm Release Cooldown (`min-release-age`)

Configure npm to refuse to install packages published fewer than N days ago.

## Why

A malicious npm version (compromised maintainer token, typo-squatted dependency, supply-chain injection) is usually discovered and unpublished within a few days. A 7-day cooldown closes the window where you would ever resolve a poisoned tarball.

## Version requirement

- **npm `>= 11.10.0`.** Earlier versions silently ignore the setting — no warning, no error.
- Verify: `npm --version`. If older, upgrade npm before relying on the setting.

## Configure

### CLI (writes to the user `~/.npmrc`)

```bash
npm config set min-release-age 7
```

### Repo `.npmrc` (commit so CI inherits it)

```ini
min-release-age=7
```

### Verify

```bash
npm config get min-release-age
```

## Unit

**Days, as a bare integer.** No letters, no quotes, no suffixes.

| Input | Result |
| ----- | ------ |
| `7` | ✅ 7 days |
| `7d` | ❌ ignored / parse error |
| `"7 days"` | ❌ ignored / parse error |
| `P7D` | ❌ ignored (that's the pip v26.1 form) |
| `10080` | ⚠️ 10080 **days** — not 7 days. That's the pnpm value. Don't confuse them. |

## Override for an urgent CVE patch

Bypass per-invocation rather than removing the policy:

```bash
npm install some-pkg --min-release-age=0
```

Document the bypass in the PR. Security teams should be able to grep CI logs for `--min-release-age=0` to find policy waivers.

## What it does NOT do

- Does not retroactively quarantine entries already in `node_modules` or `package-lock.json`. The setting gates **new resolutions** only.
- Does not apply to `git:`, `file:`, `link:`, or tarball-URL dependencies — only npm-registry resolutions.
- Does not detect malicious code. It only buys time for the ecosystem to detect and yank a bad version.

## Auditing a repo

```bash
grep -RIn 'min-release-age' . --include='.npmrc' --include='*.npmrc' 2>/dev/null
```

If the file is missing or the line is absent and the repo uses npm, add `min-release-age=7` to the committed repo-level `.npmrc`.

## Sibling tools (do not confuse the syntax)

- **pnpm** uses `minimumReleaseAge` (camelCase) and the unit is **minutes** — see the `release-cooldown-pnpm` skill.
- **npx** inherits from `.npmrc` automatically and also accepts `--min-release-age=N` inline — see `release-cooldown-npx`.
