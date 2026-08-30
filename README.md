# Mobile App Shipping

**Everything we learned getting eight apps into the App Store and Google Play — including every rejection, with the root cause.**

Most mobile-shipping guides paraphrase Apple's and Google's documentation. This one doesn't. It's the accumulated scar tissue from shipping real apps in 2026: fifteen store rejections with what actually caused them, the build and upload failures that ate whole release cycles, and the setup decisions that would have prevented most of it.

Ships as a [Claude Code](https://claude.com/claude-code) skill — answer a few questions and it routes you to the right stage. It also reads fine as plain documentation.

🇹🇷 Türkçe: **[SKILL.tr.md](SKILL.tr.md)**

---

## The lesson that cost us the most

An app was rejected under **Guideline 4.2 (Minimum Functionality)** — "the usefulness of the app is limited because it seems to be intended for a small, or niche, set of users."

The root cause wasn't the app. Apple's rejection quoted, nearly word for word, a sentence we had written ourselves in the App Review Notes:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

We handed the reviewer the rejection. That's the shape of most entries here: the notice pointed one way, the cause was somewhere else.

---

## What's inside

```
SKILL.md                              the router - asks where you are, sends you to one file
references/
  01-prototype.md                     zero to an app on your phone, built to pass review
  02-testflight-ios.md                build, sign, upload, and the errors by code
  03-google-play.md                   keystore, SHA-1s, AAB, declarations that block releases
  04-app-store-submission.md          metadata, age rating, privacy, review notes, resubmission
  05-rejections.md                    the catalogue: 15 rejections with root causes
  06-stack.md                         the infrastructure underneath, and how it broke
  tr/ es/ pt-BR/ zh-CN/ ru/ de/ fr/   full translations of all six
```

You read one file, not all of them.

### A sample of the rejection catalogue

| Guideline | What the store said | Actual root cause |
|---|---|---|
| 4.2 Minimum Functionality | "small, or niche, set of users" | Our own wording in the review notes |
| 1.2 User-generated content | no filtering / reporting / blocking | Reporting existed — behind an invisible long-press gesture |
| 2.1 Demo account | couldn't sign in | The Watch target posted `email` while the backend read `identifier` |
| 3.1.1 In-app purchase | purchase signals in a free app | A plan name and a credit counter were still on screen |
| 3.1.1 (again) | same guideline, second round | A public signup screen contradicted our own B2B argument |
| 2.1(a) App Completeness | couldn't view the terms | The legal text was there — as plain, untappable text |
| 5.1.1 Photo access | requesting library permission | The modern picker never needed the permission we asked for |
| 4.0.0 Design | "not integrated with built-in mapping" | We only handed users off to Google Maps |
| Play (production) | submission rejected | The declared privacy-policy URL returned 404 |

One app went through four consecutive rejections, another three. Fixing one can expose the next — which is the entire argument for a checklist.

### Beyond the rejections

- **Build and upload failures by error code** — 90062 and 90186 (the release train closing), 90713 (a missing target icon), ITMS-90863 (noise, not a rejection), and why `eas submit` hanging for 23 minutes is best solved by calling `altool` directly.
- **The App Store Connect API traps** — the screenshot type that doesn't exist, `whatsNew` being uneditable on a first version, mixed age-rating field types, the 2025 change that removed the 12+ band, and how blind-clicking the App Privacy wizard silently records false answers.
- **Crashes that only exist in standalone builds** — a gitignored `.env` that never reached the archive and crashed *every* build, a dynamically imported native module invisible in development, and why the simulator can't catch either.
- **The infrastructure underneath** — self-hosted Coolify, PostgreSQL and Brevo's free tier, plus the DNS trap where accepting a panel's suggested fix would have stripped authentication from every email the app sends.

---

## Install as a Claude Code skill

```bash
git clone https://github.com/dursunyartasi/app-store-review-playbook.git \
  ~/.claude/skills/mobile-app-shipping
```

Claude Code picks it up next session. It triggers on submission and rejection topics on its own, or invoke it explicitly with `/mobile-app-shipping`.

For Turkish, copy `SKILL.tr.md` over `SKILL.md` in a separate skill directory — the Turkish translation is complete.

Not using Claude Code? Start at [SKILL.md](SKILL.md) and follow the table.

---

## Scope and honesty

- **Sources are anonymised.** Apps appear as App A / B / C. No bundle IDs, keys, SHA-1 fingerprints, submission IDs, IP addresses or account details appear anywhere in this repository.
- **Dated to 2026.** Store guidelines move, and Apple changed the age bands in 2025 alone. Treat specific guideline numbers as a place to start your own reading, not as current law.
- **It is not exhaustive.** It covers what we hit. We never had a 4.3 spam rejection or a Play policy violation, so there is nothing here about either.
- **It is opinionated about a stack** — Expo, Coolify, PostgreSQL, Brevo. Most of the store material applies whatever you use; the infrastructure notes are specific.

## Translations

English is canonical. Translations live in `references/<lang>/` with a matching `SKILL.<lang>.md` router, and are currently:

| Language | Status |
|---|---|
| English | canonical |
| Türkçe | complete |
| Español | complete |
| Português (BR) | complete |
| 简体中文 | complete |
| Русский | complete |
| Deutsch | complete |
| Français | complete |

Translations beyond English were produced by an AI without native-speaker review. They are faithful to the English, but if a phrasing reads oddly to you, a correction PR is genuinely welcome.

**Translations may lag.** When a new rejection lands, English is updated first. If you read a translation and something looks thin, check the English file — and a PR closing the gap is welcome.

To add a language: copy `references/` to `references/<lang>/`, translate the six files plus the router, and keep commands, error codes and guideline numbers exactly as they are. Those are strings people search for; translating them makes the file useless.

## Disclosure

The infrastructure notes link to Hostinger with a **referral link**. Using it earns the author a commission and gives you a discount. It is disclosed as such wherever it appears, and nothing in this repository depends on that host — Coolify runs on any provider with Docker and root access.

## Contributing

A rejection you got that isn't here is a gap worth filling. Open a PR adding a row to the catalogue and, where it applies, a checklist item. Keep it anonymised and root-caused — "we changed something and it passed" doesn't help the next person.

---

## Who made this

Built by **[Dursun Yartaşı](https://dursunyartasi.com)** — digital architect and entrepreneur based in Istanbul, working across digital advertising, content production and software development.

Career stops include photo editor at *National Geographic Türkiye*, marketing at *Canon Türkiye*, social media coordination at *TV8 / Acun Medya*, and senior digital marketing at *Lighthouse Worldwide Solutions EMEA*. Founder of one of Turkey's largest photography communities; today building and shipping his own software products — the ones whose rejections wrote this repository.

**→ [dursunyartasi.com](https://dursunyartasi.com)** — projects, free tools and writing.

## License

[MIT](LICENSE) — use it, fork it, adapt it for your team.
