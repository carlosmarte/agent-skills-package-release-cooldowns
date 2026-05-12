---
name: release-cooldown-npx
description: Enforce a minimum-release-age cooldown on npx invocations to avoid executing newly published (potentially malicious) package versions. npx inherits .npmrc settings automatically but also accepts --min-release-age=N inline for one-shot runs. Use when hardening CI shell steps, README install snippets, or dev-machine npx usage.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# npx Release Cooldown

Apply a minimum-publication-age quarantine to `npx` runs so a freshly poisoned package version cannot be executed.

## How npx inherits the policy

`npx` reads the same `.npmrc` chain as `npm`. If `min-release-age=7` is set in `~/.npmrc` or the repo `.npmrc`, `npx` will refuse to fetch and execute a package version younger than 7 days. No extra flag is required.

If you are configuring the repo policy, see the `release-cooldown-npm` skill — that is the canonical place. This skill only documents the `npx`-specific surface.

## Inline override for a single execution

When the calling shell does not own an `.npmrc` (a one-liner in a doc, a remote CI runner, a sandbox), force the policy inline:

```bash
npx --min-release-age=7 <package-name>
```

Same unit as `npm`: **bare integer days** (`7`, not `7d`, not `"7 days"`).

## When to inline vs. inherit

| Situation | Recommended form |
| --------- | ---------------- |
| Project-local `.npmrc` exists | Inherit — do nothing |
| README "try it out" install snippet | Inline `--min-release-age=7` so copy-pasters get the protection |
| CI step on a runner without a checked-out `.npmrc` | Inline |
| Bypass for urgent CVE | `--min-release-age=0` with a comment explaining why |

## Version requirement

Underlying npm must be `>= 11.10.0`. `npx` ships with npm, so verifying npm covers `npx`:

```bash
npm --version
npx --version
```

## What it does NOT do

- Does not gate already-downloaded npx cache entries — `~/.npm/_npx/` may already contain a younger version from a previous unconstrained run. Clear with `npm cache clean --force` if you need a clean slate.
- Does not protect `node` invocations that load a pre-installed package; only the `npx` resolve+fetch step is covered.

## Auditing usage

```bash
grep -RIn 'npx ' . --include='*.sh' --include='*.yml' --include='*.yaml' --include='Makefile*' --include='README*' 2>/dev/null \
  | grep -v 'min-release-age'
```

Each match without `--min-release-age=` is a candidate for hardening — especially in CI workflow YAML and README install snippets that users will copy verbatim.

## Sibling skills

- **release-cooldown-npm** — sets the repo/user `.npmrc` policy that `npx` inherits.
- **release-cooldown-pnpm** — different config name and unit (minutes), do not cross-reference.
