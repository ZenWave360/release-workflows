# release-workflows

Shared, reusable GitHub Actions release pipelines for ZenWave360 OSS
libraries published to Maven Central (and optionally npm). Private repo:
callers must be in the `ZenWave360` organization (Settings → Actions →
General → Access → "Accessible from repositories in the 'ZenWave360'
organization").

## Security model

- Every `environment:` and `secrets:` reference inside these reusable
  workflows resolves against the **calling repository**, never against this
  one. This repo never holds, sees, or forwards any repo's Central token,
  signing key, or npm credential — callers pass secrets explicitly by name.
- **Never call these with `secrets: inherit`.** Pass named secrets only
  (see examples below). `validate-release.yml` needs no secrets at all.
- Pin the `uses:` reference in every caller to a full commit SHA, not a
  branch or tag (`@main` would let anyone who can land a PR here silently
  change what every consuming repo's release pipeline runs). Let Dependabot
  bump the pin via a normal reviewed PR in each caller.
- This repo's own `main` needs at least the same protection as any
  individual library's `main`: required PR, CODEOWNERS, no bypass actor.
  A compromise here affects every repo that calls it.

## Workflows

- **`validate-release.yml`** — no secrets, read-only, no build tool.
  Validates the version format, the release notes file, and that the
  version isn't already published anywhere (git tag, GitHub release, Maven
  Central, npm). Shared by both pipelines below.
- **`release-maven.yml`** — full pipeline for Maven repos: validate →
  prepare-release (`maven-release-plugin`) → build-and-publish-central
  (`mvn deploy`) → create-github-release → sync-develop.
- **`release-gradle.yml`** — full pipeline for Gradle/Kotlin-Multiplatform
  repos: validate → prepare-release (`sed` version bump) →
  build-and-publish-central (`./gradlew build publishToMavenCentral`) →
  create-github-release → publish-npm (optional, OIDC trusted publishing) →
  sync-develop.
- **`publish-snapshots-maven.yml`** / **`publish-snapshots-gradle.yml`** —
  single-job reusable workflows for `-SNAPSHOT` publication on push to
  `develop`/`next`. Not secretless: the deploy step runs with all four
  secrets present, same residual risk as `build-and-publish-central`. Each
  has no `on:` trigger of its own (reusable `workflow_call` workflows can't
  also declare `push:`/`schedule:` triggers that would fire *in this repo*)
  — the caller keeps a thin `publish-maven-snapshots.yml` with its own
  `on: push: branches: [develop, next]` and calls this as a single job.
  `publish-snapshots-maven.yml` also prints the aggregate JaCoCo coverage
  for whatever's on `develop`/`next` at that moment (via the same
  `jacoco-csv-files` input `main-maven.yml` uses) — computed and logged
  *after* the deploy step, in a no-secrets step that writes badge output to
  a throwaway directory, never checked out from or pushed to the `badges`
  branch. That's deliberate: it exists because `main-maven.yml` only ever
  reflects `main`'s coverage, and `develop`/`next` can genuinely differ
  between releases — it is not badge generation creeping back into the
  credentialed job, just a coverage number in the log/step summary for this
  branch specifically. `publish-snapshots-gradle.yml` doesn't need the
  equivalent — Kover coverage is already printed to the console there.
- **`main-maven.yml`** / **`main-gradle.yml`** — secretless CI: build, test,
  generate coverage badges (JaCoCo for Maven, Kover for Gradle), push them to
  the caller's own `badges` branch using the ambient `GITHUB_TOKEN`. Never
  touches Central/npm/signing credentials. This is also where coverage
  badges belong — not inside the release or snapshot jobs, which is why
  those don't generate badges themselves.

## Caller prerequisites

**Every caller repo needs:**

- A `main` branch ruleset requiring PRs (0 required approvals is fine for a
  solo maintainer), no bypass actor, Settings → Actions → Workflow
  permissions set to read-only by default.
- A `maven-central-upload` GitHub Environment (or whatever name you pass as
  `central-environment`): required reviewer, deployment branches restricted
  to `main`, secrets `CENTRAL_USERNAME` / `CENTRAL_TOKEN` / `SIGN_KEY` /
  `SIGN_KEY_PASS`. Repository-level copies of these secrets must not exist —
  environment-level secrets only.
- A `maven-central-snapshots` GitHub Environment (or whatever name you pass
  as `snapshot-environment`): no required reviewer (snapshots stay
  automatic), deployment branches restricted to `develop`/`next`, same four
  secrets as the upload environment (ideally a separate Central token so it
  can be rotated independently).
- A `release-notes/release-notes.v<VERSION>.md` file committed before
  dispatching a release.
- An orphan `badges` branch (or whatever name you pass as `badges-branch`)
  if you want `main-maven.yml`/`main-gradle.yml`'s coverage badges — create
  it once with `git checkout --orphan badges && git rm -rf . && git commit
  --allow-empty -m "init" && git push origin badges`.

