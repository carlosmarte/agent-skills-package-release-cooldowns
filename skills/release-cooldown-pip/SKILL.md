---
name: release-cooldown-pip
description: Configure pip release cooldown via the --uploaded-prior-to CLI flag (ISO 8601 absolute timestamp; ISO 8601 duration strings like P7D supported from pip v26.1+) to ignore PyPI versions newer than a chosen cutoff and defend against supply-chain attacks. Use when hardening a pip-using Python project, install scripts, Dockerfiles, or CI workflows.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# pip Release Cooldown (`--uploaded-prior-to`)

Configure `pip` to ignore PyPI package versions uploaded after a chosen cutoff.

## Why

Same supply-chain threat model as the other tools: a malicious PyPI upload (compromised credentials, typo-squat, dependency confusion) is typically detected and yanked within days. `--uploaded-prior-to` ensures `pip` resolves to versions that already survived a quarantine window.

## Form

`pip` uses an **absolute ISO 8601 timestamp** flag — different from `uv`'s rolling-duration default.

```bash
pip install -r requirements.txt --uploaded-prior-to="2026-05-05T00:00:00Z"
```

The flag means "ignore anything uploaded **after** `2026-05-05T00:00:00Z`". To express "7 days ago" you must compute the date yourself (or use the duration form in pip ≥ 26.1, below).

## Unit / accepted forms

| pip version | Accepted form | Example |
| ----------- | ------------- | ------- |
| All recent versions | ISO 8601 absolute timestamp (UTC) | `--uploaded-prior-to="2026-05-05T00:00:00Z"` |
| `>= 26.1` | ISO 8601 duration string | `--uploaded-prior-to="P7D"` (7 days), `P2W` (2 weeks) |

Note the duration form is **ISO 8601 duration syntax** (`P7D`, `P2W`, `P1M`), not the human-readable strings `uv` accepts.

## Where to use it

`pip` has no `.pip.conf`-level setting for `--uploaded-prior-to` — it must be passed on the command line.

### Shell script / Makefile

```makefile
COOLDOWN_DATE := $(shell date -u -v-7d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)

install:
	pip install -r requirements.txt --uploaded-prior-to="$(COOLDOWN_DATE)"
```

(The `date -v` form is BSD/macOS; `date -d` is GNU/Linux. The `||` fallback handles both.)

### Dockerfile

Prefer pinning an absolute date so the image is reproducible:

```dockerfile
RUN pip install -r requirements.txt --uploaded-prior-to="2026-05-05T00:00:00Z"
```

Rebuild and update the date weekly (renovate-bot friendly).

### CI workflow (GitHub Actions example)

```yaml
- name: Compute cooldown cutoff
  id: cutoff
  run: echo "date=$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)" >> "$GITHUB_OUTPUT"

- name: Install
  run: pip install -r requirements.txt --uploaded-prior-to="${{ steps.cutoff.outputs.date }}"
```

## Bypass for an urgent CVE patch

`pip` does not have a per-package waiver flag. The options are:

1. Drop `--uploaded-prior-to` from the specific install command that needs the fresh version, leave it on the rest, and document the bypass in the PR.
2. Move the date forward to include the patched version, accepting a wider quarantine window for that install.

## What it does NOT do

- Does not apply to `git+https://...`, local path, URL, or wheel-file install targets — only PyPI-registry resolutions.
- Does not retroactively rewrite a previously generated `pip freeze` / `requirements.txt` — it gates the next resolve only.
- Does not detect malicious code; it only buys the ecosystem time to detect and yank a poisoned version.

## Auditing a repo

```bash
grep -RIn 'uploaded-prior-to' . --include='Makefile*' --include='Dockerfile*' --include='*.sh' --include='*.yml' --include='*.yaml' 2>/dev/null
```

Cross-check against every `pip install` you find — every install command should include the flag (unless the project also uses `uv` or a wrapper that supplies it).

## Sibling skills

- **release-cooldown-uv** — different flag name (`--exclude-newer`); accepts the more convenient `"7 days"` duration form
- **release-cooldown-pipx** — wraps pip; pass through via `--pip-args="--uploaded-prior-to=..."`
- **release-cooldown-poetry** — no native support; requires bot-side workaround
