# App Store submission

The build is uploaded and `VALID`. This is the metadata, declarations and review notes — where most of our rejections were actually decided.

## Create the record (once)

**You cannot create the App Store Connect app record over the API.** We tried and confirmed it; do it in the browser. The bundle ID binds permanently to that record.

Everything after that — build attachment, metadata, age rating, review notes, submission — can be driven over the ASC API.

## Screenshots

- **There is no `APP_IPHONE_69` screenshot type.** The largest the API accepts is `APP_IPHONE_67` (1290×2796). Images rendered at 1320×2868 for the 6.9" device are **rejected**. Upload 6.7" and let Apple scale up.
- `whatsNew` **cannot be edited on a first version** — 409, "cannot be edited at this time". It only exists for updates.

## Age rating

- Field types are mixed: some BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), some STRING enums (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). The wrong type returns 409, and the error names the correct set.
- **Apple changed the bands in 2025: 12+ no longer exists.** They are 4+, 9+, 13+, 16+, 18+.
- Honest answers can produce 4+; raise it with `ageRatingOverrideV2` (e.g. `THIRTEEN_PLUS`).
- **If the app has any "meeting people / networking" angle, declare `matureOrSuggestiveThemes` at least `INFREQUENT_OR_MILD`.** Leaving it at none was a 2.3.6 rejection.

## App Privacy declaration

- **A national ID number is not "Sensitive Info".** Apple's sensitive list covers race, religion, sexual orientation, biometrics and similar — a national ID is not on it, so **"Other Data Types"** is the right bucket.
- **Bank details you store yourself are Collected.** Apple only exempts you when the payment provider holds them and you cannot access them.
- ⚠️ **Do not click through the wizard blind.** It renders at different heights per data type, so repeating the same click position produced answers like "User ID is used for tracking: YES" that were simply false. Screenshot and verify the final state of every item.

## App Review notes — the highest-leverage text you will write

One rejection was caused entirely by this field. Apple's 4.2 "small, or niche, set of users" rejection quoted our own sentence back at us:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Never frame the app as small, niche, invite-only, closed, private, for a specific community, or not mass-market.** Apple reads that as Ad Hoc distribution, not App Store.

Write instead: open to everyone, free, downloadable anywhere; any curated or membership tier is an *optional* layer. Then describe, in numbered steps, the path a reviewer can take **without an account**. If the app genuinely has no such path, build one before submitting — that is what cleared our 4.2.

**The field is capped at 4000 characters.** Exceeding it returns 409.

If the app has an unusual target (Watch, widget, a device-specific flow), put a "PLEASE READ FIRST" section at the very top with explicit sign-in steps.

## Demo account

Mark "Sign-In Required" and provide credentials.

- **Test them on a device first.** A 2.1 rejection came from sign-in that had never worked on the Watch target.
- **Make sure the account has content.** In one app, 16 of 17 seeded events were in the past, so a reviewer would have opened an empty app. Keep an idempotent script that rolls demo dates forward and run it before every submission.
- **A verification wall will trap the reviewer.** If a signed-up-but-unverified user sees nothing, the app looks closed. Let guests browse; require verification only on write actions.
- **Close the demo account after approval.** Its password lives in App Store Connect.

## Legal links

The Terms and Privacy links must be **tappable**, open in an in-app browser rather than kicking the user to Safari, and appear on the **login** screen too — not only signup. Plain untappable text was a 2.1(a) rejection: the reviewer could not read the terms and rejected on that alone.

## If the app is free but sells anything anywhere

3.1.1 is the trap for B2B and free apps. **Remove every price, plan name, credit counter, paywall, upgrade button and outbound purchase link.** A plan name alone was enough to sink one build.

The 3.1.3(f) "Free Stand-alone Apps" argument **did not work for us on its own.** The weak link was a public signup screen — it reads as consumer self-service and contradicts 3.1.3(c)'s "only sold directly by you to organizations". We deleted the signup screen and shipped sign-in only.

## Submitting, and resubmitting after a rejection

Order matters. Getting it wrong silently submits the wrong binary.

1. **Two versions cannot be in review at once.** Cancel the existing `reviewSubmission` (`canceled=true`) and wait for CANCELING → COMPLETE.
2. The version becomes `DEVELOPER_REJECTED` and is editable. PATCH the version string, then the build relationship.
3. ⚠️ **The swap trap.** Right after cancelling, the build-attach call returns 409. If your script continues anyway, it submits the **old** build. Retry the attach, then **verify** with `GET /appStoreVersions/{id}/build` before submitting. We shipped the wrong build once this way.
4. ⚠️ `POST reviewSubmissionItems` can return 409 `ENTITY_STATE_INVALID` while the transition finishes. It succeeds seconds later — make it retryable.

Release type is **manual** by default: after approval, someone still has to press release.

## Expect more than one round

One app went through four consecutive rejections, another three. Fixing one can expose the next, and a fix in one area can create a problem in another. **After every fix, re-read the whole checklist in `05-rejections.md`**, not just the item you changed.
