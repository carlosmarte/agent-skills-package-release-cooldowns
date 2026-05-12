---
name: release-cooldown-pipx
description: Enforce a release cooldown on pipx installs by passing pip's --uploaded-prior-to flag through --pip-args. pipx has no native cooldown flag but builds isolated venvs via pip under the hood. Use when hardening pipx-based developer tool installs (e.g., black, ruff, poetry, copier) against supply-chain attacks.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# pipx Release Cooldown (via `--pip-args`)

`pipx` does not implement its own release-cooldown flag, but every `pipx install` / `pipx upgrade` call invokes `pip` inside an isolated venv. That gives a clean pass-through point: forward `pip`'s `--uploaded-prior-to=...` through `--pip-args`.

## Why

Developer tools installed with `pipx` (`black`, `ruff`, `poetry`, `copier`, `httpie`, `awscli`, …) are usually installed globally per-user — a poisoned upgrade can compromise the whole machine. Applying the same supply-chain cooldown that gates `pip` and `uv` closes that gap.

## Configure

### One-shot install

```bash
pipx install <package-name> --pip-args="--uploaded-prior-to=2026-05-05T00:00:00Z"
```

### Upgrade

```bash
pipx upgrade <package-name> --pip-args="--uploaded-prior-to=2026-05-05T00:00:00Z"
```

### Upgrade all (with shell-computed rolling cutoff)

```bash
CUTOFF=$(date -u -v-7d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)
pipx upgrade-all --pip-args="--uploaded-prior-to=$CUTOFF"
```

(The `date -v` form is BSD/macOS; `date -d` is GNU/Linux. The `||` fallback handles both.)

## Unit / accepted forms

`pipx` does not parse `--pip-args` — it forwards the string verbatim to `pip`. So the rules are pip's rules:

| pip version | Accepted form | Example |
| ----------- | ------------- | ------- |
| All recent | ISO 8601 absolute timestamp | `--uploaded-prior-to="2026-05-05T00:00:00Z"` |
| `>= 26.1` | ISO 8601 duration string | `--uploaded-prior-to="P7D"` |

See the `release-cooldown-pip` skill for the underlying flag's full semantics.

## Making it the default

`pipx` reads no global config for `--pip-args`. To make the cooldown the default behavior, wrap `pipx` in a shell alias or function:

```bash
# ~/.zshrc or ~/.bashrc
pipx() {
  local cutoff
  cutoff=$(date -u -v-7d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
        || date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)
  command pipx "$@" --pip-args="--uploaded-prior-to=$cutoff"
}
```

⚠️ Caveat: this appends `--pip-args` to **every** invocation, including subcommands that don't accept it (`list`, `runpip`, `environment`). A safer form switches on the subcommand:

```bash
pipx() {
  case "$1" in
    install|upgrade|upgrade-all|reinstall|reinstall-all|inject)
      local cutoff
      cutoff=$(date -u -v-7d +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
            || date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)
      command pipx "$@" --pip-args="--uploaded-prior-to=$cutoff"
      ;;
    *)
      command pipx "$@"
      ;;
  esac
}
```

## Bypass for an urgent CVE patch

Same pattern as pip: drop the `--pip-args=...` from the specific install, document the bypass in the script or the operator's notes.

## What it does NOT do

- Does not gate already-installed venvs under `~/.local/pipx/venvs/` — only the next install/upgrade resolution.
- Does not apply to `pipx install /path/to/wheel.whl` or `pipx install git+https://...` — only PyPI-registry resolutions.
- Does not detect malicious code; it only buys the ecosystem time to detect and yank a poisoned version.

## Auditing a repo / dotfiles

```bash
grep -RIn 'pipx install\|pipx upgrade' . 2>/dev/null \
  | grep -v 'uploaded-prior-to'
```

Every `pipx install` / `pipx upgrade` line without `--pip-args="--uploaded-prior-to=..."` is a candidate for hardening. Also check `~/.zshrc` / `~/.bashrc` for an unwrapped `pipx` alias.

## Sibling skills

- **release-cooldown-pip** — the underlying flag's semantics and accepted forms
- **release-cooldown-uv** — alternative Python install tool with native cooldown support
- **release-cooldown-poetry** — sibling tool that often gets installed via `pipx`; itself has no cooldown
