# Android build and Google Play

Play is more forgiving than App Review, but it blocks releases on paperwork rather than on code — and the blocks are easy to hit.

## One-off setup

**Creating the app in Play Console is manual the first time.** There is no API path to it.

Two choices you cannot undo:
- **"Free" cannot become paid after publishing.**
- The package name is permanent, like an iOS bundle ID.

## Signing, and the SHA-1 that catches everyone

You sign with an **upload key**; Play re-signs with its own **app signing key**, which it generates only after your first AAB upload.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Keep the keystore out of the repo. Then:

**If you use Google Sign-In, you need BOTH SHA-1 fingerprints on the Android OAuth client** — your upload key *and* Play's app signing key (Play Console → App signing). Miss the second one and Google Sign-In is broken specifically in the Play build, while your local build works. It also cannot be tested on an emulator, because the debug SHA-1 is not registered either. Functional testing requires a Play-signed build on a device.

## Build

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # or your SDK path
eas build --platform android --profile production --local --output ./app.aab
```

Without `ANDROID_HOME`, Gradle reports "SDK location not found".

**Maps will crash without a key.** `react-native-maps` uses Google Maps on Android and **crashes at native init** if `app.json > android.config.googleMaps.apiKey` is missing. iOS is unaffected because Apple Maps is the default there — which is exactly why this ships unnoticed. Verify the key made it: unzip the AAB and check the manifest for `com.google.android.geo.API_KEY`.

## Upload

Manual drag-and-drop works but stays manual forever; a typical AAB is 60 MB+, past every browser automation limit. Automate it with a Play service account and `eas.json > submit.android`.

### Release errors you will see

- **"This release will not be available to existing users because it doesn't allow them to upgrade to the newly added app bundles."** → raise the version code, or publish through Internal/Closed testing first (recommended for a first release).
- **"This release adds or removes no app bundles."** → the AAB did not upload cleanly. Check the version code and re-upload.
- **Native debug symbols** must be a `native-debug-symbols.zip` containing ABI directories — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, each with `libapp.so` — and **no `__MACOSX` or `.DS_Store` entries**.

## Declarations that block the release

**Advertising ID.** Unzip the AAB and look for `com.google.android.gms.permission.AD_ID`. Firebase Analytics requires the permission and a matching "used" declaration; an app with no ads should have neither. **The rule is that the declaration must match the manifest exactly** — a mismatch in either direction locks the release, and Play's own warning text can be misleading about which side is wrong.

**Privacy policy URL.** It must return 200. Our first production submission was rejected purely because the declared URL 404'd — nothing else was wrong with the app.

**Data safety form and content rating questionnaire.** Both are required before production. Answer them from what the app actually does; they are re-checked against your declared permissions.

**Distribution countries.** Check them. One of our apps ran in production locked to a **single country** while iOS shipped to 175, which is not a state anyone chooses deliberately.

## Sensitive permissions

Background location and `FOREGROUND_SERVICE_LOCATION` trigger a Play permission declaration that requires a **demonstration video** and a review. If you do not need them yet, block them explicitly rather than shipping them and getting stuck:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Add them deliberately later, with the declaration and video prepared.

## Target API level deadlines

Play stops accepting updates for apps that miss the deadline for raising their target API level. The date moves every year. **Track it** — finding out on release day is a bad day.

## A note on Play's speed

Play approves fast, which cuts both ways: a broken release can be live within about an hour and **cannot be pulled back**. Ours shipped with a crash on the login screen; the only remedy was pushing a fixed version code and waiting. Use Internal testing first. Watch Play Vitals crash counts after a release — that is how we confirmed the fix landed (10 crashes → 0).
