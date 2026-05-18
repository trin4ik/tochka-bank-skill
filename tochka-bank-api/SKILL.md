---
name: tochka-bank-api
description: Tochka Bank (Точка-Банк) REST API at developers.tochka.com for IE/LLC business accounts — statements, payments, invoices (счета), closing documents (акты/УПД), SBP, acquiring, webhooks. NOT for Tinkoff/Sber/VTB, personal banking, 1C accounting, or tax questions.
---

# Tochka Bank API

Practical guide for the Tochka Bank REST API (developers.tochka.com). Auth, payments/statements, payment orders, invoices, closing docs, SBP acquiring, webhooks.

## Quickstart

| Goal | Auth | Setup |
|------|------|------|
| Sandbox / experiment | shared token | `export TOCHKA_TOKEN=working_token TOCHKA_SANDBOX=1` |
| Own account + payment drafts | Personal JWT | `! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init` |
| Invoices / closing docs / full statement | OAuth + Consent | `! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init --oauth` |

The `!` prefix is **mandatory** — wizards read secrets via `getpass`, which needs a real TTY that the agent's Bash tool doesn't provide. After `init`, the token + `customerCode` + default `accountId` land in the OS credential store + `~/.config/tochka-bank-api/config.json` — most subcommands then work without flags.

JWT covers: list of accounts, balance, incoming SBP/card payments, payment drafts in «На подпись» (awaiting-signature queue), SBP QR, payment links. OAuth is required for the full Open Banking statement, Invoice API, and closing documents — JWT returns **501** for those.

Full JWT/OAuth protocol + 19 permissions + OAuth-app pitfalls (including `localhost` vs `127.0.0.1`): [references/auth.md](references/auth.md).

| Env | URL |
|---|---|
| Production | `https://enter.tochka.com/uapi` |
| Sandbox | `https://enter.tochka.com/sandbox/v2` |

## Task → subcommand

| Goal | Subcommand | Auth | Safety |
|------|------------|:---:|:---:|
| List accounts | `list-accounts` | JWT | r |
| Current balance | `get-balance` | JWT | r |
| Incoming SBP/card payments | `list-incoming` | JWT | r |
| Payment drafts «На подпись» | `list-for-sign` | JWT | r |
| Create payment order | `curl POST /payment/v2.0/for-sign` (no helper) | JWT | 🛑 |
| Full statement | `list-statement --from … --to …` | OAuth | r |
| Create invoice | `create-invoice --format id` | OAuth | 🛑 |
| Email / delete invoice | `send-invoice` / `curl DELETE /invoice/v2.0/bills/{cc}/{id}` | OAuth | 🛑 |
| Closing doc (акт/УПД/ТОРГ-12/счёт-фактура) | `create-closing-doc --kind …` | OAuth | 🛑 |
| Email/download/delete closing doc | `get-/send-/delete-closing-doc` | OAuth | r / 🛑 |
| Acquiring payment link | `create-payment-link --format url` | JWT* | 🛑 |
| Accept SBP QR payments | `curl POST /sbp/v2.0/…` (no helper) | JWT* | 🛑 |
| Webhook: register / list / test / delete | `register-/list-/test-/delete-webhook` | JWT/OAuth | r / 🛑 |
| Acquiring daily registry (reconciliation) | `list-registry --date …` | JWT | r |
| Diagnose OAuth 403 | `list-consents` / `get-consent --consent-id …` | OAuth | r |
| Show saved defaults | `config` | — | r |

<sub>JWT* — JWT with the right permission (`MakeAcquiringOperation` / `EditSBPData`). 🛑 = state-changing, gated by the hook (see below).</sub>

`create-invoice` / `create-closing-doc` / `create-payment-link` support `--format {json,id,url}` — `--format id|url` writes a single value to stdout (envelope to stderr), so shell pipelines don't need `jq`.

Full schemas / request shapes / validation / IE-vs-LLC differences: [references/endpoints.md](references/endpoints.md). Webhook signature + retry: [references/webhooks.md](references/webhooks.md). **Before shipping production code**, verify against the live ReDoc — Tochka rotates paths and field names without semver: https://enter.tochka.com/doc/v2/redoc.

## Critical gotchas

These bite almost every time:

