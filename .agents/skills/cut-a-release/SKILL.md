---
name: cut-a-release
description: |
  Step-by-step checklist for releasing a new version of the vscode-unmute-video
  extension to the VS Code Marketplace and Open VSX. Use this skill when asked
  to release, bump the version, or publish.
---

# Cutting a Release

The release pipeline is fully automated by `.github/workflows/release.yml`.
The human steps are version bumping, changelog editing, and tag pushing.

---

## Step 1 — Determine the new version

Follow [Semantic Versioning](https://semver.org/):
- **Patch** (`X.Y.Z+1`): bug fixes, security updates, dependency bumps with no
  behaviour change.
- **Minor** (`X.Y+1.0`): new user-facing features, backward-compatible.
- **Major** (`X+1.0.0`): breaking changes (rare for VS Code extensions).

Current version is in `package.json` → `"version"`.

---

## Step 2 — Bump `package.json`

```json
"version": "0.2.6"
```

---

## Step 3 — Update `CHANGELOG.md`

Move everything under `## [Unreleased]` into a new dated section:

```markdown
## [0.2.6] - 2026-MM-DD

### Fixed
- Description of the fix.

## [0.2.5] - 2026-08-11
...
```

Add the link reference at the bottom of the file:
```markdown
[0.2.6]: https://github.com/yutabee/vscode-unmute-video/compare/v0.2.5...v0.2.6
```

Keep the empty `## [Unreleased]` section above the new entry for future PRs.

---

## Step 4 — Verify CI locally

```bash
npm run compile      # must produce out/ and media/player.js cleanly
npm test             # all tests must pass
npm run lint         # no ESLint errors
npm run package      # produces unmute-video-X.Y.Z.vsix
```

Optionally inspect the `.vsix`:
```bash
unzip -l unmute-video-X.Y.Z.vsix | grep -E 'media/player.js|out/'
```

---

## Step 5 — Open PR and merge

Create a PR titled `chore: release vX.Y.Z`. Wait for CI green, then merge to
`main`.

---

## Step 6 — Tag and push

```bash
git tag vX.Y.Z
git push origin vX.Y.Z
```

This triggers `.github/workflows/release.yml`, which:
1. Builds and tests.
2. Packages the `.vsix`.
3. Publishes to VS Code Marketplace (needs `VSCE_PAT` secret).
4. Publishes to Open VSX (needs `OVSX_PAT` secret).
5. Creates a GitHub Release and attaches the `.vsix`.

---

## Step 7 — Verify

Watch the **Release** workflow run in GitHub Actions. Each publish step runs
with `continue-on-error`, so check the individual step logs — a transient
registry error will not fail the whole workflow but won't publish either.

If a publish step fails, re-run via:
```bash
gh workflow run release.yml --ref main
```

---

## Checklist

- [ ] `package.json` version bumped
- [ ] `CHANGELOG.md` — `[Unreleased]` entries moved to `[X.Y.Z]` section
- [ ] `CHANGELOG.md` — link reference added at bottom
- [ ] `npm run compile` clean
- [ ] `npm test` passing
- [ ] `npm run lint` clean
- [ ] `npm run package` succeeds, `.vsix` contains `media/player.js`
- [ ] PR opened, CI green, merged to `main`
- [ ] Tag pushed: `git tag vX.Y.Z && git push origin vX.Y.Z`
- [ ] Release workflow completed successfully for both registries
