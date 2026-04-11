---
description: Set up Android Play Store publishing secrets (keystore + Google Play service account)
model: sonnet
---

Guide the user through setting up all secrets required to publish an Android app to the Play Store.

## 1. Gather info

Ask the user for:
- **Key alias** — typically the app name in lowercase (e.g. `foodsense`)
- **Keystore file name** — e.g. `foodsense.jks` (will be created locally, never committed)
- **GitHub repo** — e.g. `pcha/foodsense` (to set secrets via `gh` CLI)
- **Package name** — e.g. `com.github.pcha.foodsense.app` (to confirm Play Console setup)

## 2. Create the keystore

Run:
```bash
keytool -genkey -v \
  -keystore <keystore-file> \
  -alias <key-alias> \
  -keyalg RSA -keysize 2048 -validity 10000
```

`keytool` will prompt for:
- Keystore password (save it — you'll need it)
- Key password (can be the same as keystore password)
- Distinguished name fields (name, org, city, country)

After running, confirm the file was created:
```bash
ls -lh <keystore-file>
```

## 3. Encode the keystore to base64

```bash
base64 -w0 <keystore-file>
```

Capture the output — this is the value for `KEYSTORE_BASE64`.

## 4. Set GitHub secrets via gh CLI

```bash
gh secret set KEYSTORE_BASE64 --repo <github-repo> --body "$(base64 -w0 <keystore-file>)"
gh secret set KEYSTORE_PASSWORD --repo <github-repo>
gh secret set KEY_ALIAS --repo <github-repo> --body "<key-alias>"
gh secret set KEY_PASSWORD --repo <github-repo>
```

For `KEYSTORE_PASSWORD` and `KEY_PASSWORD`, omit `--body` so `gh` prompts for the value interactively (avoids the password appearing in shell history).

## 5. Set up Google Play service account

Instruct the user to do this manually in the browser:

1. Go to [Google Play Console](https://play.google.com/console) → **Setup** → **API access**
2. Link to a Google Cloud project (create one if needed)
3. Click **Create new service account**
4. Follow the link to Google Cloud Console → create the account → grant role **Service Account User**
5. Back in Play Console → grant the service account access with **Release manager** permissions
6. In Google Cloud Console → **IAM & Admin** → **Service Accounts** → select the account → **Keys** → **Add key** → **JSON**
7. Download the JSON file

Then set the secret:
```bash
gh secret set GOOGLE_PLAY_SERVICE_ACCOUNT --repo <github-repo> < path/to/service-account.json
```

## 6. Verify all secrets are set

```bash
gh secret list --repo <github-repo>
```

Confirm these 5 appear:
- `KEYSTORE_BASE64`
- `KEYSTORE_PASSWORD`
- `KEY_ALIAS`
- `KEY_PASSWORD`
- `GOOGLE_PLAY_SERVICE_ACCOUNT`

## 7. Confirm completion

Tell the user:
- All secrets are configured
- The next push to `main` with a conventional commit will trigger a Play Store publish to the `internal` track
- The keystore file should be stored securely offline (e.g. password manager) — it must never be committed to the repo
- Remind them: losing the keystore means losing the ability to update the app on Play Store
