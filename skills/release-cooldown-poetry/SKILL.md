---
name: release-cooldown-poetry
description: Apply a release cooldown to a Poetry-based Python project. Poetry has NO native CLI flag or pyproject.toml setting for minimum release age — protection must come from dependency-update bots (Dependabot, Renovate) configured with a cooldown / minimumReleaseAge so version-bump PRs are gated before they reach pyproject.toml. Use when hardening a Poetry repo, choosing between Poetry and uv on a new project, or auditing why an existing Poetry project has no install-time quarantine.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# Poetry Release Cooldown (bot-side workaround)

Poetry, as of 2026-05, has **no native release-cooldown support** — no CLI flag, no `[tool.poetry]` setting, no plugin in the stable distribution. There is no equivalent of `--exclude-newer` (uv) or `--uploaded-prior-to` (pip).

The actionable workaround is to gate version bumps at the **dependency-bot** layer rather than the install layer.

## Why this matters

`poetry add`, `poetry update`, and `poetry lock` will all happily resolve and lock a malicious version that was uploaded to PyPI minutes ago. The line of defense has to shift earlier in the workflow: prevent the version bump from ever being proposed.

That puts the cooldown in the bot's configuration file, not in the project's `pyproject.toml`.

## Option 1 — Renovate

`renovate.json` / `.github/renovate.json` (or the dashboard).

```jsonc
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchManagers": ["poetry"],
      "minimumReleaseAge": "7 days"
    }
  ]
}
```

- Unit: human-readable duration string (`"7 days"`, `"2 weeks"`, `"24 hours"`).
- Scope: applies to every PR Renovate would open against `pyproject.toml` / `poetry.lock`.
- Override: drop `minimumReleaseAge` on a per-package rule (`matchPackageNames: ["urgent-cve-pkg"]`) for an urgent CVE patch, with a comment in the JSON.

## Option 2 — Dependabot

`.github/dependabot.yml`.

```yaml
version: 2
updates:
  - package-ecosystem: "pip"   # also matches Poetry projects
    directory: "/"
    schedule:
      interval: "daily"
    cooldown:
      default-days: 7
      # optional per-package waivers
      semver-major-days: 14
```

- Unit: integer days.
- Scope: gates every Dependabot bump PR against `pyproject.toml`.
- Override: lower `default-days` or move to `version-update` exclusions for a specific package; document the change in the PR.

## What this DOES protect

- A malicious `1.2.4` published minutes ago will not appear as a version-bump PR until the cooldown window expires.
- The lockfile (`poetry.lock`) is only bumped through bot-driven PRs, which carry the gate.

## What this does NOT protect

- **Manual `poetry add new-pkg` / `poetry update`** on a developer machine. The cooldown lives in the bot's config, not in Poetry. A developer typing `poetry add some-pkg` will resolve to whatever is currently on PyPI.
- **Fresh installs from an old lockfile** that were locked before the bot existed. The lockfile is the install-time source of truth.
- **CI steps that call `poetry add` / `poetry update`** outside the bot's PR flow.

If the threat model requires gating manual installs and CI installs (not just bot PRs), the project should migrate to `uv` (`exclude-newer`) or layer `pip --uploaded-prior-to=...` underneath Poetry where possible.

## Auditing a repo

```bash
# Poetry-using project?
test -f pyproject.toml && grep -q '\[tool\.poetry\]' pyproject.toml && echo "uses Poetry"

# Has a bot config with cooldown?
grep -RIn 'minimumReleaseAge\|cooldown:' .github/ renovate.json .renovaterc* 2>/dev/null
```

If the project is Poetry-managed and has no bot-side cooldown, recommend adding one of the two configs above, and surface in the audit report that **manual installs remain unprotected**.

## Migration path: Poetry → uv

If the project is willing to move, `uv` reads `pyproject.toml` (including Poetry-style `[tool.poetry.dependencies]`) and provides native `exclude-newer` support at the install/lock layer. The `release-cooldown-uv` skill covers the config.

A common compromise: keep Poetry as the publish/build tool, but layer `uv pip install --exclude-newer=...` for CI/Docker install steps that don't need Poetry's dependency-group semantics.

## Sibling skills

- **release-cooldown-uv** — the recommended replacement when install-time gating is required
- **release-cooldown-pip** — usable as a per-command supplement underneath Poetry-managed projects
- **release-cooldown-pipx** — Poetry is itself often installed via `pipx`; quarantine the installer too
