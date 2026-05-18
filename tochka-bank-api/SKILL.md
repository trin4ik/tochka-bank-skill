---
name: tochka-bank-api
description: Use when working with Tochka Bank (Точка.Банк) REST API for Russian business banking — fetching incoming payments and statements, creating outgoing payment orders, issuing invoices (счета на оплату), or accepting payments via SBP QR codes. Triggers on "Точка банк API", "tochka api", "получить выписку из Точки", "выставить счёт через Точку", "создать платёжное поручение Точка", "СБП QR Точка", or any task involving developers.tochka.com. Not for other Russian banks (Tinkoff / Sber / VTB have different schemas); not for personal banking (this skill covers ИП and ООО business accounts only); not for general Russian accounting or tax-law questions.
---

# Tochka Bank API

Practical guide for the Tochka Bank REST API (developers.tochka.com). Covers authentication, fetching payments/statements, creating payment orders, issuing invoices, closing documents, SBP acquiring, webhooks.

## Quickstart

**Pick the auth flow by what you need to do:**

| Goal | Auth | Setup command |
|------|------|---------------|
| Exploratory / sandbox | shared token | `export TOCHKA_TOKEN=working_token TOCHKA_SANDBOX=1` (no wizard) |
| Read own account (accounts, balance, incoming, drafts «На подпись»); draft payments | Personal JWT | `! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init` |
| Issue invoices / closing documents / full Open Banking statement | OAuth 2.0 + Consent | `! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init --oauth` (requires one-time app registration in ЛК Точки) |
| Multi-tenant (someone else's account) | OAuth 2.0 + Consent | same as above |

The `!` prefix is mandatory — wizards use `getpass`/`input`, which need a real TTY that Claude Code's Bash tool doesn't provide. The wizard has a preflight that exits with this exact reminder if run without a TTY.

**Once set up:** token + `customerCode` + default `accountId` land in OS credential store + `~/.config/tochka-bank-api/config.json`. Most subcommands then work without flags — see [Day-to-day commands](#day-to-day-commands) and the [At-a-glance table](#at-a-glance--task--subcommand).

**Safety:** state-changing subcommands (`create-invoice`, `create-payment-link`, `register-webhook`, etc.) and raw `curl -X POST/PUT/PATCH/DELETE` are gated by `.claude/hooks/tochka-require-confirmation.sh`. Claude Code shows a permission prompt labelled `[PROD]` or `[SANDBOX]` with the subcommand name before each. Read-only calls pass through silently.

Wizard internals, credential-storage details, JWT permissions list, OAuth app registration pitfalls: [references/auth.md](references/auth.md#wizards-and-oauth-app-registration).

Official docs: https://developers.tochka.com/docs/tochka-api/
Interactive ReDoc reference: https://enter.tochka.com/doc/v2/redoc
Offline OpenAPI spec: [references/swagger.json](references/swagger.json) — OpenAPI `3.1.0`, Tochka.API `v1.90.4-stable`, downloaded **2026-04-20**. Ships only `v1.0` paths (59 operations); `v2.0` endpoints that the client uses (`/open-banking/v2.0/statements`, `/payment/v2.0/for-sign`, `/acquiring/v2.0/payments`, `/invoice/v2.0/bills`) aren't in swagger — use [references/endpoints.md](references/endpoints.md) and ReDoc for those.

## Table of Contents

- [Quickstart](#quickstart)
- [Critical first step: verify against ReDoc](#critical-first-step-verify-against-redoc)
- [Base URLs](#base-urls)
- [Authentication — JWT vs OAuth](#authentication--jwt-vs-oauth)
- [Common workflows](#common-workflows)
  - [At a glance — task → subcommand](#at-a-glance--task--subcommand)
  - [Critical gotchas before writing code](#critical-gotchas-before-writing-code)
- [Helper script](#helper-script)
  - [Day-to-day commands](#day-to-day-commands)
  - [Token resolution](#token-resolution)
  - [Repo-level hook guarding state-changing calls](#repo-level-hook-guarding-state-changing-calls)
- [Error → fix cheatsheet](#error--fix-cheatsheet)
- [Operational pitfalls (strategic)](#operational-pitfalls-strategic)

## Critical first step: verify against ReDoc

Tochka rotates endpoint paths and field names without strict semver. Before writing code that hits a real endpoint, **open the corresponding ReDoc page at https://enter.tochka.com/doc/v2/redoc and confirm the exact path, query params, and required fields**. Schemas here and in [references/endpoints.md](references/endpoints.md) are correct as of 2026-04-20, but treat ReDoc as source of truth for any request body or response shape.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://enter.tochka.com/uapi` |
| Sandbox | `https://enter.tochka.com/sandbox/v2` |

Sandbox accepts `Bearer working_token` — see [references/auth.md#sandbox](references/auth.md#sandbox).

## Authentication — JWT vs OAuth

| Flow | When to use | Effort | Wizard |
|------|-------------|--------|--------|
| **Personal JWT** | You own the account, single-company automation | Low — generate in online banking, paste into wizard | `init` |
| **OAuth 2.0 + Consents** | Need Invoice API or full statement on your own account, OR delegated access to another person's/company's account | High — one-time app registration in ЛК + local HTTPS callback | `init --oauth` |

Required header (both flows): `Authorization: Bearer <token>`.

**What each flow can do** (verified on prod 2026-04-20):

| Feature | Personal JWT | OAuth + Consent |
|---|:---:|:---:|
| List accounts, balances | ✅ | ✅ |
| `GET /acquiring/v2.0/payments` (входящие SBP/карты) | ✅ | ✅ |
| `GET /payment/v2.0/for-sign` (черновики «На подпись») | ✅ | ✅ |
| `POST /payment/v2.0/for-sign` (создать платёжку) | ✅* | ✅ |
| SBP QR / merchant | ✅* | ✅ |
| **Полная выписка по счёту** (Open Banking) | ❌ 501 | ✅ |
| **История исполненных платёжек** | ❌ (выпадают из `for-sign` после подписания) | ✅ (через выписку) |
| **Invoice API** (создание / email / PDF / удаление) | ❌ 501 | ✅ |
| **Closing documents** (акт / УПД / ТОРГ-12 / счёт-фактура) | ❌ 501 | ✅ |

<sub>* зависит от permissions выбранных при генерации токена. Full protocol + permissions list: [references/auth.md](references/auth.md#permissions).</sub>

**Prefer `/v2.0/` paths** where available (Open Banking, Acquiring, Payment, SBP). Invoice API is split: `/v2.0/` for the create endpoint, `/v1.0/` for everything else (PDF, email, payment-status, delete). Closing documents are `/v1.0/` only.

## Common workflows

### At a glance — task → subcommand

| I want to... | Subcommand | Auth | Safety |
|--------------|------------|:---:|:---:|
| List all accounts | `list-accounts` | JWT | read |
| Check current balance | `get-balance` | JWT | read |
| Incoming SBP/card payments | `list-incoming` | JWT | read |
| Outgoing drafts («На подпись») | `list-for-sign` | JWT | read |
| Create outgoing payment order | `curl POST /payment/v2.0/for-sign` (no helper) | JWT | 🛑 hook-gated |
| Full bank statement (Open Banking) | `list-statement --from ... --to ...` | OAuth | read |
| Create invoice | `create-invoice --format id` | OAuth | 🛑 hook-gated |
| Email invoice | `send-invoice --document-id ... --email ...` | OAuth | 🛑 hook-gated |
| Delete invoice | `curl DELETE /invoice/v2.0/bills/{cc}/{id}` | OAuth | 🛑 hook-gated |
| Create closing doc (акт/УПД/ТОРГ-12/счёт-фактура) | `create-closing-doc --kind ... --format id` | OAuth | 🛑 hook-gated |
| Email/download/delete closing doc | `get-/send-/delete-closing-doc` | OAuth | read / 🛑 |
| Create payment link (интернет-эквайринг) | `create-payment-link --format url` | JWT* | 🛑 hook-gated |
| Accept payments via SBP QR | `curl POST /sbp/v2.0/...` (no helper) | JWT* | 🛑 hook-gated |
| Register webhook (live push) | `register-webhook --url ...` | JWT/OAuth | 🛑 hook-gated |
| List / test / delete webhook | `list-/test-/delete-webhook` | JWT/OAuth | read / 🛑 |
| Daily acquiring reconciliation | `list-registry --date ...` | JWT | read |
| Diagnose OAuth 403 (scope issues) | `list-consents` / `get-consent --consent-id ...` | OAuth | read |
| Show saved defaults | `config` | — | read |

<sub>JWT = personal JWT from ЛК Точки. OAuth = OAuth 2.0 + Consent, set up via `init --oauth`. JWT* = works with JWT that has the specific permission (`MakeAcquiringOperation` for payment links, `EditSBPData` for SBP). 🛑 = state-changing call, gated by [the repo hook](#repo-level-hook-guarding-state-changing-calls).</sub>

`create-invoice`, `create-closing-doc`, and `create-payment-link` support `--format {json,id,url}` — use `--format id` / `--format url` to get a single value on stdout (envelope goes to stderr), so shell pipelines don't need `jq`.

Full endpoint schemas, request/response shapes, validation rules: [references/endpoints.md](references/endpoints.md). Field differences for ИП vs ООО: [endpoints.md#ип-vs-ооо — field differences](references/endpoints.md#ип-vs-ооо--field-differences-in-invoices-and-payment-orders). Webhook signature verification + retry semantics: [references/webhooks.md](references/webhooks.md).

### Critical gotchas before writing code

- **`accountId` contains a slash**: `{20-digit-account}/{9-digit-BIC}`. URL-encode as `%2F` in paths; pass as-is in bodies. Forgetting → `404 Not found statement under account`.
- **SBP QR `amount` is in KOPECKS**, but `create-payment-link` `amount` is in RUBLES (float). Easy to confuse — 1500 ₽ = `150000` for SBP, `1500.00` for payment link.
- **`paymentPurpose` cannot contain long dash `—`** (U+2014). Use hyphen `-` or comma.
- **`paymentDate` must be ≤ today** — API rejects future dates.
- **Statement dates are pure dates**, not datetimes: pass `"2026-04-01"`, not `"2026-04-01T00:00:00+03:00"`.
- **Invoice body capitalisation**: `SecondSide` (capital S), `Content.Invoice.Positions`, `taxCode` (not `INN`), `quantity` (not `count`), `totalAmount` (not `amount`), `unitCode` with trailing dot (`"шт."`, not ОКЕИ code), `ndsKind` enum `without_nds`/`nds_0`/`nds_5`/`nds_7`/`nds_10`/`nds_22` — **`nds_22` replaced old 20% in 2026**. Old `vat_20` / `entrepreneur` rejected.
- **`unitCode` is a closed whitelist**: only `шт.` `тыс.шт.` `компл.` `пар.` `усл.ед.` `упак.` `услуга.` `пач.` `мин.` `ч.` `сут.` `г.` `кг.` `л.` `м.` `м2.` `м3.` `км.` `га.` `кВт.` `кВт.ч.` are accepted (period included). **No `мес.` / `год.`** — for monthly/annual subscriptions use `услуга.` or `шт.` and put the period inside the position name. Anything else → `400 Validation Error: Field Content-…-Positions-0-unitCode : Input should be …`.
- **Invoice response**: `Data.documentId` (not `documentUid` — removed).
- **Payment-order body is a flat `{"Data": {...fields...}}`, not `Data.Payment: [...]`** — nested/array shape returns `"Field X: Field required"` for every field.

## Helper script

`scripts/tochka_client.py` is a thin Python client wrapping the most common endpoints. Stdlib-only, no dependencies. 19 subcommands.

First-time setup: `init` (personal JWT) or `init --oauth`. See [Quickstart](#quickstart) for the exact command, and [references/auth.md#wizards-and-oauth-app-registration](references/auth.md#wizards-and-oauth-app-registration) for what the wizards do, how secrets are stored, and OAuth-app-registration pitfalls.

### Day-to-day commands

After `init`, the token is picked up automatically — no need to export. `customerCode` and default `accountId` are also saved, so most commands work without flags:

```bash
# Works with personal JWT on prod:
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py config          # show saved defaults
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-accounts
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py get-balance     # balance across all accounts
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-incoming   # SBP/card incoming
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-for-sign   # drafts awaiting signature
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-registry --date 2026-04-20

# Requires OAuth + Consent flow (501 with personal JWT):
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-statement --from 2026-04-01 --to 2026-04-20
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py create-invoice --document-number 7 --amount 10000 --purpose "Услуги" --buyer-inn 7700000000 --buyer-name "ООО Покупатель" --buyer-kpp 770001001 --save-pdf ru/customers/acme/payments/ --format id
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py send-invoice --document-id <documentId> --email buyer@example.com
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py create-closing-doc --kind act --document-number A-1 --amount 10000 --purpose "Услуги" --buyer-inn 7700000000 --buyer-name "ООО Покупатель" --buyer-kpp 770001001 --parent-invoice-id <documentId> --save-pdf ru/customers/acme/payments/
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py list-consents                   # diagnose 403
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py register-webhook --url https://receiver.example.com/tochka --events incomingPayment incomingSbpPayment
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py test-webhook                    # ping the URL
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py create-payment-link --amount 1000 --purpose "Оплата услуг по договору" --format url
```

`--document-number` is **required** on `create-invoice` and `create-closing-doc` — no auto-default. Use the flat per-customer counter (highest N from `ru/customers/<slug>/payments/` + 1). `--save-pdf DIR` auto-downloads the rendered PDF **and writes a JSON sidecar with the same basename** (`<filename>.json` next to `<filename>.pdf`) containing `documentId`, `customerCode`, document number/date, amount, buyer, parent invoice id. This is the only durable record of the `documentId` — Tochka has no list endpoint, so without the sidecar the UUID has to be copied manually from ЛК. **Always commit the sidecars together with the PDF.**

For richer workflows, import the relevant `cmd_*` function and adapt it.

### Token resolution

The client tries, in order:
1. `$TOCHKA_TOKEN` env var (useful in CI / one-off calls)
2. OAuth `access_token` from Keychain (auto-refreshed when near expiry, if `auth_mode=oauth`)
3. OS credential store — service `tochka-bank-api-token` (personal JWT)
4. `~/.config/tochka-bank-api/token` file (last-resort fallback, chmod 600)

### Repo-level hook guarding state-changing calls

The hook at `.claude/hooks/tochka-require-confirmation.sh` **blocks** state-changing Tochka calls — `curl -X POST|PUT|PATCH|DELETE`, or any of these subcommands: `create-invoice`, `send-invoice`, `create-closing-doc`, `send-closing-doc`, `delete-closing-doc`, `register-webhook`, `delete-webhook`, `test-webhook`, `create-payment-link`.

When the hook blocks, Claude Code pops a permission prompt with the command and a label like `[PROD] create-invoice — real money / bank records may be affected` or `[SANDBOX] create-invoice — sandbox only, safe to allow`. Pick **Allow** (one-time) or **Allow always** (for the session). There is no env-var bypass — the built-in prompt is the only confirmation path.

Read-only subcommands (`list-*`, `get-*`, `config`, `init` validation) pass through silently.

## Error → fix cheatsheet

| Exact error / symptom | Likely cause | Fix |
|-----------------------|--------------|-----|
| HTTP 501 on `/invoice/...`, `/open-banking/.../statements`, `/closing-documents` | personal JWT can't reach this API tier | switch to OAuth (`init --oauth`) |
| HTTP 401 on OAuth endpoints after idle period | refresh_token expired (30-day window) | re-run `init --oauth` — there's no silent refresh path once the refresh token dies |
| HTTP 403 with recent token | scope missing or consent expired | `list-consents` / `get-consent --consent-id ...` to see what's actually granted; re-run `init --oauth` if consent lapsed |
| HTTP 403 with old token | JWT TTL elapsed | regenerate JWT in ЛК → Интеграции и API → новый ключ → `init` again |
| `404 Not found statement under account` | un-encoded `/` in `accountId` URL path | URL-encode as `%2F` (in body pass the slash as-is) |
| `"Datetimes provided to dates should have zero time"` | full ISO datetime in statement date fields | pass `YYYY-MM-DD` only |
| `"should be less than today or equal"` on payment | future-dated `paymentDate` | use today or earlier; for deferred send, submit draft and have user sign on target date |
| `"forbidden symbols: —"` on payment purpose | U+2014 long dash | replace with `-` (hyphen) or `,` |
| `Field …-Positions-0-unitCode : Input should be 'шт.', …` on invoice/closing-doc | unitCode outside the closed whitelist (e.g. `мес.`, `год.`, ОКЕИ code, no trailing dot) | use one from the whitelist — for subscriptions `услуга.` or `шт.` with period inside `name`. Helper now validates client-side via argparse `choices`. |
| `"Проверьте номер счёта — в выбранном банке такого счёта нет"` | fake/typo counterparty reqs | counterparty account is validated against ЦБ registry — use real reqs |
| `"Field X: Field required"` for every field on payment | nested `Data.Payment` instead of flat `Data.{...}` | flatten body — v2.0 schema is flat, not array/nested |
| Invoice creation works but response lacks `documentUid` | looking at the wrong field | use `Data.documentId` (the `documentUid` alias was removed) |
| `error: init needs a real terminal...` / wizard hangs | no TTY (ran through agent Bash) | user must run with `!` prefix: `! python3 .../tochka_client.py init` |
| Invoices / payments "succeed" but don't appear in ЛК | `TOCHKA_SANDBOX=1` stuck in env, hitting sandbox | `unset TOCHKA_SANDBOX`; confirm target env in hook prompt label or by running `list-accounts` against known prod `customerCode` |
| OAuth callback never arrives (5-min timeout) | registered Redirect URL ≠ wizard default | re-run `init --oauth --redirect-url <exact URL from ЛК>` |
| OAuth: browser shows cert warning during authorize | mkcert not installed, using self-signed cert | `brew install mkcert && mkcert -install` then re-run `init --oauth` (or click through the warning once) |
| `OSError: [Errno 48] Address already in use` during `init --oauth` | port 8443 held by previous wizard run | `lsof -i :8443` → kill the process, or pass `--redirect-url https://127.0.0.1:<other-port>/callback` (must match the ЛК registration) |
| `security: SecKeychainSearchCopyNext` / Keychain denied | macOS Keychain access denied / locked | unlock Keychain, re-run; or use `init --storage file` to fall back to chmod-600 file |
| PDF download hangs / times out | first render takes ~30s server-side | client already uses 90s timeout — if it still fails, retry once |
| Payment signed but not in `list-for-sign` anymore | once signed & dispatched, it leaves «На подпись» | use `list-statement` (OAuth) to see executed payments |

Not on the list? Fall back to:
1. Live ReDoc at https://enter.tochka.com/doc/v2/redoc (authoritative on body shapes).
2. Developer docs: https://developers.tochka.com/docs/tochka-api/.
3. Telegram support chat linked from the developer portal — Tochka's docs occasionally lag the API; support replies fast.

## Operational pitfalls (strategic)

These aren't error messages — they're architectural caveats to factor into planning.

- **No idempotency keys**: Tochka's payment endpoints don't accept `Idempotency-Key`. Track your own `paymentNumber` / external IDs to avoid duplicate sends on retry.
- **Russian field names in responses**: some fields and enum values are Cyrillic strings (payment status `"Исполнен"`, `unitCode` `"услуга."`). Match on the exact string from ReDoc, not a translation.
- **Sandbox ≠ prod**: sandbox accepts `working_token` for everything (including Invoice + Open Banking). Prod splits by auth tier. Schema debugging → sandbox; access verification → prod with a real low-scope token.
- **Отношение к стандартам АФТ/ОБР**: текущая Точка API **не соответствует** утверждённым ЦБ стандартам АФТ (wiki.openbankingrussia.ru, wiki.opendatarussia.ru) — ни Accounts v2.0/v3.0, ни Payment Initiation v1.0.0, ни ФАПИ Advanced v2.0. Ближайший watershed — **01.10.2026** (дата введения пакета стандартов v2.0 ЦБ): ждите возможного перевыпуска OAuth-flow и изменения field names в `/accounts` / `/payment`. Не копируйте OBR/OBIE-примеры в расчёте, что они отработают на Точке.
