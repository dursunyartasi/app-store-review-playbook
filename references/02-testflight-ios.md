# iOS build and TestFlight

The loop we repeat for every build. Roughly 15–40 minutes; most of the failures below cost a whole cycle when missed.

## Check these two things before building

Both are cheap to check and expensive to discover afterwards.

1. **Is the build number already used?** `GET /v1/builds?filter[app]={id}` — a duplicate is refused at upload.
2. **Is the current version string already published?** If the App Store version is `READY_FOR_SALE`, its release train is closed and the upload fails with **90186** / **ITMS-90062**. You must raise the **version string**, not just the build number — and rebuild, because the version is compiled into the IPA.

We lost four builds on one release learning this.

## Where the version actually lives

- **Managed workflow:** `app.json` → `expo.version` and `expo.ios.buildNumber`.
- **Bare workflow:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` and `CFBundleVersion`, plus pbxproj `MARKETING_VERSION`. **`app.json` is ignored.**

If you edit `app.json` from a script, parse and re-serialise it (`json.load` / `json.dump`). Reading and writing the same handle truncates the file — that cost us a build.

## Build

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # catch import errors pre-archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

The `LANG` prefix is not optional. On Ruby 4.0 with CocoaPods 1.16, `pod install` dies with `Unicode Normalization not appropriate for ASCII-8BIT` — especially with non-ASCII characters anywhere in the project.

Straight `xcodebuild` works too and is the escape hatch when EAS credentials are stale:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Look for `ARCHIVE SUCCEEDED`, then `-exportArchive` with an `ExportOptions.plist` set to `method=app-store`, `signingStyle=automatic`.

### While it builds

**Do not edit source files during an archive.** Metro embeds a half-written bundle and the app crashes on launch. This is not theoretical — it cost us a build and looked like a mystery crash.

## Upload

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Put the `.p8` at `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`; altool finds it there. Takes about 15 seconds.

**Prefer this to `eas submit`.** We have watched `eas submit` hang for 23 minutes with no output and no upload, and fail with "Unable to upload archive. Failed to authenticate for session". altool reports the real error.

## After upload

**`UPLOAD SUCCEEDED` does not mean accepted.** Poll the ASC API until the build is `VALID`; Apple rejects during processing too. Once valid, expire the previous build so testers only see the new one:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Upload errors worth recognising

| Code | Meaning |
|---|---|
| **90062** / ITMS-90062 | This version is already published — raise the version string |
| **90186** | Pre-release train closed — same cause |
| **90713** | A target is missing `CFBundleIconName` — Watch and widget targets need their own icon |
| **ITMS-90863** | Apple Silicon symbol warning. **Normal for Expo apps, not a rejection.** Ignore it. |

## Extra targets

Watch and Live Activity widgets need their own provisioning profiles in `credentials.json`, all under the same distribution certificate. Archiving a Watch target requires the watchOS **device** platform on the Mac — the simulator SDK is not enough:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 GB
```

**Test the extra target's own flows.** Sign-in on our Watch target had never worked — it posted `email` where the backend read `identifier` — and Apple found it before we did, in a 2.1 rejection.

## When Xcode updates mid-project

An auto-update leaves builds failing with `iOS <version> Platform Not Installed`, even though builds succeeded that morning:

```bash
xcodebuild -downloadPlatform iOS   # ~8.5 GB, no sudo needed
xcodebuild -runFirstLaunch
```

## Housekeeping

EAS temp grows without bound — ours reached 35 GB under `/var/folders/.../eas-cli-nodejs`. A full disk fails the build with `No space left`. Clear it between releases. Build numbers skipping after failed attempts is normal.

Next: `04-app-store-submission.md`.
