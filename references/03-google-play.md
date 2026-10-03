# Google Play Console

Play is more forgiving than App Review, but it blocks releases on paperwork rather than on code — and the blocks are easy to hit. This file is the console side. Building, signing and testing the AAB is in `07-android-build.md`.

## Day 1: create the app and upload *something*

The most expensive Android mistake we made was not a rejection. It was **sixteen production AABs built for an app that never existed in Play Console.** Every report said "ready for manual upload"; iOS moved through TestFlight; the Android track stayed at zero for days.

So on day 1, before polishing anything:

1. **Create the app in Play Console by hand.** There is no API path to it.
2. **Upload the first AAB to Internal testing.** That creates Play's app signing key, gives you the second SHA-1 you will need (see below), and unlocks `eas submit -p android`.
3. Until that is done, **stop cutting Android production builds.** They burn build quota and pile up on the Desktop.

Two choices you cannot undo:
- **"Free" cannot become paid after publishing.**
- The package name is permanent, like an iOS bundle ID.

### The closed-testing requirement — check which rule applies to you

**Personal developer accounts created after 13 November 2023** must run a closed test with **at least 12 testers opted in for 14 continuous days** before they can even apply for production access. (It used to be 20; we once quoted the old number to a user. It is 12.) The 14 days only start once a closed-test release is actually live, which is one more reason to upload on day 1.

**Older accounts and organisation accounts are exempt.** Ours dated from 2016 and went straight to production with every app. Check the account's creation date before planning a two-week delay that doesn't exist — or before promising a launch date that can't happen.

### Developer verification

From 2026 Google requires developers to verify their identity and register package names. Already-published packages showed as "Registered" on the identity tab. **Check it for every new package name**, and act on the reminder emails rather than filing them.

## Signing, and the SHA-1 that catches everyone

You sign with an **upload key**; Play re-signs with its own **app signing key**, which it generates only after your first AAB upload. Both fingerprints are on Play Console → Test and release → App integrity (URL: `.../app/<id>/keymanagement`).

**Whatever needs a fingerprint needs BOTH** — the Android OAuth client for Google Sign-In, the Google Maps API key restriction, `assetlinks.json` for App Links or a TWA. Miss Play's key and the feature is broken *specifically in the store build* while your local build works.

- **Copy fingerprints with the console's copy buttons.** The page lists SHA-1 and SHA-256 for two keys; we nearly pasted a SHA-256 into a SHA-1 field. One wrong character fails silently.
- **The upload key is a permanent identity.** An agent must not generate one without the user's explicit OK — and must not let one be generated silently. EAS does exactly that: a first `eas build -p android --non-interactive` prints `Created keystore` and stores it on Expo's servers, with no local copy. Back it up immediately with `eas credentials -p android`.
- Keep the keystore **outside the project folder**. We found one sitting in a repo root next to an iOS `.p12` and provisioning profile — one hand-written `.easignore` away from shipping in a build archive.

Before every upload, confirm which key actually signed the AAB (`07-android-build.md` has the check). A debug-signed bundle passes `jarsigner -verify` happily.

## Uploading

### An AI agent cannot upload the AAB through the browser

The browser automation file-upload limit is 10 MB; a typical AAB is 55–100 MB. Driving the macOS file picker with AppleScript needs Accessibility permission and failed with `-25211`. Every release of three apps needed a human round-trip until we stopped fighting it.

**The handoff that works:**
1. Build and verify the AAB.
2. Copy it to `~/Desktop/<app>-<version>-vc<N>.aab` and run `open -R` on it so Finder shows the file.
3. Open the release page in Play Console.
4. The user drags the file in; the agent continues with release notes and submission.

If Play Console stops responding to automation (the page never goes idle and every click times out), **stop retrying and hand off.** We lost a session to that.

