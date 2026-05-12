# Release Cooldown — Cross-Tool Cheatsheet

The seven tools all express "ignore versions younger than N" but they disagree on the config key, the file, and the unit. Mixing them up silently disables the protection. This page is the single decision matrix.

## At-a-glance matrix

| Tool | Where | Key / flag | Unit | "7 days" literal | Per-package waiver |
| ---- | ----- | ---------- | ---- | ---------------- | ------------------ |
| npm | `.npmrc` | `min-release-age=` | days, bare int | `7` | none — use `--min-release-age=0` inline |
| npx | inherits `.npmrc`, or inline | `--min-release-age=` | days | `7` | inline `=0` |
| pnpm | `pnpm-workspace.yaml` | `minimumReleaseAge:` | **minutes** | `10080` | `minimumReleaseAgeExclude:` list |
| uv | `pyproject.toml` `[tool.uv]` | `exclude-newer =` | duration str **or** RFC 3339 | `"7 days"` | `exclude-newer-package = { ... }` |
| pip | CLI only | `--uploaded-prior-to=` | ISO 8601 timestamp (or `P7D` in ≥26.1) | `"2026-05-05T00:00:00Z"` | omit flag per-invocation |
| pipx | CLI only, via passthrough | `--pip-args="--uploaded-prior-to=..."` | inherits pip | inherits pip | omit per-invocation |
| poetry | **bot config only** | Renovate `minimumReleaseAge` / Dependabot `cooldown.default-days` | bot's choice | `"7 days"` / `7` | bot's per-package rule |

## Footguns (read these before copy-pasting)

1. **`10080` is pnpm, not npm.** pnpm's unit is minutes, so 7 days = 10080. Writing `10080` in `.npmrc` means **10080 days** to npm. Writing `7` in `pnpm-workspace.yaml` means **7 minutes** to pnpm.
2. **`"7 days"` is uv-only.** npm/pnpm parsers expect integers; only uv (and the bot configs) accept the duration string.
3. **pip needs an absolute timestamp** (until v26.1 lands `P7D` duration support). To express "7 days ago" in a Makefile, compute it:
   ```sh
   date -u -v-7d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ
   ```
4. **Poetry has no install-time protection.** A bot-side cooldown gates *PRs*, not manual `poetry add` calls or CI installs from existing locks. If you need install-time gating in a Python project, prefer uv.
5. **Lockfiles bypass cooldown.** Every tool's cooldown gates *new resolutions*, not entries already in `package-lock.json` / `pnpm-lock.yaml` / `poetry.lock` / `uv.lock`. After enabling, run a fresh resolve to flush the lock.
6. **`git:` / `file:` / `link:` / URL deps are never gated.** All seven tools' cooldowns apply to registry resolutions only.

## Auditing a polyglot repo

Run all of these and any missing config is a finding:

```bash
# JS side
grep -RIn 'min-release-age'    --include='.npmrc'              --include='*.npmrc' .
grep -RIn 'minimumReleaseAge'  --include='pnpm-workspace.yaml'                       .

# Python side
grep -RIn 'exclude-newer'      --include='pyproject.toml'      --include='uv.toml'  .
grep -RIn 'uploaded-prior-to'  --include='*.sh'  --include='Makefile*' --include='Dockerfile*' --include='*.yml' --include='*.yaml' .

# Poetry (bot-side)
grep -RIn 'minimumReleaseAge\|cooldown:' .github/ renovate.json .renovaterc* 2>/dev/null
```

If a repo uses a given tool (lockfile or manifest present) and the corresponding grep returns nothing, the protection is absent — open a PR using the matching skill in this package.

## Picking a default cooldown window

A 7-day window is the common floor across vendor reference docs. Trade-offs:

| Window | Pro | Con |
| ------ | --- | --- |
| 1–3 days | catches the obvious smash-and-yank attacks | misses slower-burning supply-chain campaigns |
| 7 days (recommended) | covers most published incident-detection windows | delays legitimate hotfixes by up to a week |
| 14–30 days | strong default for enterprise / regulated environments | requires a documented CVE-bypass process or developers will work around it |

For each ecosystem the bypass form is different — see the individual skill for the canonical override.

## CVE bypass forms (per tool)

| Tool | Cleanest waiver |
| ---- | --------------- |
| npm | `--min-release-age=0` inline, comment in the PR |
| pnpm | add package to `minimumReleaseAgeExclude` with a comment, remove after cooldown elapses |
| uv | `exclude-newer-package = { "pkg" = "<timestamp>" }` — surgical, no global change |
| pip | drop `--uploaded-prior-to` from the single install command |
| pipx | drop `--pip-args=...` from the single install command |
| poetry (via bot) | per-package rule in `renovate.json` / `dependabot.yml` |

uv's per-package form is the cleanest of the seven — the rest require a global change or a documented per-invocation override.
