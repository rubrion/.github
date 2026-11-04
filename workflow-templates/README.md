# Org workflow templates

Templates offered to every repo in the org under **Actions → New workflow → "by
rubrion"**. Each is a `*.yml` paired with a `*.properties.json`.

| Template | What it does |
|---|---|
| **Mobile release (auto version bump)** (`mobile-release.yml`) | Auto-bumps an app's semantic version from Conventional Commits and tags it. |
| **Cloudflare Pages Deployment** (`cloudflare-pages-deployment.yml`) | Builds and deploys a static site / frontend to Cloudflare Pages. |

---

## Mobile release — automatic app versioning

`mobile-release.yml` wires an app repo to the shared, framework-free engine
[`.github/workflows/mobile-version-bump.yml`](../.github/workflows/mobile-version-bump.yml).
On a push it derives the next semantic version from Conventional Commits since
the app's last tag, writes it into the app's version file, then commits and tags
it — **no PRs, no personal access tokens**. The commit is authored by the
**`rubrion` GitHub App** (App ID `1117691`).

The point: an app owner never hand-manages versions, and a brand-new app inherits
all of this by adding one file.

### What you get

- `feat` → **minor**, `fix` / `perf` → **patch**, `!` or `BREAKING CHANGE` → **major**.
- Version file **auto-detected** under `app-dir`: Flutter `pubspec.yaml`, or
  Capacitor / Node `package.json`. No per-repo adapter.
- One git tag per release, `<tag-prefix><version>` — the build (Codemagic) reads
  it for the `versionName` and the store "what's new".
- The build number / `versionCode` stays the build's job (Play's latest + 1); the
  engine only owns the semver **name**.

---

### One-time setup (once for the org)

The App (`rubrion`, App ID `1117691`) already exists. Do this once:

- **Generate the App private key** (GitHub App → Private keys → Generate a private
  key) and store it in **Infisical** as the single source of truth:
  project `infra` · env `prod` · path `/github-app` · secret
  `AUTOMATION_APP_PRIVATE_KEY`. Every app repo copies this one key; rotate it here.

> **Plan note:** on GitHub **Free**, private repos can't read **org-level** Actions
> secrets, so the key and the on-switch are set **per repo** (copied from
> Infisical — see the checklist). Upgrade to Team to share them org-wide instead.

Everything else — installing the App on the repo, branch bypass, the repo secret,
the `AUTO_RELEASE` switch — is per app repo (repo-scoped on Free anyway), in the
checklist below. The workflow stays dormant (skipped, not failed) until its repo
has both the secret and `AUTO_RELEASE=true`.

---

### When a new app repo is created (per-app checklist)

1. **Add the workflow**: Actions → New workflow → "by rubrion" → *Mobile release
   (auto version bump)* → Configure. This adds `.github/workflows/release.yml`.
2. **Edit four lines** in it:
   - `app-dir:` — folder holding `pubspec.yaml` / `package.json` (`.` for a
     single-app repo; e.g. `frontend`, `rubry-app`).
   - `tag-prefix:` — unique per app (e.g. `v`, `rubry-app-v`).
   - `on.push.paths:` — narrow to that app's folder.
   - `on.push.branches:` — the source-of-truth branch (the template fills in the
     default branch; change it if you release from `develop`).
3. **Install the App on this repo**: add it to the `rubrion` App's *selected
   repositories* (Org settings → GitHub Apps → `rubrion` → Configure; Contents:
   Read and write). If the release branch is protected, add the App to that
   branch's **bypass list**.
4. **Copy the key from Infisical into a repo secret, and flip the switch**:
   ```bash
   infisical secrets get AUTOMATION_APP_PRIVATE_KEY \
     --projectId 4dbb5673-ad94-406d-9363-4634c8763afa --env prod \
     --path /github-app --type shared --plain --silent \
     --domain https://infisical.rubrion.ai/api \
     | gh secret set AUTOMATION_APP_PRIVATE_KEY -R rubrion/<repo>
   gh variable set AUTO_RELEASE --body true -R rubrion/<repo>
   ```
   (Or paste the key from the Infisical UI into the repo's Actions **secret**
   `AUTOMATION_APP_PRIVATE_KEY`, and add the **variable** `AUTO_RELEASE=true`.)
5. Done. First push to the release branch = **bootstrap** (tags the current
   version, no bump); the next `feat` / `fix` / `perf` push bumps and tags.

---

### Notes

- The engine's own release commit carries `[skip ci]`, so it never loops and never
  triggers a Codemagic build. Trigger builds in the Codemagic UI as usual; they
  read the bumped version file.
- **Exotic stack** (not `pubspec.yaml` / `package.json`)? Pass a `version-script`
  input pointing at a repo script that implements `get` and `set <X.Y.Z>`.
- This automates **versioning only**. Building and publishing the binary stays
  per-stack (e.g. Codemagic), and consumes the version/tag the engine produces.

### Example callers

- avilink-monorepo — `app-dir: frontend`, `tag-prefix: v`, on `develop`.
- rubry-monorepo — `app-dir: rubry-app`, `tag-prefix: rubry-app-v`, on `main`.
