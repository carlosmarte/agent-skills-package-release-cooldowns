# Release Cooldown — Threat Model

What release cooldown defends against, and where it fails. Read this before recommending a cooldown to a team — it is not a security boundary, it is a delay.

## What it protects against

| Attack | How cooldown helps |
| ------ | ------------------ |
| Stolen maintainer token used to publish a malicious version | The version is usually flagged by community/ecosystem scanners within hours-to-days and unpublished. A 7-day cooldown means you never resolve to it. |
| Compromised CI of an upstream library publishing a poisoned release | Same as above — the bad release gets yanked before it can be installed. |
| Dependency confusion / typosquat that gets reported quickly | If the package gets removed within the window, the resolver never picks it. |
| Reckless / accidental publish (broken release) | You install yesterday's last-known-good version instead of today's broken one. |

## What it does NOT protect against

| Threat | Why cooldown fails |
| ------ | ------------------ |
| Long-dwell attacks (months) | Cooldown windows are days; an attacker who sits on access for weeks before publishing a clean-looking patch defeats it. |
| Already-installed compromised version | Cooldown gates **new resolutions**. If your lockfile already pins the bad version, cooldown does nothing — you need lockfile audit + CVE feeds. |
| `git:` / `file:` / `link:` / URL deps | Cooldown applies to registry resolutions only. A `git+https://...` dependency is unbounded. |
| Malicious install scripts (preinstall, postinstall, build) on a 30-day-old version | The package age is fine, the script is malicious. Cooldown is orthogonal. |
| Transitive dependency confusion | Cooldown gates direct registry lookups; the resolver still walks the full dep tree using cached metadata. |
| Stolen credentials with a slow attacker | If the attacker is patient enough to publish, wait the cooldown, and let you install — the cooldown was beaten. |

## What it does NOT do that operators expect

- **Does not detect malicious code.** It only buys time for the ecosystem to detect and yank.
- **Does not rewrite existing lockfiles.** Run a fresh resolve after enabling, or the policy has no effect on the current install.
- **Does not gate manual installs in Poetry.** The bot-side workaround only catches PR bumps; a developer typing `poetry add` is unprotected. See [release-cooldown-poetry](../skills/release-cooldown-poetry/SKILL.md).
- **Does not survive a fresh CI install from an old lock.** Once a bad version is locked, it installs regardless of the cooldown until you re-resolve.

## Where cooldown belongs in a layered defense

```
1. Source-of-truth pinning      lockfile committed, checksum-verified
2. Supply-chain provenance      Sigstore / npm provenance / PEP 740 attestations
3. RELEASE COOLDOWN (this pkg)  refuse versions younger than N days
4. Vulnerability scanning       continuous CVE feeds against the lockfile
5. Sandboxed installs           install scripts disabled / containerized CI builds
6. Runtime egress controls      block unexpected outbound from build agents
```

Cooldown is **layer 3** — a delay that buys time for layers 4 and 5 to catch the bad version before it lands in your tree. On its own it is necessary-but-not-sufficient.

## Choosing a cooldown window

The 7-day default in the per-tool skills is a balance:

- Below 3 days: many supply-chain incidents take longer to surface; the protection thins out.
- 7–14 days: covers most reported detection-to-yank windows in published incident postmortems.
- 30 days: strong for regulated environments, but requires a documented CVE-bypass process or developers will work around it.

Pick once, document the choice, and write the CVE bypass procedure at the same time — a cooldown without a documented bypass becomes a self-inflicted DoS the first time security needs to ship a patch.

## When NOT to enforce cooldown

- **First-party publish-and-consume loops** within the same org: you control the publisher, the cooldown adds latency with no threat-model benefit. Use the exclude lists (pnpm `minimumReleaseAgeExclude`, uv `exclude-newer-package`) to bypass internal scopes.
- **Bootstrap of a brand-new project**: a fresh `package.json` / `pyproject.toml` with no lock has no history to defend; you'll resolve at whatever the cooldown says is the newest acceptable version on day zero anyway. Worth setting the policy at the same time so the second install is gated.
- **CI runs against pinned lockfiles** that have already passed scanning: redundant with layer 1; harmless but adds no value.

## When cooldown is mandatory

- **Floating-version installs in CI** (`npm install` without a lock, `pip install package` without pin). These re-resolve every time and are the highest-leverage attack surface — the cooldown is the only gate.
- **Developer-machine global installs** (`pipx install`, `npm install -g`) that aren't covered by a project lockfile.
- **Renovate / Dependabot bots** that auto-merge dependency bumps without human review.
