# Android build, signing and testing (Expo)

Getting a correct AAB out of an Expo project and proving it works before Play sees it. The console side is in `03-google-play.md`.

Almost every Android-only failure we had came from one fact: **the `android/` folder and `app.json` disagree, and the `android/` folder wins.**

## Local toolchain (Apple Silicon)

```bash
brew install --cask android-commandlinetools
yes | sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0"
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
export ANDROID_SDK_ROOT=$ANDROID_HOME
export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

- Without `ANDROID_HOME`, Gradle reports "SDK location not found".
- Without the UTF-8 locale, Ruby 4 crashes `pod install` with an `unicode_normalize` `Encoding::CompatibilityError` in background shells, and fastlane warns. Put the exports in your build script, not just your shell profile.
- Homebrew's `avdmanager` can't see system images under `~/Library/Android/sdk` ("Package path is not valid"). Install `sdkmanager --sdk_root=$ANDROID_HOME "cmdline-tools;latest"` and call `$ANDROID_HOME/cmdline-tools/latest/bin/avdmanager` instead.
- macOS can revoke the agent's access to Desktop, Documents and Downloads mid-session ("Operation not permitted"), which kills builds. Re-grant Files and Folders (or Full Disk Access) and restart the app.

## Building

```bash
eas build --platform android --profile production --local --non-interactive --output ./app.aab
```

**Build locally.** The EAS Free plan has a monthly build quota **per platform**: we used up iOS while Android still worked, mostly on Android AABs nobody uploaded and on failed builds. `--local` builds are free and unlimited; `eas submit` doesn't count against the quota.

**Before queuing any build, run `npx expo export` locally.** A monorepo app imported runtime constants from a shared workspace package whose `dist/` isn't built on EAS. Both the Android and iOS builds failed with nothing but `UNKNOWN_ERROR ... See logs of the Bundle JavaScript build phase`. Keep shared imports `import type` only, or build the package as part of the app.

**Run `npx expo install --check` before the first Android build.** `@react-native-async-storage/async-storage` 3.x on SDK 55 broke the release build with `Could not find org.asyncstorage.shared_storage:storage-android:1.0.0`; 2.2.0 worked.

**`npm ci` must pass in a clean folder.** Dependencies installed with `--legacy-peer-deps` leave a lockfile that `npm ci` rejects, and the cloud build dies. Add `.npmrc` with `legacy-peer-deps=true` and regenerate the lockfile.

### `.easignore` replaces `.gitignore`, it doesn't extend it

The project archive was 187 MB and failed with `EPIPE` because a 68 MB AAB and store assets were in the folder. Adding just those to a new `.easignore` made it **2.9 GB** — `node_modules`, `.git` and `Pods` were no longer excluded. Copy all of `.gitignore` into `.easignore` first, then add `*.aab`, `*.ipa`, store assets and signing files. The working archive was 3.3 MB. Every failed attempt still consumed a build number.

### EAS logs come back compressed

`curl` on `logFiles[0]` from `eas build:list --json` returns bytes that `gunzip` rejects. Use `curl --compressed`, or open the build page on expo.dev.

## Prebuild: where app.json changes go to die

Once `android/` exists — after `expo run:android`, or any `expo prebuild` — builds use it as-is. EAS logs that it skipped prebuild. From then on, **app.json changes to versionCode, keys, permissions and package do not reach the build** unless you regenerate.

What that cost us:
- `android/app/build.gradle` still said versionCode 1 / 1.0.8 while `app.json` had moved on.
- **"Version code already used" twice in one week.** `app.json` was bumped, prebuild wasn't re-run. The build script only *mentioned* prebuild in a comment.

So the release script runs prebuild, every time:

```bash
rm -rf android && npx expo prebuild -p android   # a half-finished prebuild leaves a broken android/
grep versionCode android/app/build.gradle         # check before handing off the AAB
```

Also know your version source of truth. With `"cli": { "appVersionSource": "remote" }` in `eas.json`, EAS keeps its own counter and ignores `app.json`; set it with `eas build:version:set`. Cancelled and failed builds still consume a number on that counter.

`android.package` must be set in `app.json` or prebuild stops.

### Prebuild silently resets release signing to the debug key

Expo's template `build.gradle` has `release { signingConfig signingConfigs.debug }`. Hand-edit it to use your upload key, and the next `prebuild --clean` puts the debug key back — and deletes any keystore you kept inside `android/`. We recovered ours from an old copy of the project.

Fix it permanently:
- Keep the keystore and its properties file in a gitignored `credentials/` folder **outside** `android/`.
- Write a small config plugin (`withAppBuildGradle`) that adds `signingConfigs.release` pointing at `../credentials/keystore.properties`, sets the release build type to use it, and **throws if the patch doesn't apply.**

If you let EAS manage credentials instead, none of this applies — but back up the keystore it created (`eas credentials -p android`).

## Verify the AAB before it goes anywhere

Four checks, all scriptable, all from things that reached Play broken:

```bash
# 1. Which key signed it? jarsigner -verify says "jar verified" for a debug-signed bundle too.
keytool -printcert -jarfile app.aab | grep Owner:      # fail unless it's your upload key's CN

# 2. Version code
grep versionCode android/app/build.gradle

# 3. Permissions and meta-data (the manifest is protobuf, but strings still works)
unzip -p app.aab base/manifest/AndroidManifest.xml | strings | grep -iE 'permission|API_KEY'
#    - AD_ID present or absent, matching the Play declaration
#    - com.google.android.geo.API_KEY present if you use react-native-maps
#    - no ACCESS_BACKGROUND_LOCATION if you blocked it

