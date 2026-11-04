# Flutter → Play Store + App Store — release template

Automated versioning + changelog + store publishing for a Rubrion Flutter app.
Reference implementation: **rubrion/avilink-monorepo**.

## Model (one semver, one changelog, injected build number)

| Thing | Source | Who manages it |
| --- | --- | --- |
| **Version name** (`X.Y.Z`) | `pubspec.yaml` `version:` | **release-please** (Conventional Commits) |
| **Build number** (versionCode / CFBundleVersion) | CI | Codemagic `$PROJECT_BUILD_NUMBER` (shared by both stores) |
| **"What's new"** | `CHANGELOG.md` latest section → `release_notes.txt` | generated in CI |

Why: semver (marketing) and build number (upload id) are different problems. Hand-
managing the build number is the usual source of "versionCode already used" errors —
so CI injects a project-wide monotonic number into every build, for both stores.

## Setup for a new app

1. **Conventional Commits** — the repo must use `feat:` / `fix:` / `chore:` / `!`.
2. **release-please** — copy these to the repo root and edit:
   - `release-please-config.json` — set `package-name`; the package key (`frontend`)
     is the Flutter package dir (use `.` if the app is at the repo root).
   - `.release-please-manifest.json` — set the current semver.
   - Add the workflow: either the **"Release Please" starter** (Actions → New
     workflow, from this org), or call the reusable one:
     ```yaml
     # .github/workflows/release-please.yml
     name: release-please
     on: { push: { branches: [main] } }
     permissions: { contents: write, pull-requests: write }
     jobs:
       release:
         uses: rubrion/.github/.github/workflows/release-please.yml@main
     ```
3. **Codemagic** — copy `codemagic.yaml`, replace every `CHANGE_ME`:
   - `CHANGE_ME_flutter_version`, `CHANGE_ME_keystore_reference`,
     `CHANGE_ME_env_group`, `CHANGE_ME_asc_api_key_name`, `CHANGE_ME_bundle_id`.
   - Env group vars (Secure): `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` (Play
     service-account JSON), `GOOGLE_PLAY_TRACK` (`internal`|`alpha`|`production`).
   - Android keystore uploaded under the reference name above.
   - iOS: an App Store Connect API key integration named `CHANGE_ME_asc_api_key_name`,
     plus the App ID + app record in App Store Connect.

## Flow per release

1. Merge feature branches to `main` (Conventional Commits).
2. **release-please** opens/updates a release PR (version bump + CHANGELOG). Merge it.
3. Run the Codemagic store workflow → it builds with the new semver +
   `$PROJECT_BUILD_NUMBER`, generates `release_notes.txt`, and publishes.

## Notes → store "what's new"

The build generates `release_notes.txt` (latest CHANGELOG section). The declarative
Codemagic publish does **not** push it automatically; to auto-fill it, use a scripted
publish:

- **Play**: `google-play bundles publish … --release-notes "pt-BR=$(cat release_notes.txt)"`
- **App Store**: `app-store-connect` / fastlane `deliver --release_notes` with the same file.

Confirm the flag on the first CI run; the artifact is always produced as a manual
fallback.
