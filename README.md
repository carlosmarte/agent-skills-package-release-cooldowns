# agent-skills-package-release-cooldowns

A package of 7 Claude Code skills covering **release cooldown** (minimum-release-age / quarantine) configuration across the major JavaScript and Python package managers.

A release cooldown refuses to install package versions younger than N days. Malicious uploads (compromised maintainer tokens, supply-chain injection, typosquat) are usually detected and yanked within days, so refusing fresh versions closes the window where a poisoned tarball could land in your tree.

The seven tools all express the same idea but each uses a different config name, file location, and unit — which is why this is seven separate skills rather than one.

## Skills

| Skill | Tool | Where |
| ----- | ---- | ----- |
| [`release-cooldown-npm`](./skills/release-cooldown-npm/SKILL.md) | npm | `.npmrc` — days |
| [`release-cooldown-npx`](./skills/release-cooldown-npx/SKILL.md) | npx | inherits `.npmrc` or inline |
| [`release-cooldown-pnpm`](./skills/release-cooldown-pnpm/SKILL.md) | pnpm | `pnpm-workspace.yaml` — minutes |
| [`release-cooldown-uv`](./skills/release-cooldown-uv/SKILL.md) | uv | `pyproject.toml` — duration string or RFC 3339 |
| [`release-cooldown-pip`](./skills/release-cooldown-pip/SKILL.md) | pip | `--uploaded-prior-to` CLI — ISO 8601 |
| [`release-cooldown-pipx`](./skills/release-cooldown-pipx/SKILL.md) | pipx | passthrough via `--pip-args` |
| [`release-cooldown-poetry`](./skills/release-cooldown-poetry/SKILL.md) | poetry | no native; Renovate / Dependabot bot config |

## LLM entry point

See [`.agent.md`](./.agent.md) for the structured index used by Claude Code and other agent tooling. Cross-cutting references:

- [`.agents/cheatsheet.md`](./.agents/cheatsheet.md) — single-page decision matrix across all 7 tools
- [`.agents/threat-model.md`](./.agents/threat-model.md) — what release cooldown protects and where it fails

## Install

### Recommended — `npx skills` (from [vercel-labs/skills](https://github.com/vercel-labs/skills))

The `skills` CLI is the open-agent-skills installer. It clones this repo into a canonical location and symlinks each skill into your agent's skills directory (`~/.claude/skills/` for Claude Code). It also works with Codex, Cursor, OpenCode, and 50+ other agents.

```bash
# Install all 7 skills globally into Claude Code (~/.claude/skills/)
npx skills add carlosmarte/agent-skills-package-release-cooldowns --all -a claude-code -g

# Or pick one
npx skills add carlosmarte/agent-skills-package-release-cooldowns \
  --skill release-cooldown-npm -a claude-code -g

# Or install all into the current project (./claude/skills/ committed with the repo)
npx skills add carlosmarte/agent-skills-package-release-cooldowns --all -a claude-code

# List what is available without installing
npx skills add carlosmarte/agent-skills-package-release-cooldowns --list
```

Useful flags:

| Flag | Effect |
| ---- | ------ |
| `-g, --global` | Install to `~/<agent>/skills/` instead of the current project |
| `-a claude-code` | Target Claude Code specifically (repeatable; `-a '*'` for all agents) |
| `-s, --skill <name>` | Install a specific skill (`-s '*'` for all) |
| `--copy` | Copy instead of symlinking — use when symlinks aren't supported |
| `-y, --yes` | Non-interactive (CI/CD friendly) |
| `--list` | Show available skills, install nothing |
| `--all` | Install every skill to every selected agent without prompts |

Update later with `npx skills update`, list installed with `npx skills list`, remove with `npx skills remove`.

### Alternative — manual symlink from a local clone

When you already have the repo checked out and want the in-repo `SKILL.md` files to be the single source of truth (no `skills` CLI dependency):

```bash
for s in skills/release-cooldown-*; do
  name=$(basename "$s")
  ln -sfn "$(pwd)/$s" "$HOME/.claude/skills/$name"
done
```

Each skill then appears as `/<skill-name>` inside Claude Code.

## Skill format

Each `skills/<name>/SKILL.md` follows the Claude Code Skill spec:

```yaml
---
name: <folder-name>            # must match the directory
description: <≤1024 chars>     # activation trigger
allowed-tools: <comma list>    # optional
argument-hint: "<example>"     # optional
disable-model-invocation: true # optional
---
```

Validation rules:
- `name` ∈ `[a-z0-9-]{1,64}`, no leading/trailing hyphens, no `--`.
- `name` equals the folder name.
- No absolute host-specific paths in any file (use `$HOME`, `$(pwd)`, or relative paths).
- Body content under 500 lines.

## License

See [LICENSE](./LICENSE).
