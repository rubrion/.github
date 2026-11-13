# Capacitor store-app pipeline (Codemagic)

A copy-paste **template** for shipping a Capacitor app to Google Play (Android
AAB) and TestFlight (iOS IPA) from Codemagic.

Codemagic has no cross-repo include, so this is a **copy**, not a live shared
config — when the canonical pipeline changes, re-sync this file.

- **Canonical live reference (Capacitor):** `rubrion/rubry-monorepo/codemagic.yaml`
- **Flutter apps:** there is no Flutter template here — copy
  `rubrion/avilink-monorepo/codemagic.yaml` instead (same publish pattern, Flutter
  build steps).

## Use it

1. Copy `codemagic.yaml` to the new repo **root**.
2. Replace every `CHANGE_ME_*`:
   - `CHANGE_ME_app_id` — the app id (`ai.rubry`, `br.com.yourapp`); used for the
     Android package, iOS bundle id, and the Play publish.
   - `CHANGE_ME_app_dir` — the Capacitor app folder (`rubry-app`), as
     `working_directory`.
   - `CHANGE_ME_env_group` — the per-app Codemagic env group.
   - `CHANGE_ME_app_name` — display name in the workflow titles.
   - `CHANGE_ME_asc_integration` — the App Store Connect API key integration.
3. In Codemagic, create the env group with the secure variable
   `GCLOUD_SERVICE_ACCOUNT_CREDENTIALS` (the Play service-account JSON, with access
   to the app in Play Console). The `rubrion-release` keystore + the ASC
   integration are team-level (shared).

## Track follows the branch

`production` branch → **production** track; every other branch → **internal**.
(The publish is scripted with `fastlane supply` because Codemagic validates the
declarative `publishing.*.track` before scripts run, so it can't read a runtime
var.)

## Versioning is separate (already standardized)

The version **name** comes from `package.json` `"version"`, auto-bumped from
Conventional Commits by the shared GitHub Actions engine — add the **Mobile
release** workflow template (`workflow-templates/mobile-release.yml`) to the repo.
This pipeline only reads the name and sets the build number to the store's
latest + 1.

## Before the first PRODUCTION release (Play Console, one-time per app)

- **App access** — depends on whether login is required:
  - **Core app works without login** (optional sign-in): mark the main app as
    *available without special access*. For any **login-gated** features, declare
    *some functionality is restricted* and give a **demo account** (+ a way past
    phone/email OTP) so reviewers can test them. Not a hard blocker.
  - **App behind a login wall**: you **must** give a demo account + OTP bypass
    (reviewers can't receive your SMS/WhatsApp/email code) or review fails.
- **Data safety** — declare the PII collected by sign-up, even when optional
  (e.g. phone number, email).
- **Account deletion** — if the app can create accounts (even optional), provide
  an in-app or web account-deletion path.
- Organization developer accounts are **exempt** from the closed-testing gate,
  but the **first** production release may need to be created manually as a draft;
  after that the scripted `completed` releases work.
