---
name: mobile-app-shipping
description: End-to-end guide for building and shipping a mobile app to the App Store and Google Play, written from real rejections and real build failures. Use when the user wants to start a mobile app prototype, set up Expo/React Native, build an IPA or AAB, upload to TestFlight, upload to Google Play, submit for App Store review, write App Review notes, fill store declarations, or fix a store rejection. Also triggers on - "app rejected", "Guideline", "App Review", "TestFlight", "eas build", "eas submit", "altool", "provisioning profile", "AAB", "Play Console", "App Store Connect", "age rating", "privacy declaration", "app store screenshots".
metadata:
  version: 2.0.0
  source: 8 shipped iOS/Android apps, 2026
---

# Shipping a mobile app

Everything here comes from apps that actually shipped — and from the rejections and build failures encountered on the way. Nothing is paraphrased from official docs.

## Start by finding out where the user is

Ask only what you still need. If the user's message already answers a question, skip it. **Never ask more than five.**

1. **What do you want to do right now?**
   `new prototype` · `build and get it on my phone` · `TestFlight` · `Google Play` · `submit for App Store review` · `I got rejected`
2. **Which platforms?** iOS, Android, or both.
3. **Do users sign in or create content in the app?** (accounts, posts, comments, messages, photos)
4. **Do you have the developer accounts?** Apple Developer is $99/year and required before anything reaches a device other than a simulator. Google Play is $25 once.
5. **Is there a backend, or does it need one?**

Then go to the matching file. Read only that one.

| Answer | Read |
|---|---|
| new prototype | `references/01-prototype.md` |
| build / TestFlight | `references/02-testflight-ios.md` |
| Google Play | `references/03-google-play.md` |
| submit for review | `references/04-app-store-submission.md` |
| rejected | `references/05-rejections.md` |
| backend, database, email | `references/06-stack.md` |

Turkish equivalents of the last two live in `references/tr/`.

## What the answers change

**Question 3 is the one that matters most.** If users can sign in, you owe Apple in-app account deletion (5.1.1(v)) or you will be rejected. If users can post anything visible to others, you owe visible reporting, blocking, content filtering and DM consent (1.2) — four separate things, and a long-press gesture does not count as visible. Build these into the prototype. Retrofitting them after a rejection costs a full review cycle, which is days.

**Question 4 gates everything.** Without a paid Apple account there is no TestFlight, no device install beyond a 7-day free provisioning profile, and no submission. Say this before the user spends a day building.

**Question 5 has a cheap answer.** `references/06-stack.md` covers a self-hosted setup that avoids per-service pricing: Coolify on a plain VPS, PostgreSQL, and Brevo's free tier for transactional mail.

## Rules that apply at every stage

- **Never type the user's Apple or Google password or 2FA code.** App Store Connect needs its own sign-in and the session does not carry over from the developer portal. Ask the user to sign in, wait for them to confirm, then drive the API and console steps.
- **Do not tick declaration or consent checkboxes for them.** Those are legal statements about their app.
- **"Upload succeeded" is not "accepted".** Apple can reject during processing. Poll until the build reads `VALID`.
- **Test what the reviewer will see, not what you see.** Most rejections in `05-rejections.md` were things that worked on the developer's own device and account.
- **Before blaming the reviewer, check that the thing they could not reach is reachable from outside your machine.** Several guideline rejections were a 404, a stale feature flag, or a missing DNS record.

## Non-negotiable ordering

Some steps cannot be reordered, and getting this wrong costs whole builds:

1. Bump the **version string** before building if the current one is already approved or published — the release train is closed and the upload will be refused.
2. Check the **build number is free** before building, not after.
3. Cancel any **in-review submission** before attaching a new build. Two versions cannot be in review at once.
4. **Verify the attached build** before submitting. After a cancellation, the attach call can fail while the rest of the flow carries on and submits the old binary.