**The real fix is a Play service account** and `eas.json > submit.android` (`serviceAccountKeyPath`, `track: "internal"`). Set it up once the app exists; after that, `eas submit -p android` replaces the whole dance. We proposed it on every release for two months and never did it — don't repeat that.

### Release errors you will see

- **"Version code N has already been used."** A version code is consumed the moment *any* upload reaches Play — including one in a draft you never published. Bump and rebuild. If you did bump it, the native project probably didn't pick it up: see the prebuild section of `07-android-build.md`. This hit us twice in one week.
- **"This release will not be available to existing users because it doesn't allow them to upgrade to the newly added app bundles."** → raise the version code, or publish through Internal/Closed testing first.
- **"This release adds or removes no app bundles."** → the AAB did not attach. It may be sitting in App bundle explorer without being in the draft; use **Add from library** in the release instead of uploading again.
- **Native debug symbols** must be a `native-debug-symbols.zip` containing ABI directories — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, each with `libapp.so` — and **no `__MACOSX` or `.DS_Store` entries**.
- **"No deobfuscation file"** is a warning, not a blocker, when R8 is off (the Expo default).

### Release notes

- **Wrap them in language tags** — `<tr-TR>…</tr-TR>`, `<en-US>…</en-US>` — one block per listing language, or the field turns red.
- **500 characters per language.** Our first draft was rejected for length.
- Filling the field via JavaScript or form automation can HTML-escape the tags into `&lt;tr-TR&gt;`. Set the raw value and look at it before saving.

## App content: every declaration that blocked us

Entry point: `.../app/<id>/app-content/overview`. Deep links to individual declaration pages are unreliable; start from the overview. Answer everything from what the app *actually does* — answers are cross-checked against your manifest. **An agent must not tick these for the user**; they are legal statements about their app.

| Declaration | What tripped us |
|---|---|
| **Privacy policy** | The URL must return 200. Our first production submission was rejected purely because it 404'd. |
| **App access** | See the demo-account section below. |
| **Ads** | Separate from the advertising-ID declaration. Both exist. |
| **Content rating** | Questionnaire; quick, but required before production. |
| **Target audience** | We picked 18+ to stay out of the Families policy. Choosing an under-13 band pulls in a whole extra review. |
| **Data safety** | Use CSV import (below). **Requires a public account-deletion URL** if the app has accounts — separate from in-app deletion, and Google fetches it: an undeployed page shows up as a 404. A partial-deletion URL is asked for too. |
| **Advertising ID** | Must match the manifest exactly. See below — this one cost us the most. |
| **Government apps, Financial features, Health** | Each needs an explicit "none" if it doesn't apply. Storing an IBAN with no money movement was "no financial features". A step counter is a **Health** declaration ("Activity and fitness"). |
| **Store listing** | AI-generated listing images must be declared. Contact details in Store settings are **public** — we left the phone number blank deliberately. |
| **Content rights, category, price** | Submission was blocked with "this resource can't be reviewed" until primary category, content rights, price (free) and countries were all set. |
| **Countries / regions** | Set at app level **and** on the production track's own Countries tab, which starts empty. One of our apps ran in production in a single country while iOS shipped to 175. If installs look like zero, check this before anything else. |

### Data safety: import, don't click

The click-through is about 50 dialogs and repeatedly lost selections ("0/9 saved"). Instead:
1. Export the CSV template from the Data safety page.
2. Fill it with a script, using **the exact keys from the export** — invented names fail. Real ones looked like `PSL_DEVICE_ID`, `PSL_FILES_AND_DOCS`, `PSL_OTHER_MESSAGES`. A push token went under "Device or other IDs".
3. Import — then click the separate **Import** confirmation. We missed it the first time.
4. Export again and diff, to prove it saved.

### The advertising-ID blocker

Unzip the AAB and look for `com.google.android.gms.permission.AD_ID` (command in `07-android-build.md`). Firebase Analytics pulls the permission in and needs a matching "used" declaration; an app with no ads and no analytics SDK should have neither. **The declaration must match the manifest exactly** — a mismatch in either direction locks the release, and Play's own warning text can be misleading about which side is wrong.

