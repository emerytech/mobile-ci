# mobile-ci

Shared, app-agnostic CI for Expo apps. Builds run **locally** via `eas build --local`
on self-hosted runners — **zero EAS cloud build billing**. Only cost is your own compute.

- iOS builds run on the macOS runner (`macbook-publish`, MacBook Pro M2).
- Android builds run on the Linux runner.
- All per-app config (bundle IDs, credentials, submit keys) lives in each app repo's
  `eas.json` + EAS-hosted credentials — this repo holds none of it.

## Reusable workflow

`.github/workflows/eas-local-build.yml` — called by app repos.

Inputs: `platform` (ios|android|all), `profile` (eas.json build profile),
`submit` (bool), `node-version`, `eas-version`. Secret: `EXPO_TOKEN`.

## Add to an app repo

Drop this into `<app>/.github/workflows/build.yml`:

```yaml
name: Build
on:
  push:
    tags: ['v*']
  workflow_dispatch:
    inputs:
      platform: { type: choice, options: [all, ios, android], default: all }
      submit:   { type: boolean, default: false }
jobs:
  build:
    uses: emerytech/mobile-ci/.github/workflows/eas-local-build.yml@main
    with:
      platform: ${{ inputs.platform || 'all' }}
      submit:   ${{ inputs.submit || false }}
    secrets:
      EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
```

## One-time setup

### 1. EXPO_TOKEN
Create a robot/access token at https://expo.dev (account with access to the app's EAS
project). Then per app repo (or org-wide if you have `admin:org`):

```bash
gh secret set EXPO_TOKEN --repo emerytech/<app> --body '<token>'
```

`eas build --local` uses this to pull each app's signing credentials from EAS (free).

### 2. Runners (register at ORG scope so all apps share them)
Org → Settings → Actions → Runners → New runner → copy the registration token.

macOS runner (`macbook-publish`) must be labeled `macOS` and org-scoped. Needs Xcode,
CocoaPods, Fastlane. Must run as a **LaunchAgent in a logged-in user session** (not a
root LaunchDaemon) or `codesign` fails on a locked keychain.

Linux runner (Android):

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o r.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf r.tar.gz
./config.sh --url https://github.com/emerytech --token <REG_TOKEN> --labels linux,android --name linux-android
sudo ./svc.sh install && sudo ./svc.sh start
```

Linux prereqs: `openjdk-17-jdk`, Android cmdline-tools, `ANDROID_HOME`/`ANDROID_SDK_ROOT`
in `~/actions-runner/.env`, `sdkmanager --licenses` accepted.

### 3. Store submit (only if `submit: true`)
- iOS: `eas.json` submit profile needs the ASC API key. If the `.p8` is a local path
  (git-ignored), materialize it in CI from a base64 secret before submit, or switch to
  an EAS-hosted ASC key.
- Android: add a `submit.<profile>.android` block with a Google service-account JSON.

## Cost
`eas build --local` build compute = **$0** on EAS. `eas submit` = **$0**. No cloud minutes.
