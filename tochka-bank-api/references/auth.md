# Tochka Bank API — Authentication

Two flows: **JWT** (simple, single-company) and **OAuth 2.0 + Consents** (multi-tenant). Pick JWT unless you're building a SaaS that authorizes access to other people's accounts.

## JWT (recommended for own-account automation)

### Generation in online banking

1. https://enter.tochka.com (web online banking)
2. **Интеграции и API** → **Подключить**
3. **Сгенерировать JWT-ключ**
4. Set:
   - Token name (any label)
   - TTL — shortest workable duration
   - Permissions (see [Permissions](#permissions)) — uncheck what you don't need
   - Companies — by default all the user's companies are included
5. Confirm via SMS
6. Copy both the **JWT** and the **client_id** (client_id is needed for webhook setup)

The JWT can't be re-displayed after closing the page. Save it to a secret manager immediately.

### Using the JWT

```
Authorization: Bearer <jwt>
```

Notes:
- Editing permissions isn't supported — any change requires reissuance (new JWT and new `client_id`).
- Revocation: delete the token in online banking → subsequent calls return 403.
- A reissuance reminder shows up about 1 week before TTL expiry.
- Source: https://developers.tochka.com/docs/tochka-api/algoritm-raboty-s-jwt-tokenom

## OAuth 2.0 + Consent flow (multi-tenant)

Use this only when an end user (a different person/company than the integrator) grants your app access to their Tochka account. Standard OAuth 2.0 with two grant types: `client_credentials` (app-to-app) and `authorization_code` (delegated user access).

### High-level flow

1. **Register the app** with Tochka — receive `client_id` and `client_secret`.
2. **Get an app token** via `client_credentials`:
   ```
   POST https://enter.tochka.com/connect/token
   Content-Type: application/x-www-form-urlencoded
   grant_type=client_credentials&client_id=...&client_secret=...&scope=accounts payments
   ```
3. **Create a Consent** (defines what the user is being asked to authorize):
   ```
   POST https://enter.tochka.com/uapi/consent/v1.0/consents
   Authorization: Bearer <app_token>
   Content-Type: application/json
   { "Data": { "permissions": ["ReadAccountsBasic", "ReadStatements"] } }
   ```
   Response: `Data.consentId`. Canonical path per swagger v1.90.4-stable is `/uapi/consent/v1.0/consents`. The legacy alias `/uapi/v1.0/consents` (without the `consent/` segment) still resolves on prod as of 2026-04-20, but new code should use the canonical form.

4. **Redirect the user** to authorize the consent:
   ```
   https://enter.tochka.com/connect/authorize?
     response_type=code&
     client_id=<your_client_id>&
     redirect_uri=<your_callback>&
     scope=accounts statements&
     consent_id=<consentId>
   ```
5. **Exchange the returned `code` for tokens**:
   ```
   POST https://enter.tochka.com/connect/token
   grant_type=authorization_code&code=...&redirect_uri=...&client_id=...&client_secret=...
   ```
6. Use the resulting `access_token` (24h) for API calls. Refresh with the `refresh_token` (30d).

### Token TTLs

| Token | Lifetime |
|---|---|
| Access token | 24 hours |
| Refresh token | 30 days |
| JWT (online-banking) | user-configurable |

Source: https://developers.tochka.com/docs/tochka-api/algoritm-raboty-po-oauth-2.0

## Sandbox

Base: `https://enter.tochka.com/sandbox/v2`. No registration required. Use the documented sandbox JWT (typically `Bearer working_token` — confirm against https://developers.tochka.com/docs/tochka-api/pesochnica). Sandbox returns mock data and accepts mock writes, but not all production endpoints are mirrored.

## Permissions

Granular permissions, selectable when generating a JWT or requesting an OAuth consent. The list below is **authoritative** — copied verbatim from Tochka's validation response on 2026-04-20. Anything not in the list is rejected.

### Full accepted set (19 values)

```
ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements,
ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits,
ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData,
CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData,
MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData,
MakeCustomer, ManageGuarantee
```

(Source: the error body returned by `POST /uapi/v1.0/consents` when an invalid permission is submitted — Tochka helpfully enumerates all accepted values. The public docs at https://developers.tochka.com/docs/tochka-api/api/rabota-s-razresheniyami are out of date; the response above is ground truth.)

### Grouped by use case

**Accounts & balances:** `ReadAccountsBasic` (list accounts), `ReadAccountsDetail` (full details), `ReadBalances` (balances), `ReadCustomerData` (legal entity data — **not** `ReadCustomers` as older docs say).

**Statements & transactions:** `ReadStatements` (full async Open Banking statement), `ReadTransactionsBasic` / `Credits` / `Debits` / `Detail`.

**Outgoing payments:** `CreatePaymentForSign` (draft into «На подпись» queue), `CreatePaymentOrder` (direct creation).

**Invoices:** `ManageInvoiceData` — create/read/update/delete (covers everything; `ReadInvoiceData` doesn't exist).

**SBP:** `ReadSBPData` (read merchant data), `EditSBPData` (register/edit QR codes, merchant settings).

**Acquiring:** `ReadAcquiringData` (read), `MakeAcquiringOperation` (create/manage — **not** `EditAcquiringData`).

**Webhooks:** `ManageWebhookData`.

**Specialized (usually not needed):** `MakeCustomer` (partner banks), `ManageGuarantee` (bank guarantees).

**Principle of least privilege:** only request what you actually need. For OAuth, changing a consent after creation means recreating it and re-running user authorization. `scripts/tochka_client.py` uses 17 of 19 (`DEFAULT_OAUTH_PERMISSIONS` constant; skips `MakeCustomer` and `ManageGuarantee`).

## Token lifecycle and 403 handling

On a 403:
1. Check token expiry (JWT — decode the `exp` claim; OAuth — check refresh date).
2. If expired:
   - JWT — regenerate in online banking.
   - OAuth — refresh via `grant_type=refresh_token`.
3. If not expired — the permission scope is wrong; verify the token holds the permission for the endpoint.
4. Sandbox sometimes returns 403 on endpoints it doesn't implement — try the same call against prod with a minimal-permission token.

OAuth refresh:
```bash
curl -X POST https://enter.tochka.com/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&refresh_token=<rt>&client_id=...&client_secret=..."
```

## Wizards and OAuth app registration

### Why the `!` prefix is mandatory

`init` and `init --oauth` read secrets via `getpass` (and `input()` for confirmations). These require a real TTY (`termios`); the Bash tool in Claude Code doesn't provide one, so running through the agent silently hangs or fails with `EOFError`. The wizard detects this on startup and exits with an actionable message.

**Pattern:** the agent instructs the user to run the command in their own shell via the `!` prefix:

```
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init
```

With `!` the command runs in the user's live TTY; the JWT/OAuth secret is entered directly into their terminal and never passes through the agent's tool-use channel or the conversation transcript.

**Never ask the user to paste a production JWT into chat** — it would land in transcript backups. If they did so by mistake, rotate immediately: online banking → Интеграции и API → delete the token → new one. Pasting the sandbox `working_token` is fine; only the prod JWT warrants the `!` flow.

### `init` wizard (personal JWT)

The wizard:
1. Explains how to generate a JWT in online banking (Интеграции и API → Сгенерировать JWT-ключ) and which permissions to tick.
2. Reads the JWT via hidden input (`getpass` — not written to shell history).
3. **Validates** against PROD via `GET /accounts` — only saves on success.
4. **Stores the JWT in the OS credential store** (encrypted at rest, password/biometry-gated):
   - macOS → Keychain (`security add-generic-password`)
   - Linux → `secret-tool` (GNOME Keyring / KWallet via libsecret)
   - Windows → Credential Manager
   - WSL → falls back to Windows Credential Manager via `powershell.exe`
5. If the credential store is unavailable — falls back to `~/.config/tochka-bank-api/token` chmod 600 (warns this is less secure).
6. Prints `customerCode` + `accountId` (needed for invoice/statement commands).
7. Saves defaults to `~/.config/tochka-bank-api/config.json`.

Force file-only storage (headless): `init --storage file`.

Sandbox: prefix `TOCHKA_SANDBOX=1`. Use the shared `working_token` — easier to just `export TOCHKA_TOKEN=working_token TOCHKA_SANDBOX=1` without running `init`.

### OAuth app registration (only needed for `init --oauth`)

One-time at https://i.tochka.com/bank/services/m/integration/new — required for the Invoice API, closing documents, and the full Open Banking statement. Verified pitfalls (2026-04-20):

- **`Redirect URL` must be HTTPS.** Plain `http://` is rejected.
- **`localhost` is NOT accepted** in the Redirect URL field (despite what Tochka's own example docs show). Use `127.0.0.1` or a real domain. The form silently rejects `https://localhost:8443/callback` but accepts `https://127.0.0.1:8443/callback`.
- Recommended template: `https://127.0.0.1:8443/callback`. The port can be any free one; `/callback` is arbitrary but fixed once registered.
- `client_secret` **cannot be re-viewed** after closing the registration form — if you lose it, you must regenerate or recreate the app.

`init --oauth` defaults to `https://127.0.0.1:8443/callback`. Pass `--redirect-url <full URL>` if your registered URL is different (missing this flag = `invalid_redirect_uri` at the browser stage, then a silent 5-min callback-server timeout — always pre-check the match). The wizard starts a local HTTPS server (`mkcert` if installed, otherwise self-signed — the browser will warn, click through), opens the browser at `connect/authorize`, exchanges the code, and stores `client_id` / `client_secret` / `refresh_token` in Keychain.

Refresh token lifetime is 30 days. If nothing calls an OAuth endpoint for that long, the token dies silently — symptom is 401 on `list-statement` / `create-invoice`. Fix: re-run `init --oauth`.
