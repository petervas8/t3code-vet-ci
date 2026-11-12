# t3code-vet-ci

Builds audited commits of `petervas8/t3code-vetted` on throwaway GitHub runners, so `pnpm install` and build scripts never run on your machine. This repository holds only the workflow; Actions stay disabled on the vetted fork.

Each run builds one `audited/*` tag and publishes a release named `vetted-<sha12>`:

| Asset | What it is |
|---|---|
| `T3-Code-*-arm64.dmg` | macOS desktop app, Apple Silicon. Signed ad hoc, not notarized. |
| `T3-Code-*.AppImage`, `*.deb` | Linux x64 desktop app |
| `t3-*-linux-x64.tar.gz` | Linux x64 server (`t3` executable) for a headless machine |
| `T3-Code-*-android-arm64.apk` | Android app, arm64 phones, signed with your key |
| `SHA256SUMS` | Checksums of everything above |

## One-time setup

1. Create the repository (private) and push this one: `gh repo create petervas8/t3code-vet-ci --private --source . --push`.
2. Make a fine-grained token with **Contents: read** on `petervas8/t3code-vetted` only, and store it: `gh secret set VETTED_REPO_TOKEN -R petervas8/t3code-vet-ci`.
3. Make an Android signing key once and keep a backup. Android only installs an update signed by the same key.
   ```bash
   keytool -genkeypair -v -keystore t3code-release.jks -alias t3code -keyalg RSA -keysize 4096 -validity 10000
   base64 -i t3code-release.jks | gh secret set ANDROID_KEYSTORE_BASE64 -R petervas8/t3code-vet-ci
   gh secret set ANDROID_KEY_ALIAS -R petervas8/t3code-vet-ci --body t3code
   gh secret set ANDROID_KEYSTORE_PASSWORD -R petervas8/t3code-vet-ci
   ```
   Set `ANDROID_KEY_PASSWORD` too only if the key's password differs from the keystore's.

## Running a build

`vet.sh approve` and `vet.sh retag` start a build after they push the tag. To start one by hand:

```bash
gh workflow run build.yml -R petervas8/t3code-vet-ci                          # audited head of vetted
gh workflow run build.yml -R petervas8/t3code-vet-ci -f tag=audited/<sha12>   # a specific tag
gh workflow run build.yml -R petervas8/t3code-vet-ci -f force=true            # rebuild an existing release
```

A run fails at the start if the head of `vetted` has no `audited/*` tag. It skips a commit that already has a release unless you pass `force`.

## Installing

- **macOS:** the app is not notarized, so remove the quarantine flag after copying it to Applications: `xattr -dr com.apple.quarantine "/Applications/T3 Code.app"`.
- **Linux server:** unpack the archive, then follow the upstream docs for `t3 serve` and `t3 service install`. Keep it on localhost or your tailnet.
- **Android:** uninstall the Play Store T3 Code first. It uses the same package name and a different signing key, so Android refuses to install over it.

## Things that still reach upstream

- **`t3 update`, and Update server in the mobile app**, install upstream's unvetted release from `pingdotgg/t3code`. Don't use them. Install the next build from here instead.
- **Desktop auto-update** looks for releases in this repository, which is private, so its checks fail. Set `T3CODE_DISABLE_AUTO_UPDATE=1`.
- **Over-the-air mobile updates** are turned off at build time (`T3CODE_MOBILE_UPDATES_ENABLED=0`). Without that, the APK would download JS from upstream's Expo project on every launch.
- **Telemetry** can only be turned off at runtime: `T3CODE_TELEMETRY_ENABLED=false`.
- **T3 Connect** sign-in is not built in, because its Clerk and relay settings come from upstream's own deploy. Connect over LAN or Tailscale instead.

## Cost

Private repositories pay for runner minutes, and macOS minutes count 10×. One full run uses about 20 macOS minutes (≈200 billed minutes) plus about 60 Linux minutes. On GitHub Free's 2,000 minutes a month that is roughly 7 builds. Approve in larger steps, or make this repository public: the source is open anyway, and public repos run for free.