# 4. Compare the signer with Play Console -> App integrity -> Upload key certificate
```

## Testing on an emulator

```bash
sdkmanager "emulator" "system-images;android-36;google_apis;arm64-v8a"
avdmanager create avd -n test -k "system-images;android-36;google_apis;arm64-v8a" -d pixel_7
emulator -avd test -no-snapshot-load -gpu swiftshader_indirect &
adb wait-for-device
until [ "$(adb shell getprop sys.boot_completed | tr -d '\r')" = 1 ]; do sleep 2; done
```

**Test the release build, not the dev build:**

```bash
cd android && ./gradlew installRelease
adb shell am start -n <package>/.MainActivity
adb logcat | grep FATAL
adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png
```

False alarms that cost us a session:
- `npx expo run:android --device emulator-5554` → "Could not find device". `--device` wants the **AVD name**; omit it with one emulator running.
- A debug build dies when Metro stops. That's not a crash — release builds embed the JS.
- A low-RAM emulator killed the app with **no FATAL in logcat**. Give it more memory before blaming the code.
- Push tokens are never issued on an emulator, and Expo Go (SDK 53+) has no remote push. Expected.
- Google Sign-In fails unless the debug keystore's SHA-1 is registered too. Functional testing of sign-in needs a Play-signed build on a real device.

**What the first emulator run caught in a release build** — bugs iOS testing never shows:
- **Android autocorrect rewrote a typed slug** into two dictionary words. Set `autoCorrect={false}` and `autoCapitalize="none"` on identifiers, and normalise server-side.
- **The hardware back button closed the app mid-form** and lost the input. Add a `BackHandler` with a confirmation.
- **`SafeAreaView` from `react-native` does nothing on Android.** Content slid under the status bar on 33 screens. Use `react-native-safe-area-context` with a root `SafeAreaProvider` and `edges={['top']}`.

## Blocking a permission isn't enough

`blockedPermissions` removes background location from the manifest (see `03-google-play.md`), but the code still asked for it. On targetSdk 36 the app showed a pointless permission prompt and then threw when it started the background task. Gate `requestBackgroundPermissionsAsync` and `startLocationUpdatesAsync` behind `Platform.OS === 'ios'`, and tell Android users that tracking only runs with the screen on.

## Google Maps

`react-native-maps` uses Google Maps on Android and **crashes at native init** if `app.json > android.config.googleMaps.apiKey` is missing. iOS is unaffected because Apple Maps is the default there — which is exactly why this ships unnoticed. It shipped for us as a crash right after login: every tab mounted at once, so a `MapView` appeared the moment the user signed in.

- Check the key is in the AAB (manifest check above).
- **Lazy-mount tabs**, and render a placeholder instead of `MapView` when the key is missing. A missing key then becomes a grey box, not a crash.
- Restrict the key to the package plus **both** SHA-1s (upload and Play signing). **Restrict only after the unrestricted key is verified working**, so a failure points at one cause.

## Google Sign-In on Android

This was our largest single dead end: one review cycle and one wasted release.

**The browser OAuth flow does not work with an Android OAuth client.** `expo-auth-session` / `Google.useIdTokenAuthRequest` with the Android client ID returned `Error 400: invalid_request` for every redirect variant we tried (`<pkg>:/oauthredirect`, `<pkg>://`, the reversed client ID). Android clients are only for Play Services and Credential Manager.

Diagnosis trick worth keeping: request `https://accounts.google.com/o/oauth2/v2/auth` with each client ID. A made-up ID gives `invalid_client`; the Android client gives `invalid_request`; iOS and web are accepted. So the client existed — the *request type* was wrong.

What works:
- Native `@react-native-google-signin/google-signin`, configured with the **web** client ID. The returned ID token's audience is the web client.
- **Two Android OAuth clients**, both with the package name: one with the upload-key SHA-1 (local builds), one with the Play app-signing SHA-1 (store installs). With only the upload key registered, the store build gave the same 400.
- **The backend's token audience check must accept the web, iOS and Android client IDs**, or the server rejects tokens the app happily obtained.
- OAuth consent screen set to **In production**. Sticking to the default email/profile scopes avoids the unverified-app warning and the 100-user cap.
- None of this works in Expo Go; it is native code.

## Push notifications (FCM)

On one app, Android push was broken in three places at once:
1. The Firebase project only had a **web** app. Add an Android app and ship its `google-services.json`.
2. The client returned early on `Platform.OS !== 'ios'` and never registered.
3. The backend only queried tokens where `provider = 'apns'`.

Also:
- **Create a notification channel.** Android 8+ silently drops notifications without one.
- The legacy FCM server key is gone. Use **FCM HTTP v1**: service-account JWT → OAuth2 access token → send.
- Smoke test for the server side: sending to a fake token should return `INVALID_ARGUMENT`. If you get an auth error, your credentials are wrong; `INVALID_ARGUMENT` proves auth works.
- With Expo push, upload the FCM credentials to EAS, and test in a real build — never in Expo Go.

## Wrapped websites (TWA, Capacitor with a remote URL)

- **TWA:** serve `/.well-known/assetlinks.json` with the **Play app-signing SHA-256**, not only the upload key's — otherwise the app shows a browser URL bar. Serving it from environment variables (and `[]` until set) keeps it out of the code. Two traps on the way: Apache's `mod_headers` was disabled, so every header rule in `.htaccess` was silently ignored; and Cloudflare's browser-cache override beat the site's own headers.
- **An app that only loads a remote URL** is a minimum-functionality risk on Play as well as on Apple. Give it at least one real native feature (push, share target, offline) before submitting.
