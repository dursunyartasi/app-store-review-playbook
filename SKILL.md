---
name: app-store-review-playbook
description: App Store and Google Play submission playbook - a checklist distilled from real rejections, plus build and upload traps with their fixes. Use when - "submit to App Store", "upload to Play", "app was rejected", "Apple rejection", "Guideline", "App Review", "TestFlight", "upload AAB", "eas submit", "altool", "provisioning profile", "store listing", "review notes", "privacy declaration", "age rating". Read BEFORE submitting a new mobile app.
metadata:
  version: 1.0.0
  source: 8 shipped iOS/Android apps, 2026
---

# App Store & Play review playbook

Not generic store advice. Every item below is a rejection that actually happened while shipping eight apps, with the root cause we eventually found. Apps are anonymised as **App A** (social events), **App B** (map/venue guide), **App C** (B2B agency tool).

**How to use:** run the pre-submission checklist before you submit. If you got rejected, find the match in the rejection catalogue. If a build or upload is stuck, check the traps section.

---

## The most expensive lesson — what NOT to write in App Review Notes

App A build 49 was rejected under **Guideline 4.2 (Minimum Functionality)**. Apple's rejection text quoted, almost word for word, the sentences we had written ourselves in App Review Notes:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple read that as "this belongs in Ad Hoc distribution, not the App Store."

**Rule: never frame your app this way in review notes** — "small", "niche", "few dozen", "invite-only", "not mass-market", "private group", "closed community", "for a specific community".

**Correct framing:** "open to everyone, free, downloadable from any city or country; the curated membership tier is an OPTIONAL layer." Even if the product really is invite-only, it needs a face that works **without an account**, and the notes should describe that face.

Also: **App Store Connect review notes are capped at 4000 characters.** Exceed it and the API returns 409.

---

## Pre-submission checklist

Every box below comes from a rejection. Verify them one by one.

### Accounts and data (5.1.1)
- [ ] **Account deletion.** If users can create an account, they must be able to delete it from inside the app — 5.1.1(v). *(Caused rejections in two of our apps.)* The flow Apple finally accepted met all of these, and each one matters:
  - **Immediate and permanent.** No deactivation, no cooling-off period.
  - **No support/email/phone step and no redirect to the web** — Apple treats those as "not deletable from inside the app".
  - Password re-authentication plus a destructive confirmation warning.
  - An explicit list of what gets deleted; third-party grants (Instagram/Facebook) revoked on the platform side too.
- [ ] **Permission purpose strings.** — 5.1.1(ii). **Bare-workflow trap:** if the `ios/` directory is committed, purpose strings in `app.json` do **not** propagate to `Info.plist` automatically. Check `Info.plist` by hand.
- [ ] **Don't request permissions you don't need.** App B was rejected under 5.1.1 because `pickImage` called for full photo-library permission before opening the picker. Modern iOS PHPicker returns a single photo with **no permission at all** → delete the permission call.
- [ ] **Are the legal links tappable?** — 2.1(a). App C was rejected here: the "by creating an account you agree to the Terms" sentence on the signup screen was **plain text**, and the login screen had no links at all, so the reviewer could not view the terms. Open them in an **in-app browser** (`expo-web-browser`) rather than kicking the user out to Safari, and put them on the login screen too.
- [ ] **Location priming screen** — 5.1.1(iv). No leading button copy: "Use My Location" ❌ → "Continue" ✅. Don't ship two confusing "Skip" affordances.

### User-generated content (1.2) — mandatory if users can post anything
App A build 51 was rejected here. You need all four:
- [ ] A **visible** reporting affordance. A long-press gesture is **not enough** — the reviewer will not find it. Put a visible "⋯" next to every message, post and comment.
- [ ] Blocking (two-way — a blocked user cannot write to you either).
- [ ] Content filtering on **every** write endpoint, not just the obvious ones.
- [ ] DM consent: the initiator can send one message until the recipient accepts.

