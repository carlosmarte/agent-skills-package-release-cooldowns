---
name: release-cooldown-uv
description: Configure uv release cooldown via exclude-newer (in pyproject.toml [tool.uv] or uv.toml, or via the uv lock --exclude-newer CLI flag) to quarantine newly published PyPI packages and defend against supply-chain attacks. Accepts human-readable duration strings ("7 days", "2 weeks") or absolute RFC 3339 timestamps. Use when hardening a uv-using Python project, editing pyproject.toml, setting up CI lock policy, or per-package CVE bypass with exclude-newer-package.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# uv Release Cooldown (`exclude-newer`)

Configure `uv` to ignore PyPI package versions published more recently than a given cutoff — either a rolling duration or an absolute timestamp.

## Why

Same supply-chain threat model as the JS ecosystem: a malicious PyPI upload (compromised maintainer credentials, typo-squat, dependency confusion) is usually detected and yanked within days. `exclude-newer` ensures `uv` resolves to versions that have already survived a quarantine window.

## Configure

### `pyproject.toml` (preferred — project-scoped)

```toml
[tool.uv]
exclude-newer = "7 days"
```

### `uv.toml` (alternative — same syntax, no `[tool.uv]` table header)

```toml
exclude-newer = "7 days"
```

### CLI flag (one-shot, no file edit)

```bash
uv lock --exclude-newer="7 days"
uv pip install --exclude-newer="7 days" -r requirements.txt
```

## Unit / accepted forms

`exclude-newer` accepts **either** a human-readable duration string **or** an absolute RFC 3339 timestamp.

| Form | Example | Meaning |
| ---- | ------- | ------- |
| Duration string | `"7 days"` | Rolling: anything younger than 7 days ago |
| Duration string | `"2 weeks"` | Rolling: 14 days |
| Duration string | `"24 hours"` | Rolling: 1 day |
| RFC 3339 timestamp | `"2026-05-05T00:00:00Z"` | Fixed cutoff — locks are reproducible |

Duration strings are convenient for development; absolute timestamps are better for reproducible CI builds — pin a date and your lockfile cannot drift just because the wall clock advanced.

## Verify

```bash
uv lock --dry-run
```

The resolver will refuse versions newer than the cutoff and report the chosen version per package.

## Bypass for an urgent CVE patch

Unlike npm/pnpm, `uv` provides a **per-package** bypass — preferred over disabling the policy globally:

```toml
[tool.uv]
exclude-newer = "7 days"
exclude-newer-package = { "vulnerable-pkg" = "2026-05-12T00:00:00Z" }
```

Or via CLI:

```bash
uv lock --exclude-newer="7 days" --exclude-newer-package="vulnerable-pkg=2026-05-12T00:00:00Z"
```

This is the cleanest CVE-bypass form across all package managers — it leaves the global cooldown in place and surgically waives one package with a documented cutoff.

## What it does NOT do

- Does not gate `git+https://...`, local path, or URL dependencies — only PyPI-registry resolutions.
- Does not retroactively rewrite `uv.lock`. Existing locked entries stay locked; `exclude-newer` gates the **next** resolve.
- Does not detect malicious code; it only buys the ecosystem time to detect and yank a poisoned version.

## Auditing a repo

```bash
grep -RIn 'exclude-newer' . --include='pyproject.toml' --include='uv.toml' 2>/dev/null
```

If the file is missing the setting and the repo uses uv (presence of `uv.lock` or `[tool.uv]`), recommend adding the block above.

## Do not confuse with pip's syntax

| Tool | Key | Accepted unit |
| ---- | --- | ------------- |
| uv | `exclude-newer` (config) / `--exclude-newer` (CLI) | duration string OR RFC 3339 timestamp |
| pip | `--uploaded-prior-to` (CLI only) | ISO 8601 absolute timestamp (and ISO 8601 duration in pip ≥ 26.1) |

A repo that uses both tools must set both — they read different config locations and use different flag names.

## Sibling skills

- **release-cooldown-pip** — different flag name and unit
- **release-cooldown-pipx** — wraps pip; pass through via `--pip-args`
- **release-cooldown-poetry** — no native support; requires bot-side workaround
