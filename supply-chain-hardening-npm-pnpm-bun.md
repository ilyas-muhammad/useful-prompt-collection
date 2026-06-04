# Machine-wide supply-chain hardening for npm, pnpm, and Bun

Apply two protections to all three package managers (user-level, not per-project):

1. Disable/block dependency lifecycle scripts (postinstall attack vector — Shai-Hulud, qix/chalk compromises)
2. Minimum release age cooldown of 7 days (malicious releases are typically yanked within hours/days)

> **Units differ per manager:** npm = DAYS, pnpm = MINUTES, Bun = SECONDS.

---

## 1. npm (requires npm >= 11.10)

Upgrade if older:

```bash
npm install -g npm@latest
```

Apply:

```bash
npm config set ignore-scripts true
npm config set min-release-age 7
```

Verify:

```bash
npm config ls -l | grep -E "ignore-scripts|min-release-age"
# ignore-scripts = true
# min-release-age resolves to a `before` date (7 days ago from now)
```

Writes to `~/.npmrc`.

### Caveats

- `ignore-scripts` also disables the **project's own** npm lifecycle hooks (pre/postbuild, prepare, etc.) — run those scripts explicitly where needed.
- Native deps (sharp, esbuild, lightningcss) still work — they ship prebuilt binaries via `optionalDependencies`. If a package breaks: `npm rebuild <pkg>` runs only that package's scripts.
- Urgent fresh install: `npm install <pkg> --min-release-age=0`
- If npm was bundled with Homebrew node, `brew upgrade node` may downgrade npm — re-upgrade after.

---

## 2. pnpm (requires >= 10.16)

### Lifecycle scripts

Already blocked by default since pnpm 10.0.0 — nothing to do.

Per-project allowlist when needed: `pnpm.onlyBuiltDependencies` in `package.json` (v10) or `allowBuilds` map (v11). Never set `dangerouslyAllowAllBuilds`.

### Cooldown (7 days = 10080 minutes)

```bash
pnpm config set --global minimumReleaseAge 10080
```

Verify:

```bash
pnpm config get minimumReleaseAge
# 10080
```

Writes `minimum-release-age=10080` to the global rc (macOS: `~/Library/Preferences/pnpm/rc`, Linux: `~/.config/pnpm/rc`).

### Escape hatches

- `minimumReleaseAgeExclude` list (supports globs like `@myorg/*`).
- pnpm 11 enables 1440-min cooldown by default + adds `strictDepBuilds`, `trustPolicy: no-downgrade`, `blockExoticSubdeps` — worth upgrading when convenient.

---

## 3. Bun (requires >= 1.3.8 — older 1.3.x had a bug ignoring global bunfig)

### Lifecycle scripts

Already blocked by default (built-in allowlist of ~vetted packages) — nothing to do.

Per-project trust when needed: `bun pm trust <pkg>` → adds to `trustedDependencies` in `package.json`.

### Cooldown (7 days = 604800 seconds)

Create/append `~/.bunfig.toml`:

```toml
[install]
minimumReleaseAge = 604800
```

### Escape hatch

```toml
[install]
minimumReleaseAgeExcludes = ["<pkg>"]
```

---

## Live verification

Pick any package version published <7 days ago and confirm install is blocked:

```bash
V=$(npm view electron-to-chromium version)
npm view electron-to-chromium time --json | grep "$V"   # confirm publish date is recent

mkdir -p /tmp/cooldown-test && cd /tmp/cooldown-test
echo '{"name":"t","version":"0.0.0"}' > package.json

bun add "electron-to-chromium@$V"    # expect: blocked by minimum-release-age
pnpm add "electron-to-chromium@$V"   # expect: error suggesting minimumReleaseAgeExclude

cd / && rm -rf /tmp/cooldown-test
```

If the latest version happens to be >7 days old, pick another fast-moving package (e.g. `caniuse-lite`).

---

## Summary

| Manager | Version req | Scripts setting | Cooldown setting | Config location |
|---------|-------------|----------------|-----------------|-----------------|
| **npm** | >= 11.10 | `ignore-scripts=true` | `min-release-age=7` (days) | `~/.npmrc` |
| **pnpm** | >= 10.16 | default blocked (v10+) | `minimumReleaseAge=10080` (minutes) | global rc |
| **Bun** | >= 1.3.8 | default blocked | `minimumReleaseAge=604800` (seconds) | `~/.bunfig.toml` |