### Purchase signals (3.1.1) — the nastiest trap for free and B2B apps
App C was rejected under this guideline **twice**.
- [ ] **Leave no price, plan name, credit counter, paywall, upgrade button or outbound purchase link in the app.** What sank build 25 was a line reading "Intelligence isn't in your current plan — Solo or above required", a remaining credit counter, and a "1 credit per brand" label. Merely displaying a plan name is enough.
- [ ] **Don't lean on the 3.1.3(f) "Free Stand-alone Apps" argument alone — Apple rejected it.** We tried it on build 26.
- [ ] **A public signup screen destroys a B2B claim.** That was the weak link: a "join" screen reads as consumer self-service to a reviewer and directly contradicts 3.1.3(c)'s *"only sold directly by you to organizations"*. The fix in build 27 was **deleting the signup screen entirely** — the app offers sign-in only.

### Metadata
- [ ] **Age rating** — 2.3.6. An app with any "meeting people / networking" angle needs `matureOrSuggestiveThemes` at least `INFREQUENT_OR_MILD`. Fixable over the API: `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **Does the privacy policy URL return 200?** Our first Play production submission was rejected purely because the declared URL 404'd.

### What the reviewer will actually see
- [ ] **Does the demo account really work?** Test it on a device. App A was rejected under 2.1 because sign-in from the Apple Watch target had never worked — it posted `email` while the backend read `identifier`, returning 422. Nobody noticed because the phone target sent the right field.
- [ ] **Is the demo data fresh?** In App A, 16 of the seeded events were in the past — the reviewer would have opened an empty app. Keep an idempotent script that rolls demo dates forward.
- [ ] **Does a verification wall trap the reviewer?** If a signed-up but unverified user sees nothing, the app looks "closed". Let guests browse; require verification only on write actions.
- [ ] **Ship no empty or disabled modules.** A feature flag left off in App A produced an empty "Courses" section that caused two consecutive 2.1 rejections (App Completeness, then Information Needed). It was eventually deleted. **Don't submit a half-feature — remove it.**
- [ ] **You don't get to pick the review device.** Ours came back on an iPad Air and an Apple Watch. Test targets beyond your primary device.

### Platform integration (4.0 Design)
- [ ] **Is your map/location feature integrated with the native app?** App B was rejected under 4.0.0 for only handing users off to Google Maps. Offer Apple Maps (`maps.apple.com`) as an option.

### Android
- [ ] **Does the advertising-ID declaration match the manifest?** Unzip the `.aab` and check for `com.google.android.gms.permission.AD_ID`. If it isn't there, the declaration must say "not used" — a wrong declaration blocks the release.
- [ ] **Distribution countries.** One of our apps was accidentally locked to a **single country** in production while iOS shipped to 175. Production → Countries/regions.
- [ ] **Background location** may trigger Play's Location Permission Declaration.
- [ ] **Maps API key** in `app.json > android.config.googleMaps.apiKey` — without it `react-native-maps` **crashes at native init on Android**. iOS is fine (Apple Maps is the default there), which is exactly why this slips through.
- [ ] **Google Sign-In needs TWO SHA-1s:** your upload key **and** the Play app-signing key. Play generates its signing key only after the first AAB upload; if that SHA-1 isn't added to the Android OAuth client, Google Sign-In is broken in the Play build. It also can't be tested on an emulator (the debug SHA-1 isn't registered) — functional testing requires a Play-signed build on a device.

---

## Rejection catalogue

| Guideline | What Apple said | Actual root cause | Fix |
|---|---|---|---|
| **4.2** Minimum Functionality | "small, or niche, set of users" | **Our own sentence in the review notes** | No-account discovery flow + public API endpoints + content rebalance |
| **1.2** UGC | no filtering / reporting / blocking | Reporting existed only behind an invisible long-press; absent entirely in DMs and lobbies | Visible "⋯" menu across 8 surfaces, filter on 9 write endpoints, DM consent |
| **2.1** Demo account | couldn't sign in | Watch target posted `email`, backend read `identifier` → 422 | Fixed the field; added a "WATCH — PLEASE READ FIRST" section to the notes |
| **2.1** App Completeness | "could not access the courses" | Feature flag off; the section rendered empty | Feature removed entirely |
| **2.1** Information Needed | "how many users are you targeting?" | Same empty module + the 4.2 framing | Notes rewritten |
| **2.3.6** Age rating | "Mature or Suggestive Themes" | Meeting/networking theme not declared | `PATCH ageRatingDeclarations` over the ASC API |
| **5.1.1(v)** Data Collection | no account deletion | — | In-app account deletion |
| **5.1.1(ii)** | missing purpose string | Bare workflow doesn't sync `app.json` → `Info.plist` | Edit `Info.plist` directly |
| **5.1.1(iv)** Location flow | leading priming screen | Button copy + duplicate skip | "Continue", single exit |
| **5.1.1** Photo access | requesting library permission | PHPicker needed no permission | Permission call deleted |
| **4.0.0** Design | "not integrated with built-in mapping" | Google-Maps-only handoff | Apple Maps option added |
| **3.1.1** IAP | purchase signals in a free app | Plan name, credit counter, "Solo plan required" copy | Every price and plan trace removed |
| **3.1.1** IAP (second time) | same guideline again | 3.1.3(f) argument wasn't enough; a **public signup screen** contradicted the B2B claim | Signup screen deleted, sign-in only |
| **2.1(a)** App Completeness | couldn't view the terms | Legal text was plain, untappable; absent from the login screen | Tappable links opening in an in-app browser |
| **Play** (production) | submission rejected | Declared privacy-policy URL returned 404 | Permanent alias + corrected console entry |

**Note:** fixing one rejection can invite the next. One app went through four consecutive rejections, another three. After every fix, re-run the **entire** checklist.

---

## App Store Connect API and declaration traps

Found while filling submission fields over the API:

- **There is no `APP_IPHONE_69` screenshot type.** The largest iPhone type the API accepts is `APP_IPHONE_67` (1290×2796). Images rendered at 1320×2868 for the 6.9" device are **rejected** — upload 6.7" and let Apple scale up.
- **`whatsNew` cannot be edited on a first version** — 409, "cannot be edited at this time". It only exists for updates.
- **Age-rating field types are mixed:** some are BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), others are STRING enums (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). The wrong type returns 409, and the error message names the correct set.
- **Apple changed the age bands in 2025: 12+ no longer exists.** The bands are 4+, 9+, 13+, 16+, 18+. Honest answers can yield 4+; raise it with `ageRatingOverrideV2` (e.g. `THIRTEEN_PLUS`).

**App Privacy declaration:**
- **A national ID number is not "Sensitive Info".** Apple's sensitive category covers race, religion, sexual orientation, biometrics and the like — a national ID isn't on that list, so **"Other Data Types"** is the correct bucket.
- **If you store bank details (IBAN) in your own database, declare them Collected.** Apple only exempts you when the payment provider holds them and you cannot access them.
- ⚠️ **Blind-clicking the wizard produces wrong answers.** The declaration flow renders at different heights per data type; clicking the same coordinates repeatedly produced answers like "User ID is used for tracking: YES" that were simply false. Screenshot and verify the final state of every item.

## Build and upload traps

### The version train
**You cannot upload a new build against an already-approved version string** — altool errors 90062 / 90186 ("Invalid Pre-Release Train ... closed"). Bump `version` in `app.json` and **rebuild**; the version string is baked into the IPA. We burned a whole build learning this.

### Uploading
- `eas submit` can hang (23+ minutes, no output) or fail with "Failed to authenticate for session". **The reliable path is altool directly:**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  Put the `.p8` at `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` — altool reads it from there. Finishes in about 15 seconds.
- **"Upload succeeded" ≠ accepted.** Apple can still reject during processing. Poll for `VALID`, and once valid, expire the previous build (`PATCH /v1/builds/{id}` with `{"expired": true}`).
- **Watch and widget targets need an icon** (`CFBundleIconName`) or Apple refuses the upload with error **90713**.
- **ITMS-90062** means "this version is already published" — bump the version string.
- **ITMS-90863** (Apple Silicon symbol warning) is **normal for Expo apps and does not cause a rejection.** Don't chase it.

### Resubmission order
1. **Two versions cannot be in review at once.** Cancel the existing `reviewSubmission` first (`canceled=true`) and wait for CANCELING → COMPLETE.
2. The version becomes `DEVELOPER_REJECTED` (editable) → PATCH the version string → PATCH the build relationship.
3. ⚠️ **Swap trap:** immediately after cancelling, attaching the build returns 409 — and if your script carries on regardless, it submits the **old** build. Retry the attach and **verify the attached build before submitting** (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `POST reviewSubmissionItems` can return 409 `ENTITY_STATE_INVALID` while the state transition finishes. It succeeds a few seconds later — make that step retryable.

### Local build environment
- **If Xcode auto-updates mid-session**, builds fail with "iOS X Platform Not Installed". Fix: `xcodebuild -downloadPlatform iOS` (~8.5 GB, no sudo) plus `xcodebuild -runFirstLaunch`. Builds that succeeded earlier the same day are no evidence the environment is still good.
- **CocoaPods on Ruby 4.0:** `pod install` dies with `Unicode Normalization not appropriate for ASCII-8BIT`. Run it with `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8`.
- **Podfile modular headers:** GoogleSignIn 9.2.0 needs `:modular_headers => true` for `AppCheckCore`, `GoogleUtilities` and `RecaptchaInterop`.
- **A provisioning profile that predates a new capability** fails the local build. Non-interactive EAS will not update credentials — either refresh interactively or go through the ASC API.
- **Apple Developer capabilities can be enabled over the API** (no portal visit): `POST /v1/bundleIdCapabilities`. Without a `settings` payload it returns 409.
- **`ANDROID_HOME` is required** for local Android builds, otherwise Gradle reports "SDK location not found".
- **Never edit source files while an archive is compiling** — Metro embeds a half-written bundle and the app crashes on launch.
- **EAS temp grows without bound** (ours reached 35 GB under `/var/folders/.../eas-cli-nodejs`). Clean it after builds; a full disk fails the build with "No space left".
- Build numbers skip when attempts fail. That's normal.

### Crashes that only happen in standalone builds
- **Simulators and dev clients do not catch these.** Test on a real device over cable with `devicectl --console`.
- If `.env` is gitignored it never reaches the EAS archive → empty variables in the bundle → crash on launch. In one app *every* build crashed for this reason.
- A dynamically `import()`ed native module that isn't installed is invisible in dev (Metro serves it) and crashes standalone with `RCTFatalException: Cannot find module`.
- **Hermes stores strings as UTF-16.** Grepping the bundle for non-ASCII strings as UTF-8 returns nothing — verify in UTF-16.

---

### Play release errors
- **"This release will not be available to existing users because it doesn't allow them to upgrade to the newly added app bundles."** → bump the version code, or better, publish through Internal/Closed testing first.
- **"This release adds or removes no app bundles."** → the AAB didn't upload cleanly; check the version code and re-upload.
- **Native debug symbols** must be a `native-debug-symbols.zip` with ABI directories (`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, each containing `libapp.so`) and no `__MACOSX` or `.DS_Store` entries.
- ⚠️ **Target API level deadlines.** Play blocks publishing updates for apps that miss the deadline for raising their target API level. Track the date — release day is a bad time to find out.
- **The AD_ID nuance:** Firebase Analytics requires the permission in the manifest and a matching "used" declaration; an app with no ads should have neither. **The rule is that the declaration must match the manifest exactly** — a mismatch in either direction blocks the release.