**Get it right in the very first submission.** On one app Play kept showing "Advertising ID declaration missing — affects all your changes" with the send-for-review button locked, while the declaration page showed "No" saved and the to-do list was empty. We spent roughly fifteen hours across several days on it:
- waiting out pre-checks (they take up to ~12 minutes) — no change;
- toggling Yes → No to re-enable Save — no change;
- uploading a new AAB — no change;
- checking the separate Ads declaration — no change.

Two releases even went out while the warning was showing. It ended as an open support ticket. **Do not flip it to "Yes" to unblock** — "Yes" forces you to pick a purpose (functionality, analytics, advertising), and that is a false declaration.

### Sensitive permissions

Background location and `FOREGROUND_SERVICE_LOCATION` trigger a permission declaration that needs a **demonstration video** and a review. A walking app built with them got three blockers at once:

- "This release contains permissions that haven't been declared in Play Console"
- "You need to tell us whether your app uses foreground service permissions"
- "You need to complete the Health declaration form"

If you don't need background location yet, block it and rebuild:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

The foreground-service warning cleared on its own; the Health form still had to be filled because of the step counter. **The code must match too** — see "Blocking a permission isn't enough" in `07-android-build.md`.

## Review

### The demo account: 2FA got us rejected

Play rejected a production submission because **"multi-factor authentication blocks access"**. The review account had 2FA enabled, and the reviewer hit an emailed six-digit-code screen. The App access instructions said "No 2FA" — copied from an earlier draft, never checked against the database flag.

The same rejection's evidence screenshot showed the Google sign-in button failing with a 400. **Reviewers try social login too**, not just the credentials you give them.

Before submitting, **sign in with the demo account from a clean device, using a real request**, and confirm what the reviewer will see.

**App access form behaviour:**
- Add the credentials row with **Add** before **Save**, or you get a leave-page warning and lose it.
- Instructions: **English, 500 characters max.**
- After a rejection the credentials table can *look* empty while the entry still exists — creating a new one fails with "name already used". Edit the existing one.
- The password field is the user's to type. An agent does not enter it.

### Timing we actually saw

| Event | Time |
|---|---|
| Internal testing | instant, no review |
| First production review | several days (one took four and ended in rejection) |
| Updates after that | about 1 hour to a few hours |
| Store-listing changes | 1–3 days; the live app stays up meanwhile |

- **Submitting new changes while a review is pending resets your place in the queue.**
- Resubmitting after a rejection shows no "review will reset" warning, because there's no place left to lose.

### Play's speed cuts both ways

A broken release can be live within an hour and **cannot be pulled back**. One of ours shipped with a crash right after login; the only remedy was a new version code and another wait. **Use Internal testing first.**

## After release

**Play Vitals gives you the real stack trace without a phone on a cable.** That login crash was proven from Vitals — `IllegalStateException: API key not found ... com.rnmaps.maps.MapView.onCreate` — and Vitals is also how we confirmed the fix (10 crashes → 0).

**Pre-launch report and Vitals warnings** we saw, none blocking:
- Memory usage flagged as "bad behaviour" with a deadline months out. Missing it hurts discoverability; it doesn't remove the app.
- Code shrinking at 2% because R8 is off. Turning it on cuts AAB size, but needs a full test pass.
- Edge-to-edge deprecated APIs on Android 15. An Expo SDK upgrade fixes most of it.
- Portrait-only / large-screen orientation restrictions.
- "4 deep links may fail" — web domains with no `assetlinks.json`.

**"0+ downloads" is not "not published."** One user read it that way. Check countries first.

## Target API level deadlines

Play stops accepting updates for apps that miss the deadline for raising their target API level. The date moves every year. **Track it** — finding out on release day is a bad day.
