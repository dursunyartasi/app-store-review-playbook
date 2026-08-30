# App Store & Play Review Playbook

**A submission checklist written from rejections that actually happened.**

Most store-submission guides paraphrase Apple's own documentation. This one doesn't. Every item here is a rejection collected while shipping eight iOS and Android apps in 2026 — with the root cause we eventually found, which was frequently not what the rejection notice said.

Ships as a [Claude Code](https://claude.com/claude-code) skill, but it reads fine as a plain checklist for any human or agent.

🇹🇷 Türkçe sürüm: **[SKILL.tr.md](SKILL.tr.md)**

---

## The lesson that cost us the most

An app was rejected under **Guideline 4.2 (Minimum Functionality)** — "the usefulness of the app is limited because it seems to be intended for a small, or niche, set of users."

The root cause wasn't the app. Apple's rejection quoted, nearly word for word, a sentence we had written ourselves in the App Review Notes:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

We had handed the reviewer the rejection. The full write-up, and what to say instead, is in the playbook.

---

## What's inside

- **Pre-submission checklist** — grouped by guideline (5.1.1 data & permissions, 1.2 user-generated content, 2.3.6 age rating, 4.0 design), plus what the reviewer will actually see and an Android section.
- **Rejection catalogue** — twelve real rejections in a table: what the store said, the actual root cause, and the fix that cleared it.
- **Build and upload traps** — the version train closing, altool vs. a hanging `eas submit`, the resubmission "swap trap" that silently ships your *old* build, Xcode auto-updates breaking the platform SDK, CocoaPods on Ruby 4.0, crashes that only appear in standalone builds.
- **Store registration facts** — what genuinely cannot be automated, and which choices are irreversible.

### A sample of the catalogue

| Guideline | What the store said | Actual root cause |
|---|---|---|
| 4.2 Minimum Functionality | "small, or niche, set of users" | Our own wording in the review notes |
| 1.2 User-generated content | no filtering / reporting / blocking | Reporting existed, but only behind an invisible long-press gesture |
| 2.1 Demo account | couldn't sign in | The Watch target posted `email` while the backend read `identifier` |
| 5.1.1 Photo access | requesting library permission | PHPicker never needed the permission we were asking for |
| 4.0.0 Design | "not integrated with built-in mapping" | We only handed users off to Google Maps |
| Play (production) | submission rejected | The declared privacy-policy URL returned 404 |

Fixing one rejection can invite the next. One app went through four consecutive rejections, another three — which is the real argument for a checklist.

---

## Install as a Claude Code skill

```bash
git clone https://github.com/dursunyartasi/app-store-review-playbook.git \
  ~/.claude/skills/app-store-review-playbook
```

Claude Code picks it up on the next session. Invoke it by asking about a submission — "we're submitting to the App Store", "the app got rejected", "Guideline 1.2" — or explicitly with `/app-store-review-playbook`.

For the Turkish version, copy `SKILL.tr.md` to `SKILL.md` in a separate skill directory.

Not using Claude Code? Read [SKILL.md](SKILL.md) directly — it's a plain Markdown checklist.

---

## Scope and honesty

- **Sources are anonymised.** Apps appear as App A / App B / App C. No bundle IDs, keys, SHA-1 fingerprints, submission IDs or account details are included anywhere in this repository.
- **Dated to 2026.** Store guidelines move. Treat specific guideline numbers as a starting point for your own reading, not as current law.
- **It is not exhaustive.** It covers what we hit. A rejection you get that isn't here is a gap — please open an issue or PR with the guideline, the root cause, and the fix. Root cause is the part that matters; "we changed something and it passed" doesn't help anyone.

## Contributing

Open a PR that adds a row to the rejection catalogue and, where relevant, a checklist item. Keep entries anonymised and root-caused.

---

## Who made this

Built by **[Dursun Yartaşı](https://dursunyartasi.com)** — digital architect and entrepreneur based in Istanbul, working across digital advertising, content production and software development.

Career stops include photo editor at *National Geographic Türkiye*, marketing at *Canon Türkiye*, social media coordination at *TV8 / Acun Medya*, and senior digital marketing at *Lighthouse Worldwide Solutions EMEA*. Founder of one of Turkey's largest photography communities; today building and shipping his own software products — the ones whose rejections wrote this playbook.

**→ [dursunyartasi.com](https://dursunyartasi.com)** — projects, free tools and writing.

## License

[MIT](LICENSE) — use it, fork it, adapt it for your own team.