- **`accountId` contains a slash** `{20-digit-account}/{9-digit-BIC}`: encode as `%2F` in URLs, pass as-is in bodies. Otherwise: `404 Not found statement under account`.
- **`paymentPurpose` can't contain em-dash `—` (U+2014)** — the bank rejects it. Use hyphen `-` or comma.
- **`paymentDate` must be ≤ today** — future dates are rejected.
- **Statement dates are pure dates**, not datetimes: `"2026-04-01"`, not `"2026-04-01T00:00:00+03:00"`.
- **SBP QR `amount` is in KOPECKS**, but `create-payment-link` `amount` is in RUBLES (float). 1500 ₽ = `150000` for SBP QR, `1500.00` for payment link.
- **Invoice body — exact capitalisation matters**: `SecondSide` (capital S), `Content.Invoice.Positions`, `taxCode` (not `INN`), `quantity` (not `count`), `totalAmount` (not `amount`), `unitCode` with trailing dot (`"шт."`, not OKEI code), `ndsKind` ∈ `without_nds`/`nds_0`/`nds_5`/`nds_7`/`nds_10`/`nds_22` (**`nds_22` replaced `vat_20` from 2026**). Response → `Data.documentId` (not `documentUid` — removed).
- **`unitCode` is a closed whitelist** — `шт. тыс.шт. компл. пар. усл.ед. упак. услуга. пач. мин. ч. сут. г. кг. л. м. м2. м3. км. га. кВт. кВт.ч.` (trailing dot included). **No `мес.`/`год.`** — for subscriptions use `услуга.` or `шт.` with the period inside the position name. `tochka_client.py` enforces this via argparse `choices`.
- **Closing-doc `SecondSide` uses v1.0 lowercase fields** `kpp` / `secondSideName` — NOT the v2.0 invoice shape `KPP` / `legalName`. Tochka silently drops the v2.0 shape on closing-docs → buyer КПП missing in PDF → ЭДО rejects signing. Also: `Content.Act.totalAmount` is required at the block level, not just per-position. See [endpoints.md#closing-documents](references/endpoints.md#closing-documents-акты--упд--торг-12--счёт-фактура).
- **Payment-order body is a flat `{"Data": {…fields…}}`**, not `Data.Payment: [...]` — otherwise the API returns `"Field X: Field required"` for every field.

Error → fix cheatsheet (HTTP 501/403/401, OAuth callback timeouts, Keychain, PDF render): [references/errors.md](references/errors.md). Strategic caveats (no idempotency keys, Cyrillic enum values, divergence from Russian Central Bank Open Banking standards): [references/endpoints.md#operational-pitfalls](references/endpoints.md#operational-pitfalls).

## Helper script

`scripts/tochka_client.py` — Python, stdlib-only, 19 subcommands. After `init`, the token and defaults are picked up automatically.

Token resolution: `$TOCHKA_TOKEN` → OAuth `access_token` from Keychain (auto-refresh) → OS credential store → `~/.config/tochka-bank-api/token` (chmod 600 fallback).

Each subcommand's `--help` lists exact flags. Full invoice flow example:

```bash
python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py create-invoice \
  --document-number 7 --amount 10000 --purpose "Услуги" \
  --buyer-inn 7700000000 --buyer-name "ООО Покупатель" --buyer-kpp 770001001 \
  --save-pdf ru/customers/acme/payments/ --format id
```

`--document-number` is **required** on `create-invoice` / `create-closing-doc` — there's no auto-default. Use the flat per-customer counter (`max N + 1`). `--save-pdf DIR` auto-downloads the rendered PDF **and writes a JSON sidecar with the same basename** (`<filename>.json` next to `<filename>.pdf`) containing `documentId`, `customerCode`, document number/date, amount, buyer, parent-invoice id. This is the only durable cache of `documentId` — Tochka has no list endpoint, so without the sidecar the UUID has to be copied manually from online banking on every later operation. **Commit the sidecar together with the PDF.** For richer workflows, import the relevant `cmd_*` function from the script.

## Repo-level hook (Claude Code only)

`.claude/hooks/tochka-require-confirmation.sh` blocks state-changing calls (`curl -X POST|PUT|PATCH|DELETE` to `enter.tochka.com`, plus `create-*` / `send-*` / `delete-*` / `register-webhook` / `test-webhook` subcommands) and raises a permission prompt labelled `[PROD]` or `[SANDBOX]` + subcommand name. Read-only calls (`list-*`, `get-*`, `config`) pass through silently. There's no env-var bypass — the built-in prompt is the only confirmation path.

Installation and registration in `.claude/settings.json`: see [README](../../README.md#хук-подтверждения-только-claude-code).

## Offline OpenAPI

[schemas/swagger.json](schemas/swagger.json) — OpenAPI 3.1.0, Tochka.API v1.90.4-stable (downloaded 2026-04-20). Contains only `v1.0` paths (59 operations); the `v2.0` endpoints the client uses (`/open-banking/v2.0/statements`, `/payment/v2.0/for-sign`, `/acquiring/v2.0/payments`, `/invoice/v2.0/bills`) are **absent** — use [endpoints.md](references/endpoints.md) + live ReDoc for those.

Look up schemas via `jq` (don't `Read` the file — it's 440 KB):
```bash
jq '.paths."/payment/v1.0/for-sign".post' schemas/swagger.json
jq '.components.schemas.PositionModel' schemas/swagger.json
jq '.paths | keys' schemas/swagger.json | head -20
```

Refresh: `curl https://enter.tochka.com/doc/openapi/swagger.json -o schemas/swagger.json`.

## Sources

- Docs: https://developers.tochka.com/docs/tochka-api/
- Live ReDoc (source of truth for request/response shapes): https://enter.tochka.com/doc/v2/redoc
