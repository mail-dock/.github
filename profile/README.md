# Mail Dock

**A commercially licensed, multi-domain mail server.** Postfix, Dovecot and Rspamd orchestrated as containers behind a Node 24 / Fastify control plane — shipped as a Docker Compose bundle a customer installs with one command.

> **Start in [`context`](https://github.com/mail-dock/context)** — the superproject. It aggregates every repo as a submodule and holds the architecture and conventions the others defer to.

---

## Repositories

| Repo | Purpose |
| --- | --- |
| **[context](https://github.com/mail-dock/context)** | Superproject. No application logic — submodules, architecture, conventions, run-books, skills. **Start here.** |
| **[mailserver](https://github.com/mail-dock/mailserver)** | The product. pnpm/Turborepo monorepo — `apps/{api,web,worker,cli}`, `packages/*`, `docker/`, `compose/`. |
| **[license-server](https://github.com/mail-dock/license-server)** | Licensing SaaS — key issuance, Stripe Billing, registry auth. Deployed by us, never shipped. |

```bash
git clone --recurse-submodules git@github.com:mail-dock/context.git
```

Architecture and decisions: [`architecture.md`](https://github.com/mail-dock/context/blob/main/docs/architecture.md) · [ADRs](https://github.com/mail-dock/mailserver/tree/main/docs/adrs)

---

### Install and operations

**One command to working mail** — the bootstrap script installs Docker and `mailctl`; `mailctl install` runs preflight on ports, DNS, memory and disk, brings up the stack, and hands off to the Setup Wizard.

**[Headless install](https://github.com/mail-dock/mailserver/blob/main/docs/feature-docs/headless-install.md)** — one answers file drives the same API the wizard uses, for MSPs, scripted provisioning and CI.

**Day two** — `mailctl doctor` diagnoses host, DNS, services and API in one read-only pass. `backup` and `restore` cover the database, all mail and the environment file, with checksums verified before anything is written. `upgrade` takes a backup first, so rolling back is restoring it.

## Features

### Mail

**[Domains](https://github.com/mail-dock/mailserver/blob/main/docs/feature-docs/domains.md)** — many domains on one server. Add one and Postfix accepts mail for it immediately; nothing is rendered and no daemon reloads. Each domain carries a default mailbox quota, and an `active` flag that stops inbound acceptance and IMAP login across the whole domain without deleting anything. The API returns the DNS records to publish — A, MX, SPF, DMARC, IMAP/SMTP SRV autodiscover and PTR.

**[Mailboxes](https://github.com/mail-dock/mailserver/blob/main/docs/feature-docs/mailboxes.md)** — IMAP and POP3 accounts stored as Maildir, with quotas enforced by Dovecot. Argon2id passwords, admin-initiated resets, and suspend, which rejects mail at SMTP time rather than queueing it. Sending is authenticated submission on `:587` or `:465`, with the envelope sender pinned to the mailbox or an alias that points at it.

**[Aliases, forwarders and catch-all](https://github.com/mail-dock/mailserver/blob/main/docs/feature-docs/aliases-forwarders-catchall.md)** — the cPanel routing set, all pure data. One alias to as many as 50 local or remote destinations; per-mailbox forwarders that optionally keep a local copy; a domain catch-all, or the safer default of rejecting unknown recipients outright to protect reputation.

**Spam, signing and TLS** — Rspamd runs as a Postfix milter for filtering, DKIM, ARC and DMARC, with thresholds you set per server. Caddy terminates TLS with ACME, or you bring your own certificate; a cert-sync sidecar pushes renewals into Postfix and Dovecot within 30 seconds.

### Control plane

**[REST API](https://github.com/mail-dock/mailserver/blob/main/docs/feature-docs/rest-api.md)** — `/api/v1`, with OpenAPI 3.1 generated from the Zod schemas and Swagger UI at `/api/v1/docs`. The Setup Wizard, admin UI and `mailctl` all go through it, so anything you can click you can script.

**Auth and audit** — 15-minute JWTs with rotating refresh cookies, TOTP for admin roles, and revocable `msk_` tokens for integrators. Four roles from owner down to mailbox user. Every mutation writes an audit entry with actor, before/after and IP.

**Versioned configuration** — server settings are typed data, not files. Preview a render without applying it, apply with validate-before-reload, browse the history with each version's validator output, and roll back to any earlier one.

### AI

Bring your own key (Anthropic, OpenAI-compatible) or point at a local model (Ollama, vLLM). Mail Dock ships the orchestration and prompts, never the inference — so there is no per-token cost and we never become a processor of your mail. Off until configured, with per-user opt-in, per-domain policy, attachments excluded by default, a metadata-only audit log, and a local-only flag that refuses non-private endpoints.

Every feature has a doc in [`feature-docs/`](https://github.com/mail-dock/mailserver/tree/main/docs/feature-docs).

---

## Licence model

Mail Dock is commercial software sold as an **annual per-server subscription**, priced by mailbox-count tier. Enforcement is designed around one rule: **a billing problem must never stop a customer's mail.**

**Signed keys, verified offline.** A licence is an Ed25519-signed payload — licence id, tier, mailbox and domain caps, expiry, enabled features, and an optional server fingerprint. The public key is embedded in the product, so validation needs no network. The fingerprint is a hash of machine-id, primary MAC and hostname recorded at activation; a mismatch starts a transfer flow rather than a lockout.

**Heartbeat and graceful degrade.** The worker posts usage counts daily and receives a refreshed key. Miss the heartbeat and a grace counter runs for **30 days** — no change in behaviour. Past that, the server enters *degraded*: new mailboxes and domains are blocked, an admin banner appears, AI switches off. **Delivery, IMAP and webmail keep running.** Renewing restores full function.

**Billing and distribution.** Subscriptions live in Stripe Billing, with a customer portal for self-service; `customer.subscription.*` webhooks issue, refresh and revoke keys. Container images come from a private registry that authenticates with the licence id, so a lapsed subscription loses access to *new tags only* — a running install keeps running.

**Not DRM.** Bypass is possible and accepted. Enforcement leans on the update channel and support, not on fighting the customer. The signing private key and Stripe secrets live only in the [`license-server`](https://github.com/mail-dock/license-server) deployment — never in a repository, never in a shipped image.

Details: [subscription and licence model ADR](https://github.com/mail-dock/license-server/blob/main/docs/adrs/0001-subscription-license-model.md)

---

## Install

Debian 12 / Ubuntu 22.04+, public IPv4, a hostname with matching A and PTR records.

```bash
curl -fsSL https://get.<bootstrap-host>/install.sh | sh -s -- --hostname mail.example.com
```

Installs Docker and `mailctl`, runs preflight, starts the stack, then prints the Setup Wizard URL. Add `--answers setup.yaml` to complete setup without a browser. Publish the DNS records it prints and you have working mail.

Day two: `mailctl doctor` · `backup` · `restore` · `upgrade`. Everything else is the admin UI or the API — there is nothing to hand-edit on the host.

Guides: [install](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/install.md) · [upgrade](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/upgrade.md) · [backup & restore](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/backup-restore.md) · [doctor](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/doctor.md) · [bring your own TLS](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/bring-your-own-tls.md)

---

## Develop

Node 24, pnpm 11, Docker with Compose v2.

```bash
git clone --recurse-submodules git@github.com:mail-dock/context.git
cd context/mailserver
pnpm install
cp compose/.env.example compose/.env
pnpm stack:up                 # 8 services, migrations on boot
pnpm test                     # needs TEST_DATABASE_URL for the integration half
pnpm smoke                    # end-to-end mail flow against the stack
```

API docs at `http://localhost:3000/api/v1/docs`. Full walkthrough, ports and gotchas: [dev-environment.md](https://github.com/mail-dock/mailserver/blob/main/docs/run-books/dev-environment.md).

**One rule to internalise:** never hand-edit daemon config. Edit the templates in `packages/engine-postfix-dovecot/`, then apply through the API.
