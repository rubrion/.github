# Flutter → Play Store + App Store — release template (push-based)

Store publishing for a Rubrion Flutter app, **without release PRs** — fits a solo
maintainer. Reference implementation: **rubrion/avilink-monorepo**.

## Model (push → build → ship)

| Thing | Source | Managed by |
| --- | --- | --- |
| **Version name** (`X.Y.Z`) | `pubspec.yaml` `version:` | **you** — a one-line bump per release (deliberate) |
| **Build number** (versionCode / CFBundleVersion) | CI | Codemagic `$PROJECT_BUILD_NUMBER` (shared across stores) |
| **"What's new"** | Conventional Commits since the last tag | generated at build → `release_notes.txt` |

Why no release-please / PRs: the PR gate is a team review step. Solo, you don't
need it (and org settings often block Actions from opening PRs). The only
error-prone part — the build number — is fully automated; semver is a one-liner
you control.

## Setup for a new app

1. **Conventional Commits** — use `feat:` / `fix:` / `perf:` / `chore:` / `!`.
   The build notes are generated from `feat`/`fix`/`perf` since the last tag.
2. **Codemagic** — copy `codemagic.yaml`, replace every `CHANGE_ME`:
   - `CHANGE_ME_flutter_version`, `CHANGE_ME_keystore_reference`,
     `CHANGE_ME_env_group`, `CHANGE_ME_asc_api_key_name`, `CHANGE_ME_bundle_id`.
   - Env group (Secure): `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` (Play service-account
     JSON), `GOOGLE_PLAY_TRACK` (`internal`|`alpha`|`production`).
   - Android keystore uploaded under the reference name.
   - iOS: an App Store Connect API key integration + the App ID / app record.
   - **One-time:** set the Codemagic project build number **above the latest store
     versionCode** (so `$PROJECT_BUILD_NUMBER` never collides with an existing
     upload).

## Flow per release

1. Commit features to `main` (Conventional Commits).
2. Bump `pubspec` `version:` if it's a notable release (one line — or leave it).
3. Run the Codemagic store workflow → builds with your semver +
   `$PROJECT_BUILD_NUMBER`, generates `release_notes.txt` from the commits, tags
   the release, and publishes.

## Notes → store "what's new"

`release_notes.txt` (generated at build) is an artifact. The declarative Codemagic
publish does not push it automatically; to auto-fill the store "what's new", use a
scripted publish:

- **Play**: `google-play bundles publish … --release-notes "pt-BR=$(cat release_notes.txt)"`
- **App Store**: `app-store-connect` / fastlane `deliver --release_notes` with the same file.

Confirm the flag on the first CI run; the artifact is always produced as a fallback.