## Store registration — one-off, and manual

- **You cannot create the App Store Connect app record over the API.** We tried and confirmed it. Do it in the browser.
- **Creating the app in Play Console is also manual** the first time.
- **The bundle ID is bound permanently to the record and cannot be changed.**
- **Choosing "Free" in Play is irreversible** — you cannot switch to paid after publishing.
- ⚠️ **Non-ASCII characters can be dropped at registration.** On an individual Apple account, the developer name shown on the App Store is your legal name — ours lost its Turkish diacritics during signup. Fixing it through App Store Connect → Business → Legal Entity **does not work**: that flow pulls you into address verification and the Paid Apps Agreement chain, and the name alone won't save. The working route is Apple Support → "Membership & Account" → legal-name correction, which is identity-verified. **Check the spelling character by character while registering.**
- In Play Console the real path for in-app content pages is `app-content/**` (e.g. `app-content/ad-id-declaration`); `/app-content` alone redirects to the app list.

## Boundaries for an AI assistant running this playbook

- **Never type the developer's Apple or Google password or 2FA code.** App Store Connect requires its own sign-in (the Developer session doesn't carry over). The workflow is: the human signs in, says so, and the assistant drives the API and console steps from there.
- **Browser file upload is capped at 10 MB;** a typical `.aab` is 60 MB+. The human uploads it, or you automate it with a Play service account and `eas.json > submit.android`.
- **Never tick declaration or consent checkboxes** without the human's explicit approval.

---

When a new rejection arrives, find the root cause first, then add a row here. A playbook is only worth what the last incident taught it.
