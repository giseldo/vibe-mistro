# Distribution: Windows x64 target — unsigned NSIS on windows-latest

**Status: ACCEPTED** (2026-11-02). Amends **ADR-0018** (distribution: GitHub Releases — signed
arm64) with a third platform; ADR-0018's mac/Linux decisions stand.

## Context

ADR-0018 scoped the Release to macOS arm64 (signed + notarized) and Linux x64 (unsigned) —
no Windows job existed, so tags published nothing Windows users could install. A locally-proven
`electron-builder --win nsis -c.electronVersion=…` build succeeds on a Windows dev machine
(`apps/desktop/dist/Vibe-Mistro-x64.exe`, 2026-11-02), and `@lydell/node-pty` ships a prebuilt
`win32-x64` binary, so the terminal subsystem needs no native rebuild. There is no Windows code
signing certificate, and the repo has no Windows signing secrets — only the Apple triple
(CSC `/ APPLE_ID / APPLE_APP_SPECIFIC_PASSWORD / TEAM_ID` per ADR-0018).

## Decision

1. **NSIS installer (x64), built on `windows-latest`** — the Electron default. A new
   `release-win` job runs as an **independent lane with no `needs:`**: electron-builder
   creates-or-appends to the tagged Release itself, and a repo without Apple credentials must
   not implicitly skip Windows whenever the macOS job fails its secrets guard
   (proven: releases skipped "needs"-gated jobs on failed deps). Unlike `release-linux`
   (which still sequences after `release`), Windows may create the Release if it lands first —
   with `releaseType: release` electron-builder publishes it once complete. The job forces
   `shell: bash` (Windows runners default to PowerShell; the shared guards are bash) and keeps
   the same Checkout → Bun → `bun install` → tag-version-guard shape.
2. **Unsigned, by design** — Symantec/DigiCert EV code-signing certs are costly (OV/EV tiers,
   annual renewal) and only matter once Windows users exist at scale (mirrors ADR-0018's
   unsigned Linux). Consequence: SmartScreen will show "unknown publisher" for the first
   installers. Signing (Azure Trusted Signing or an OV cert) is a revisitable follow-up, not a
   blocker for an unreleased app.
3. **`-c.electronVersion` pinned per build, not in `electron-builder.yml`**: electron-builder
   cannot compute the version from installed node modules in a Bun-workspaces layout — it does
   not walk up out of `apps/desktop` and `^42.5.2` is no fixed version
   (electron-userland/electron-builder#3984). The job resolves the *installed* electron's
   version and passes it via `-c.electronVersion`, so the packaged binary always matches the
   electron it was packed with. **The resolver must hard-fail on a miss**: the first version of
   it (`node -p "$(require(…))"` → `echo … >> $GITHUB_OUTPUT`) silently wrote an EMPTY
   `version=` output when `require` failed inside Bun's `node_modules/.bun` layout, and the
   failure only surfaced as app-builder's `version not specified` a minute later
   (run 41562753264). The resolver now tries the known layout roots and throws — failing the
   step — when it finds neither, instead of a silent output.
4. **New scripts** in `apps/desktop/package.json`: `dist:win` / `release:win` (mirror
   `dist:linux` / `release:linux`, `--publish never`/`always`). The workflow calls
   `release:win`, appending the pin flag — bun forwards args after `--`.
5. **`icon.ico` already exists** in `buildResources` (`resources/icon.ico`), so the installer
   carries the app icon without generating one from the PNG.
6. **Updater feed**: electron-builder's NSIS default — `latest.yml` + `.exe.blockmap` — is what
   `electron-updater` reads on Windows; nothing new to wire. (Note: latest.yml has no signing
   of its own; an unsigned updater on an unsigned app is the Windows norm, as with Linux.)

## Consequences

- Windows users download `Vibe-Mistro-x64.exe` (versionless top-level `artifactName`) from the
  Release, with SmartScreen's unknown-publisher prompt on first run.
- The app self-updates on Windows via `electron-updater` reading `latest.yml` from the same
  GitHub Releases channel (running-app updates still passive, ADR-0018 point 6).
- The `release-win` job runs in parallel with `release-linux` after `release`; CI for a tag is
  three runners signing/uploading to one stable Release.
- Re-running the workflow on an old tag re-uploads Windows assets with a stale
  `latest.yml` — same re-run caveat as macOS; workflow_dispatch on the tag's ref is the
  re-publish path.
