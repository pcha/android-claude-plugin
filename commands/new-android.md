---
description: Create a new Android project from the official architecture-templates (base or multimodule)
model: sonnet
---

Create a new Android project from https://github.com/android/architecture-templates using the following steps:

## 1. Ask for parameters

Ask the user for:
- **Package name** (required) — e.g. `com.example.myapp` (lowercase, valid Android app ID)
- **Data model type** (required) — e.g. `Task`, `Note`, `Product` (PascalCase, used for screen name, ViewModel state, and Room entity)
- **App name** (optional) — e.g. `MyApp` (PascalCase, defaults to `MyApplication`)
- **Template branch** (optional) — `base` (single module, default) or `multimodule`
- **Target directory** (optional) — where to clone (defaults to the last meaningful segment of the package name, avoiding generic names like `app`)
- **Add GitHub release workflow?** (optional, default: `yes`) — adds a CI workflow that creates a GitHub Release on every push to `main`, with version and changelog generated from conventional commits. Also includes optional Play Store publishing (skipped with a warning if secrets are not configured).
- **Create GitHub repository?** (optional, default: `yes`) — creates a public GitHub repo using `gh` CLI, commits all project files, and pushes. Ask for the repo name (defaults to the target directory name, e.g. `foodsense`).

## 2. Validate inputs

- Package name must be lowercase and contain at least one dot (e.g. `com.example.app`)
- Data model must be PascalCase (first letter uppercase)
- App name, if provided, must be PascalCase
- Target directory must not already exist

If any required input is invalid, explain the issue and ask again.

## 3. Clone the template

```bash
git clone https://github.com/android/architecture-templates.git --branch <branch> <target-dir>
```

Where `<branch>` is `base` or `multimodule` and `<target-dir>` is the chosen directory.

## 4. Run the customizer script

```bash
cd <target-dir>
bash customizer.sh <package-name> <DataModel> [AppName]
```

The script will:
- Move source files to the correct package directory
- Replace `android.template` package references with the new package name
- Replace `MyModel` / `myModel` / `mymodel` with the data model name variants
- Rename files and directories accordingly
- Rename `MyApplication` to the app name (if provided)
- Remove template metadata files (`.github/`, `README.md`, `customizer.sh`, `.git/`, etc.)

## 5. Add GitHub release workflow (if requested)

If the user chose to add the workflow (default: yes):

### 5a. Update `app/build.gradle.kts`

Replace the hardcoded `versionCode` and `versionName` lines inside `defaultConfig` with:

```kotlin
versionCode = (findProperty("versionCode") as String?)?.toInt() ?: 1
versionName = (findProperty("versionName") as String?) ?: "1.0"
```

### 5b. Create `.version.yml` in the project root

```yaml
version: 0.0.0
```