**Maven callers additionally need**, in `pom.xml`, on `maven-release-plugin`:

```xml
<configuration>
    <pushChanges>false</pushChanges>
    <tagNameFormat>v@{project.version}</tagNameFormat>
    <autoVersionSubmodules>true</autoVersionSubmodules>
</configuration>
```

`pushChanges=false` is why `release:prepare` never attempts (and never
fails) a direct push to `main` — it commits and tags locally only, and
`peter-evans/create-pull-request` handles pushing to a throwaway branch and
opening the PR. Without this, `release:prepare` will try to push directly
and fail against the required-PR ruleset.

**Gradle/npm callers additionally need**, if `publish-npm: true`:

- A `npm-publish` GitHub Environment (or whatever name you pass as
  `npm-environment`): required reviewer, deployment branches restricted to
  `main`, **no secrets** (OIDC trusted publishing).
- The npm package's Trusted Publisher configured on npmjs.com (GitHub
  Actions, this org/repo, the caller's release workflow filename, the
  `npm-environment` name) — see the individual repos' `docs/release-security.md`
  for the one-time manual first-publish steps.

## Example caller: Maven

```yaml
# .github/workflows/release-from-notes.yml
name: Release from Notes

on:
  workflow_dispatch:
    inputs:
      version:
        required: true
        type: string
      developmentVersion:
        required: false
        type: string
        default: ""

permissions:
  contents: read

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  release:
    uses: ZenWave360/release-workflows/.github/workflows/release-maven.yml@<pin-to-full-sha> # vX.Y.Z
    with:
      version: ${{ inputs.version }}
      development-version: ${{ inputs.developmentVersion }}
      # group-id/artifact-id omitted -- derived from pom.xml's <groupId>/<artifactId>
    secrets:
      CENTRAL_USERNAME: ${{ secrets.CENTRAL_USERNAME }}
      CENTRAL_TOKEN: ${{ secrets.CENTRAL_TOKEN }}
      SIGN_KEY: ${{ secrets.SIGN_KEY }}
      SIGN_KEY_PASS: ${{ secrets.SIGN_KEY_PASS }}
```

## Example caller: Gradle with npm

```yaml
# .github/workflows/release-from-notes.yml
name: Release from Notes

on:
  workflow_dispatch:
    inputs:
      version:
        required: true
        type: string
      developmentVersion:
        required: false
        type: string
        default: ""
      publishNpm:
        required: false
        type: boolean
        default: false

permissions:
  contents: read

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  release:
    uses: ZenWave360/release-workflows/.github/workflows/release-gradle.yml@<pin-to-full-sha> # vX.Y.Z
    with:
      version: ${{ inputs.version }}
      development-version: ${{ inputs.developmentVersion }}
      # group-id omitted -- derived from build.gradle.kts's 'group = ...' line
      artifact-id: json-schema-ref-parser-kmp
      publish-npm: true
      dispatch-publish-npm: ${{ inputs.publishNpm }}
      npm-package: "@zenwave360/json-schema-ref-parser-kmp"
      npm-package-dir: build/js/packages/json-schema-ref-parser-kmp
    secrets:
      CENTRAL_USERNAME: ${{ secrets.CENTRAL_USERNAME }}
      CENTRAL_TOKEN: ${{ secrets.CENTRAL_TOKEN }}
      SIGN_KEY: ${{ secrets.SIGN_KEY }}
      SIGN_KEY_PASS: ${{ secrets.SIGN_KEY_PASS }}
```

## Example caller: CI + coverage badges (Gradle)

```yaml
# .github/workflows/main.yml
name: Verify Main and Publish Coverage

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    uses: ZenWave360/release-workflows/.github/workflows/main-gradle.yml@<pin-to-full-sha> # vX.Y.Z
```

For Maven, use `main-maven.yml` instead and pass the module-specific
`jacoco-csv-files` input (no sensible shared default — every multi-module
repo lists its own paths):

```yaml
jobs:
  build:
    uses: ZenWave360/release-workflows/.github/workflows/main-maven.yml@<pin-to-full-sha> # vX.Y.Z
    with:
      jacoco-csv-files: >
        ./plugins/one-module/target/site/jacoco/jacoco.csv
        ./plugins/another-module/target/site/jacoco/jacoco.csv
```

## Example caller: snapshot publication

```yaml
# .github/workflows/publish-maven-snapshots.yml
name: Build and Publish Snapshots

on:
  push:
    branches: [develop, next]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: maven-snapshots-${{ github.ref }}
  cancel-in-progress: false

jobs:
  publish-snapshot:
    uses: ZenWave360/release-workflows/.github/workflows/publish-snapshots-gradle.yml@<pin-to-full-sha> # vX.Y.Z
    secrets:
      CENTRAL_USERNAME: ${{ secrets.CENTRAL_USERNAME }}
      CENTRAL_TOKEN: ${{ secrets.CENTRAL_TOKEN }}
      SIGN_KEY: ${{ secrets.SIGN_KEY }}
      SIGN_KEY_PASS: ${{ secrets.SIGN_KEY_PASS }}
```

