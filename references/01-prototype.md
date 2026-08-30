# From nothing to a running prototype

Goal: an app running on the user's own phone, built so that the store requirements in `05-rejections.md` are already satisfied rather than retrofitted.

## Decisions to make before the first file

**Managed or bare workflow.** Managed keeps `ios/` and `android/` generated; bare commits them. Bare gives you native control and costs you this: **`app.json` stops being the source of truth.** Permission purpose strings no longer propagate to `Info.plist`, and the version comes from `Info.plist` `CFBundleShortVersionString` plus the pbxproj `MARKETING_VERSION`. Both have caused rejections. Start managed unless a native module forces otherwise.

**Bundle identifier.** Pick it now and be certain — once an App Store Connect record exists, **the bundle ID is permanent.** Use reverse-DNS on a domain you control.

**The name Apple will display.** On an individual Apple Developer account, the developer name shown on the App Store is your legal name as registered. Non-ASCII characters can be silently dropped at signup (ours lost its Turkish diacritics), and the App Store Connect self-service fix does not work — it drags you into address verification and the Paid Apps Agreement chain and never saves the name. Fixing it means an identity-verified support request. **Check the spelling character by character while registering.**

## Scaffold

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # scan the QR with Expo Go
```

Expo Go is enough until you add a native module or need a signed build. After that you need a development build or a real archive.

## Wire in the store requirements now

These are cheap on day one and expensive after a rejection.

**If users can sign in — account deletion (5.1.1(v)).** It must be reachable inside the app, immediate and permanent. No deactivation, no cooling-off period, no "email support to delete", no redirect to a website. Ask for the password again, show a destructive confirmation, list what will be deleted, and revoke third-party grants on the provider's side too.

**If users can post anything — the four 1.2 requirements.** A visible affordance on every message, post and comment (a "⋯" button; a long-press gesture is invisible to a reviewer and was rejected), blocking that works in both directions, content filtering on **every** write endpoint, and a consent step before a stranger can send more than one direct message.

**Legal links must be tappable (2.1(a)).** "By signing up you agree to the Terms" as plain text is a rejection. Make them real links, open them in an in-app browser rather than throwing the user out to Safari, and put them on the login screen too, not just signup.

**Permissions.** Ask for nothing you do not use. Requesting full photo-library access before opening the picker was a rejection — the modern iOS picker returns one photo with no permission at all. Priming screens must not use leading button copy: "Continue", not "Use My Location".

**A contact address that actually receives mail.** If you publish one in your listing or in-app rules, the domain needs an MX record. See the MX trap in `06-stack.md` — ours could send but not receive, so the moderation address in our published rules reached nobody.

## Environment variables

```
.env            → committed to .gitignore, as usual
.easignore      → THIS is what EAS reads, and it replaces .gitignore
```

**An ignored `.env` never reaches the EAS archive.** The bundle ships with empty variables and the app crashes on launch — in standalone builds only, so the simulator and dev client both look fine. In one of our apps every single build crashed for this reason before we found it. Either configure EAS environment variables, or make sure `.easignore` does not exclude what the build needs.

## Get it onto a real device

A simulator does not prove the app works. Standalone-only crashes are the class of bug that reaches reviewers:

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # catches import errors early
```

Then build and install over cable, and watch the log with `devicectl --console`. A dynamically `import()`ed native module that is not installed is invisible in development — Metro serves it — and crashes standalone with `RCTFatalException: Cannot find module`.

## Before you go further

Run `npx tsc --noEmit` and your tests, and make them clean. Every build cycle past this point costs 5–40 minutes and, once you are in review, days.

Next: `02-testflight-ios.md` or `03-google-play.md`.
