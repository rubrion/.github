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

### One-time org setup (admin — once for the whole org)

Do these once; every current and future repo inherits them. The App (`rubrion`,
App ID `1117691`) already exists.

1. **Install the `rubrion` App** on the app repos (or "All repositories") with
   **Repository permissions → Contents: Read and write** and **Metadata: Read**.
2. **Branch protection / rulesets**: on each branch the bump runs on, add the
   `rubrion` App to the **bypass list** so it can push the release commit + tag.
3. **Org Actions secret** `AUTOMATION_APP_PRIVATE_KEY` = the full contents of a
   private key (`.pem`) generated for the App.
4. **Org Actions variable** `AUTO_RELEASE` = `true`. Every caller is gated on it,
   so nothing runs until it is set — and flipping it off pauses releases org-wide.

Until steps 3–4 are done the workflow is dormant (it is skipped, not failed).

---

### When a new app repo is created (per-app checklist)

1. In the repo: **Actions → New workflow → "by rubrion" → Mobile release (auto
   version bump) → Configure**. This adds `.github/workflows/release.yml`.
2. Edit four things in that file:
   - `app-dir:` — the folder holding `pubspec.yaml` or `package.json` (`.` for a
     single-app repo; e.g. `frontend`, `rubry-app` in a monorepo).
   - `tag-prefix:` — unique per app so monorepo apps don't share a version line
     (e.g. `v`, `rubry-app-v`).
   - `on.push.paths:` — narrow to that app's folder so unrelated commits don't
     bump it.
   - `on.push.branches:` — your source-of-truth branch (the template fills in the
     default branch; change it if you release from `develop`).
3. Confirm the one-time org setup covers this repo (App installed + secret +
   `AUTO_RELEASE=true`).
4. Done. The **first** push to that branch is a **bootstrap**: it tags the current
   version without bumping, to establish a baseline. The next `feat` / `fix` /
   `perf` push bumps and tags.

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