Use `publish-snapshots-maven.yml` for Maven repos instead — same shape, same
secrets, plus a required `jacoco-csv-files` input (same list `main-maven.yml`
uses) to print the aggregate coverage for whatever's on `develop`/`next`:

```yaml
jobs:
  publish-snapshot:
    uses: ZenWave360/release-workflows/.github/workflows/publish-snapshots-maven.yml@<pin-to-full-sha> # vX.Y.Z
    with:
      jacoco-csv-files: >
        ./plugins/one-module/target/site/jacoco/jacoco.csv
        ./plugins/another-module/target/site/jacoco/jacoco.csv
    secrets:
      CENTRAL_USERNAME: ${{ secrets.CENTRAL_USERNAME }}
      CENTRAL_TOKEN: ${{ secrets.CENTRAL_TOKEN }}
      SIGN_KEY: ${{ secrets.SIGN_KEY }}
      SIGN_KEY_PASS: ${{ secrets.SIGN_KEY_PASS }}
```

Maven callers also need a `maven-enforcer-plugin` execution in `pom.xml`
with id `enforce-snapshot-version` that fails the build unless the project
version ends in `-SNAPSHOT`; the workflow invokes it by that exact id.

Get `group-id`/`artifact-id` right — they're used to derive the Maven Central
path (`https://repo1.maven.org/maven2/<group-id with dots replaced by
`/`>/<artifact-id>/`) for the duplicate-publish check. A wrong value doesn't
break the release, it silently makes the duplicate-publish check a false
negative, which is worse than an obvious failure. Double-check it against an
existing published version's actual URL before trusting it.

**Both build tools should omit `group-id`/`artifact-id` when they can be
derived from the checked-out source**, rather than copying them into the
caller's `with:` block — that copy is exactly one groupId rename (this org
has already had one) away from silently drifting out of sync and turning
the duplicate-publish check into a false negative:

- **Gradle**: `group-id` is read from `build.gradle.kts`'s `group = "..."`
  line when left empty. `artifact-id` is **not** auto-derived for Gradle —
  there's no single canonical field for it (it usually defaults to
  `rootProject.name` in a separate `settings.gradle.kts`, which isn't
  authoritative for every layout) — pass it explicitly.
- **Maven**: both `group-id` and `artifact-id` are read from `pom.xml`'s
  root `<project>` element when left empty (falling back to
  `<project><parent><groupId>` if `groupId` isn't declared directly).
  Parsed with Python's XML tree scoped to direct children only — never a
  plain `grep`, since every `<dependency>` also declares its own
  `<groupId>`/`<artifactId>`.

Only pass these explicitly as an override (e.g. a repo whose Gradle layout
doesn't fit the single-`group =`-line assumption, or a multi-module Maven
repo whose top-level `pom.xml` doesn't carry the coordinates you want
checked).

## npm snapshot train for Gradle callers

Enable optional npm snapshots on the existing snapshot caller:

```yaml
permissions:
  contents: read
  id-token: write
jobs:
  publish-snapshot:
    uses: ZenWave360/release-workflows/.github/workflows/publish-snapshots-gradle.yml@<pin-to-full-sha>
    with:
      publish-npm: true
      npm-package: "@zenwave360/dsl"
      npm-package-dir: build/js/packages/dsl-kotlin
    # Pass the same four named Central/signing secrets as before.
```

After Maven snapshot publication succeeds, a separate npm job rebuilds/tests
without Central/signing secrets, verifies the generated package identity and
version, and maps Maven `1.9.0-SNAPSHOT` to immutable npm
`1.9.0-next.RUN_NUMBER.RUN_ATTEMPT`. Each publish moves the `next` tag.
The generated version may be the Maven snapshot version or its local `-next.0`
equivalent; this shared workflow owns the CI version and tag.

Create a secretless `npm-snapshots` environment in the caller, allowing
`develop`/`next` without a required reviewer for automatic publishing. Add an
npm trusted publisher for the caller repository, its snapshot workflow filename,
and that environment. The package must first be published manually. Optional
inputs `npm-environment` and `npm-node-version` default to `npm-snapshots`
and Node 24. Existing callers without `publish-npm: true` keep Maven-only
publishing. npm-enabled callers must grant `id-token: write`.

The shared Gradle release workflow explicitly publishes stable `1.9.0` to
`latest` and prereleases to `next`. Removing Maven's `-SNAPSHOT` therefore
removes npm's prerelease suffix. It does not unpublish old npm versions or delete
the `next` tag. npm OIDC supports publication, not `dist-tag rm`; removing
that registry tag requires a separate authenticated operation.
