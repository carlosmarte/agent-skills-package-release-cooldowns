---
name: release-cooldown-cargo
description: Apply a release cooldown to a Cargo/Rust project. Stable Cargo has NO native cooldown setting. A nightly-unstable feature exists (`cargo generate-lockfile -Zunstable-options --publish-time <time>`, tracking rust-lang/cargo#16271) but is not yet on stable. The recommended workaround is bot-side gating via Renovate (matchManagers cargo + minimumReleaseAge) or Dependabot (cooldown). Use when hardening a crates.io-consuming Rust project, choosing between Cargo and a bot-driven update flow, or auditing why a Rust project has no install-time quarantine.
allowed-tools: Bash,Read,Write,Edit,Grep
---

# Cargo Release Cooldown (nightly-unstable + bot-side workaround)

Stable Cargo, as of 2026-05, has **no native release-cooldown flag** — no `--exclude-newer`, no `--uploaded-prior-to`, no `[tool.cargo]` setting. A nightly-only unstable feature has landed and is making its way toward stabilization, but cannot be relied on in stable builds yet.

The actionable answer is the same shape as the [`release-cooldown-poetry`](../release-cooldown-poetry/SKILL.md) skill: gate version bumps at the **dependency-bot** layer.

## Why this matters

`cargo add`, `cargo update`, and `cargo build` will all happily resolve and lock a malicious version that hit crates.io minutes ago. Defense has to shift earlier in the workflow — prevent the version bump from ever being proposed.

## Option 1 — Nightly Cargo (`-Z lockfile-publish-time`)

Tracking issue: [rust-lang/cargo#16271](https://github.com/rust-lang/cargo/issues/16271).

```bash
# Requires a nightly toolchain
rustup install nightly
cargo +nightly generate-lockfile \
  -Zunstable-options \
  --publish-time 2026-05-05T00:00:00Z
```

- Unit: RFC 3339 timestamp.
- Behavior: the resolver refuses any package published after the cutoff. `Cargo.lock` is regenerated against the older view.
- Caveats:
  - Nightly only — not usable on stable. Tying production lock generation to nightly is itself a supply-chain risk.
  - The flag set only applies to `generate-lockfile`. Downstream `cargo build` consumes the lock as-is.
  - May be renamed / removed before stabilization.

Treat this as forward-looking: track the tracking issue, but do not depend on it for stable workflows. If/when it stabilizes, this skill should be updated to lead with the stable form.

## Option 2 — Renovate (recommended on stable)

`renovate.json` / `.github/renovate.json`:

```jsonc
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchManagers": ["cargo"],
      "minimumReleaseAge": "7 days"
    }
  ]
}
```

- Unit: human-readable duration string (`"7 days"`, `"2 weeks"`, `"1 month"`).
- Scope: every PR Renovate would open against `Cargo.toml` / `Cargo.lock`.
- Override: per-package rule with a narrower `matchPackageNames` and no `minimumReleaseAge` for an urgent CVE patch. Document the reason in the JSON.

## Option 3 — Dependabot

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "daily"
    cooldown:
      default-days: 7
      semver-major-days: 14
```

- Unit: integer days.
- Cargo is in Dependabot's supported-SemVer ecosystem list, so the `cooldown` block applies.
- Override: lower `default-days` or move to a per-ecosystem block; document in the PR.

## What this DOES protect

- A malicious crate version published minutes ago will not appear as a Renovate/Dependabot bump PR until the cooldown window expires.
- `Cargo.lock` only moves through bot-driven PRs, which carry the gate.

## What this does NOT protect

- **Manual `cargo add some-crate` / `cargo update`** on a developer machine. The bot config never sees these — the developer resolves whatever crates.io currently serves.
- **CI runs that do `cargo update` outside the bot's PR flow.**
- **Fresh builds from a pre-existing `Cargo.lock`** that was last regenerated before the bot existed. The lock is the install-time source of truth.
- **`git`, `path`, and tarball-URL dependencies** in `Cargo.toml` — bot rules typically gate registry deps only.

If install-time gating is required for the threat model, the only stable option today is to wrap `cargo generate-lockfile` in CI with a date-check script, fail the build if the lock contains crates younger than N days (parse `Cargo.lock` for `[[package]]` blocks, cross-check against `crates.io/api/v1/crates/<name>/<version>`), and refuse to merge.

## Ecosystem-specific alternatives (orthogonal to cooldown)

Rust's supply-chain story leans on different primitives than JS/Python. None of these are cooldowns, but a defense-in-depth Rust project usually combines several:

| Tool | What it does | Where it sits |
| ---- | ------------ | ------------- |
| [`cargo-audit`](https://github.com/rustsec/rustsec) | Scans `Cargo.lock` against the RustSec advisory DB | Vulnerability scanning — layer 4 |
| [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny) | Policy enforcement (banned crates, license rules, advisories) | Layer 4 / hard-stop on policy violations |
| [`cargo-vet`](https://mozilla.github.io/cargo-vet/) | Human-attested audits of crate versions before they enter the lock | Provenance — layer 2 |
| [`cargo-crev`](https://github.com/crev-dev/cargo-crev) | Distributed peer review of crates | Provenance — layer 2 |

These are not interchangeable with a cooldown. `cargo-vet` is the closest in spirit (refuse to consume a version until a human has reviewed it), but is more expensive than a time delay. Combine `cargo-vet` for high-trust transitive deps with a 7-day Renovate cooldown for the long tail.

## Auditing a repo

```bash
# Cargo-using project?
test -f Cargo.toml && echo "uses Cargo"

# Has a bot config with cargo + cooldown?
grep -RIn 'matchManagers.*cargo\|package-ecosystem:.*cargo' \
  .github/ renovate.json .renovaterc* 2>/dev/null

# Does the bot block include minimumReleaseAge / cooldown?
grep -RIn 'minimumReleaseAge\|cooldown:' \
  .github/ renovate.json .renovaterc* 2>/dev/null
```

If the project has `Cargo.toml` and no bot config with cooldown, recommend adding one of the two configs above, and surface that **manual `cargo` commands remain unprotected**.

## What to do today

| Constraint | Recommendation |
| ---------- | -------------- |
| Stable toolchain, no nightly | Renovate `minimumReleaseAge: "7 days"` with `matchManagers: ["cargo"]` |
| GitHub-hosted, prefer GH-native tooling | Dependabot `cooldown.default-days: 7` for `package-ecosystem: cargo` |
| Strict supply-chain (regulated, enterprise) | Renovate cooldown + `cargo-vet` + `cargo-deny` advisories block + CI date-check on `Cargo.lock` |
| Willing to run nightly in CI | Add `cargo +nightly generate-lockfile -Zunstable-options --publish-time <date>` as a CI step; still keep the bot cooldown for PR ergonomics |

## Sibling skills

- [`release-cooldown-poetry`](../release-cooldown-poetry/SKILL.md) — same bot-side workaround pattern in a different ecosystem
- [`release-cooldown-uv`](../release-cooldown-uv/SKILL.md) — the kind of native install-time gating Cargo will eventually grow once `--publish-time` stabilizes
- See [`.agents/cheatsheet.md`](../../.agents/cheatsheet.md) for the cross-tool decision matrix
- See [`.agents/threat-model.md`](../../.agents/threat-model.md) for what cooldown does and does not defend against