### 5c. Create `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate Changelog and Version
        id: changelog
        uses: TriPSs/conventional-changelog-action@v5
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          release-count: 0
          version-file: ./.version.yml
          skip-on-empty: false

      - name: Set up JDK 17
        if: steps.changelog.outputs.skipped == 'false'
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        if: steps.changelog.outputs.skipped == 'false'
        uses: gradle/actions/setup-gradle@v4

      - name: Build Release APK
        if: steps.changelog.outputs.skipped == 'false'
        run: |
          ./gradlew assembleRelease \
            -PversionName=${{ steps.changelog.outputs.version }} \
            -PversionCode=${{ github.run_number }}

      - name: Create GitHub Release
        if: steps.changelog.outputs.skipped == 'false'
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.changelog.outputs.tag }}
          name: ${{ steps.changelog.outputs.tag }}
          body: ${{ steps.changelog.outputs.clean_changelog }}
          files: app/build/outputs/apk/release/app-release-unsigned.apk

      # ── Play Store publishing ──────────────────────────────────────────────
      # Requires the following repository secrets:
      #   KEYSTORE_BASE64              base64-encoded .jks keystore file
      #   KEYSTORE_PASSWORD            keystore password
      #   KEY_ALIAS                    signing key alias
      #   KEY_PASSWORD                 signing key password
      #   GOOGLE_PLAY_SERVICE_ACCOUNT  Google Play service account JSON (contents)
      #
      # Run /android-play-store-setup to configure these secrets.
      # ────────────────────────────────────────────────────────────────────────

      - name: Check Play Store secrets
        if: steps.changelog.outputs.skipped == 'false'
        id: play_secrets
        run: |
          if [ -n "${{ secrets.KEYSTORE_BASE64 }}" ] && [ -n "${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}" ]; then
            echo "available=true" >> $GITHUB_OUTPUT
          else
            echo "available=false" >> $GITHUB_OUTPUT
            echo "::warning::Play Store publishing is not configured. To enable it, follow these steps:%0A%0A1. Create a keystore: keytool -genkey -v -keystore <app>.jks -alias <alias> -keyalg RSA -keysize 2048 -validity 10000%0A2. Encode it: base64 -w0 <app>.jks%0A3. Create a Google Play service account: https://play.google.com/console → Setup → API access%0A4. Add these repository secrets (Settings → Secrets → Actions):%0A   - KEYSTORE_BASE64 (output of step 2)%0A   - KEYSTORE_PASSWORD%0A   - KEY_ALIAS%0A   - KEY_PASSWORD%0A   - GOOGLE_PLAY_SERVICE_ACCOUNT (contents of the downloaded JSON)%0A%0AOr run /android-play-store-setup for a guided setup."
          fi

      - name: Build Release AAB
        if: steps.changelog.outputs.skipped == 'false' && steps.play_secrets.outputs.available == 'true'
        run: |
          ./gradlew bundleRelease \
            -PversionName=${{ steps.changelog.outputs.version }} \
            -PversionCode=${{ github.run_number }}

      - name: Sign AAB
        if: steps.changelog.outputs.skipped == 'false' && steps.play_secrets.outputs.available == 'true'
        id: sign
        uses: r0adkll/sign-android-release@v1
        with:
          releaseDirectory: app/build/outputs/bundle/release
          signingKeyBase64: ${{ secrets.KEYSTORE_BASE64 }}
          alias: ${{ secrets.KEY_ALIAS }}
          keyStorePassword: ${{ secrets.KEYSTORE_PASSWORD }}
          keyPassword: ${{ secrets.KEY_PASSWORD }}

      - name: Publish to Play Store (internal track)
        if: steps.changelog.outputs.skipped == 'false' && steps.play_secrets.outputs.available == 'true'
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}
          packageName: <package-name>
          releaseFiles: ${{ steps.sign.outputs.signedReleaseFile }}
          track: internal
          mappingFile: app/build/outputs/mapping/release/mapping.txt
```

Replace `<package-name>` with the actual package name (e.g. `com.github.pcha.foodsense.app`).

## 6. Create GitHub repository (if requested)

If the user chose to create a GitHub repo (default: yes):

If the `/create-repo` command is available, delegate to it from within the target directory.

Otherwise, execute the steps directly:

1. Get the authenticated GitHub username:
```bash
gh api user --jq '.login'
```

2. Ask for the repo name (defaults to the target directory name) and visibility (`public` by default).

3. Check the repo name is not already taken:
```bash
gh repo view <username>/<repo-name> 2>&1
```

4. Initialize, commit and push:
```bash
cd <target-dir>
git init
git add .
git commit -m "feat: initial project setup"
gh repo create <username>/<repo-name> --public --source . --push
```

## 7. Confirm completion

Report:
- Project location (absolute path)
- Package name used
- Data model name used
- App name used
- Template branch used
- Whether the release workflow was added
- GitHub repo URL (if created)
- Next steps: open the project in Android Studio and `/android-play-store-setup` to configure Play Store publishing

## Notes

- The customizer script requires **bash 4+**. On macOS use `brew install bash` if needed.
- Do NOT open the project in Android Studio before running the customizer.
- Use `gh` CLI to inspect branches or README if the user asks about template options.
- The workflow uses conventional commits for versioning: `feat:` → minor bump, `fix:` → patch bump, `BREAKING CHANGE` → major bump.
