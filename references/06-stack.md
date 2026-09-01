# The stack behind the playbook

The [rejection playbook](05-rejections.md) is about getting past review. This file is about what runs underneath — the stack these apps actually ship on, and the failures we hit operating it.

Same rule as the playbook: only things that actually happened. This is **what we use**, not a claim about what you should use.

---

## What we run

| Layer | Choice | Why |
|---|---|---|
| Mobile | **Expo / React Native** (SDK 57) | One codebase for both stores; EAS or fully local builds |
| Web / API | **Next.js** | Same TypeScript on both ends |
| Database | **PostgreSQL**, self-hosted | Predictable cost; no per-row pricing surprises |
| ORM | **Prisma** | Migrations we can review before they touch production |
| Files | **MinIO** (S3-compatible), or Cloudflare R2 | Self-hosted objects; no egress bill |
| Hosting | **Coolify** on a plain VPS | Self-hosted PaaS: git deploys, TLS and containers without per-service pricing |
| Email | **Brevo** free tier over SMTP | 300 mails/day free, enough for OTP and notifications for a long while |
| Payments (Turkey) | **iyzico** | Local cards and instalments, which Stripe doesn't cover here |

One app runs on Supabase instead of self-hosted Postgres. Both work; the notes below flag which lessons are Supabase-specific.

---

## Coolify and deployment

Coolify is a self-hosted PaaS on your own VPS. It removes per-service hosting bills, and hands you the operational failures a managed platform would have absorbed.

### Disk pressure is the failure you will actually hit
Once the server disk passes roughly **80%**, deploys fail at the layer-export stage even though the build itself succeeded. Coolify surfaces this as `exit code 255` or a generic `DeploymentException` — **the real cause is hidden.** Export needs something like 20 GB free.

```bash
docker system df           # look first
docker builder prune -af   # build cache is the safe thing to delete
```

Pruning is **not symmetric**, and getting it backwards costs a deploy:

- `docker image prune -af` — safe at any time.
- `docker builder prune -af` — **never right before a deploy.** It deletes the
  build cache, so the `apt-get` layer re-runs and re-downloads packages; one
  transient network hiccup then kills the build. Off the deploy path it is the
  most effective thing you have.

Images are mostly referenced, so pruning them frees little. More Coolify failure
modes, including three unrelated causes that produce an identical error, are in
the [Coolify operations playbook](https://github.com/dursunyartasi/coolify-operations-playbook). **Never touch volumes — that's your application data.** On one incident this took the disk from 92% to 83% and freed 7.6 GB; the deploy then succeeded on retry.

The same disk pressure also shows up as a transient `No such container: <uuid>` when a build helper dies mid-build. Memory pressure produces the same symptom, so check both.

### Other deploy behaviour worth knowing
- **A deploy recreates every service in the compose file**, not just the one that changed — including your database container, whose **name changes**. Anything pinned to a container name breaks. Re-resolve it after each deploy.
- **A deploy takes roughly 200–300 seconds.** Poll for the new container plus an HTTP 200; don't assume success from the trigger call.
- **The first attempt can fail for no reason** at the compose stage. Retrying usually works and production stays up.
- **Deploys are not triggered by webhook by default** — it's a manual or API action.
- If your VPS sits **behind Cloudflare**, note that the default `urllib` user agent is blocked. Use curl or set a browser user agent when you script against your own API.

### Postgres notes
- **Supabase / PostgREST:** a new table returns `PGRST205 "Could not find the table in schema cache"` even though the table exists. The REST cache is stale. Fix: `NOTIFY pgrst, 'reload schema'`.
- **Realtime needs `wal_level=logical`.** On the default `replica`, `postgres_changes` subscribes happily and then never delivers an event — a silent failure that looks like a client bug. Changing it needs a container restart, so take a maintenance window.

---

## Email on the free tier — and the DNS trap that nearly broke it

Brevo's free tier (300 mails/day) covers OTP, password resets and notifications for a long time. Point your app at `smtp-relay.brevo.com:587`.

To be delivered rather than binned, the domain must show as **Authenticated** in Brevo, which needs:
- **DKIM** — the two CNAME records Brevo gives you
- **DMARC** — start at `p=none`
- **SPF** — `include:spf.brevo.com`
- Brevo's verification TXT record

### ⚠️ The SPF trap
We turned on Cloudflare Email Routing to *receive* mail on the same domain. Cloudflare offered to "add the missing records", saw that an SPF record already existed for Brevo, and proposed resolving the conflict by **deleting the Brevo record**.

Accepting that would have stripped authentication from every mail the app sends — OTP, notifications, password resets — and dropped them into spam. The fix is to merge both includes into **one** record:

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**A domain must have exactly one SPF record.** More than one violates the RFC and breaks all sending. Verify with `dig`, don't trust the panel.

### The MX trap — and why it's a store problem
The same domain had **no MX record at all**. It could send but not receive. The moderation contact address we had published was reaching nobody.

That is not just an email bug. App Store **Guideline 1.2** expects a working way to report content, and our own published rules promised a response within three business days. A contact address that silently discards mail is a broken commitment and a review risk. **If you publish a contact address anywhere in your store listing or in-app rules, send a test message to it.**

Also worth knowing: Brevo can restrict sending to an allowlist of IPs. Add both your development machine and your server, or production mail dies while local tests pass.

---

## Mobile build notes

Full build and upload traps are in the [playbook](05-rejections.md#build-and-upload-traps). The stack-level decisions behind them:

- **Local builds beat EAS remote when you're iterating.** Remote build queues fill up, and non-interactive EAS will not update credentials — so a provisioning profile that predates a new capability blocks you with no way through. Local `xcodebuild` plus `xcrun altool` is the escape hatch.
- **Keep `.env` out of `.gitignore` reasoning for EAS.** An ignored `.env` never reaches the EAS archive, producing empty variables and a launch crash that only appears in standalone builds.
- **Android local builds need `ANDROID_HOME`** or Gradle reports "SDK location not found".
- **Automate the Play upload with a service account** (`eas.json > submit.android`). Manual `.aab` uploads are the step that stays manual longest, and browser automation can't help — the files are far past any upload size limit.

---

## Where the VPS comes from

Coolify needs a plain VPS with root access — no managed platform required. Any provider with Docker and a public IP works. Sizing from what we run: a small instance is fine for the app, but **give the disk more room than feels necessary**, because the layer-export failure above is a disk problem, not a CPU one. Budget 20 GB of headroom beyond your images.

We run ours on Hostinger. **Referral link — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — using it earns the author a commission and gives you a discount. It is not a requirement: Coolify runs on any provider with Docker and root access, and nothing in this guide depends on the host.

---

## How this connects back to review

Several store rejections in the playbook were infrastructure problems wearing a guideline number:

| Looked like | Actually was |
|---|---|
| Guideline 1.2, no way to report content | A published contact address with no MX record |
| Play submission rejected | The declared privacy-policy URL returned 404 |
| 2.1 App Completeness, "app crashed on launch" | `.env` never reached the build |
| 2.1, "could not access the feature" | A feature flag left off in production |

Before you blame the reviewer, check whether the thing they couldn't reach is actually reachable from outside your machine.
