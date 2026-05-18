# Tochka Bank API — Endpoints catalogue

Full request/response schemas and validation rules. All paths use `v2.0` where the bank offers it (Open Banking, Acquiring, Payment, SBP). Invoice API: `v1.0`/`v2.0` on prod, both **501 with personal JWT**. Closing docs — `v1.0` only.

Source of truth: **live ReDoc https://enter.tochka.com/doc/v2/redoc** — Tochka rotates paths and field names without semver. Schemas here are correct as of 2026-04-20.

Base: `https://enter.tochka.com/uapi` (prod) or `https://enter.tochka.com/sandbox/v2`.

Headers: `Authorization: Bearer <token>`, `Content-Type: application/json` for POST/PUT.

## Accounts

```
GET /open-banking/v2.0/accounts
```

Returns `Data.Account[]` with `accountId`, `customerCode`, `currency`, `accountType`, `accountSubType`, `status`. `customerCode` is Tochka's internal ID for your legal entity / IE — needed almost everywhere; save it once. Example (prod, redacted):

```json
{"Data": {"Account": [{"customerCode": "100000001",
  "accountId": "40702810900000000001/044525225",
  "status": "Enabled", "currency": "RUB",
  "accountType": "Business", "accountSubType": "CurrentAccount"}]}}
```

## Incoming payments (acquiring / SBP)

```
GET /acquiring/v2.0/payments?customerCode={code}
```

Payments received via Tochka's merchant tools (SBP QR, pay-by-link, internet acquiring). Optional query params: `status`, `fromDate`, `toDate`, `page`, `perPage` (default 1000).

Response — `Data.Operation[]` with `paymentType` (`sbp` / `card`), `paymentId`, `transactionId`, `createdAt`/`paidAt`, `purpose`, `amount` (float rubles), `status`, `operationId`, `paymentLink`, `consumerId`.

Observed statuses (v2.0): `APPROVED`, `REJECTED`, `PENDING`, `REFUNDED`. Full `AcquiringPaymentStatus` enum (v1.0): `CREATED`, `APPROVED`, `ON-REFUND`, `REFUNDED`, `EXPIRED`, `REFUNDED_PARTIALLY`, `AUTHORIZED`, `WAIT_FULL_PAYMENT`. v2.0 additionally surfaces `PENDING` and `REJECTED` (not in the v1.0 enum).

**Does not include** wire transfers to your settlement account from non-merchant sources — those only show up in the full Open Banking statement (OAuth).

## Outgoing payment orders — «На подпись» queue

### List drafts
```
GET /payment/v2.0/for-sign?customerCode={code}
```

Payments currently in the «На подпись» (awaiting-signature) queue, regardless of whether they were created via API or manually in online banking. Once signed and dispatched, they **drop out of this list** — personal JWT has no access to executed payments.

Optional `status`. Observed for drafts: `Created`, `AwaitingSignature`.

### Create payment order
```
POST /payment/v2.0/for-sign
```

**Body shape — flat `Data`** (verified prod 2026-04-20). Older docs and agent-generated examples show `Data.Payment: [...]` (array) or `Data.Payment: {...}` (nested object) — **both are wrong**; the API ignores those fields and returns `"Field X: Field required"` for every required field.

```json
{
  "Data": {
    "accountCode": "40702810900000000001",
    "bankCode": "044525225",
    "counterpartyAccountNumber": "40702810400001234567",
    "counterpartyBankBic": "044525225",
    "counterpartyINN": "7707083893",
    "counterpartyKPP": "773601001",
    "counterpartyName": "ПАО Сбербанк",
    "paymentAmount": 15000.00,
    "paymentDate": "2026-04-20",
    "paymentNumber": 42,
    "paymentPriority": "5",
    "paymentPurpose": "Оплата по счёту №7 от 18.04.2026, без НДС",
    "supplierBillId": "0",
    "taxInfo": {"status":"0","kbk":"0","oktmo":"0","reasonCode":"0",
                "taxPeriod":"0","documentNumber":"0","documentDate":"0","payerStatus":"0"}
  }
}
```

Required: `accountCode`, `bankCode`, `counterpartyBankBic`, `counterpartyAccountNumber`, `counterpartyName`, `paymentAmount`, `paymentDate`, `paymentPurpose`, plus `paymentNumber`, `paymentPriority`, `supplierBillId`, `taxInfo`, `counterpartyINN`, `counterpartyKPP` for company counterparties.

**Field constraints (swagger `PaymentForSignRequestModel`):**

| Field | Constraint |
|---|---|
| `accountCode` | exactly 20 chars |
| `bankCode` | exactly 9 chars (your bank's БИК) |
| `counterpartyAccountNumber` | exactly 20 chars |
| `counterpartyBankBic` | exactly 9 chars |
| `counterpartyINN` | 10–12 chars (10 for legal entity, 12 for IE) |
| `counterpartyKPP` | up to 9 chars (`"0"` for IE / individuals) |
| `paymentNumber` | integer, 1–999999 |
| `paymentPurpose` | up to 210 chars |
| `paymentDate` | `date` format (`YYYY-MM-DD`, not datetime) |

Optional (per swagger): `counterpartyBankCorrAccount` (20 chars), `email` (bank will send a PDF), `codePurpose` (field 20 of the payment purpose, string `1–5` or empty — for transfers to individuals on certain account types), `payerINN` / `payerKPP` (if paying on behalf of a third party).

For non-tax payments, the `taxInfo` block must still be present with all-`"0"` placeholders (schema requirement). For tax payments, fill the actual КБК / ОКТМО / period.

**Verified validation rules (prod):**
- `paymentDate` ≤ today (`"Value error, should be less than today or equal"` for future dates).
- `paymentPurpose` can't contain em-dash `—` (U+2014) (`"Value error, forbidden symbols: —"`). Hyphen `-` or comma. Cyrillic letters are fine.
- `counterpartyBankBic` + `counterpartyAccountNumber` are validated against the Russian Central Bank registry. Fake combos return `"Проверьте номер счёта — в выбранном банке такого счёта нет"`.
- `accountCode` = the 20-digit account number alone, without the BIC-slash suffix from `accountId`. Split `accountId = "40702810900000000001/044525225"` → `accountCode = "40702810900000000001"`, `bankCode = "044525225"`.

### Signing
The API creates a *draft* in the «На подпись» queue. Actual signing and dispatch go through mobile / online banking with SMS, unless the JWT carries `SignPaymentOrder` (rare, requires a hardware-key setup with Tochka).

### Get payment status
```
GET /payment/v2.0/status/{requestId}
```
Caveat from docs: "If a payment was edited in online banking after API creation, the real status can't be determined through this method — it returns `Created`."

Permissions: `CreatePaymentForSign` (draft), `SignPaymentOrder` (auto-sign, rare).

## Full bank statement (Open Banking)

> ⚠️ **501 with personal JWT on prod.** OAuth + Consent is required ([auth.md](auth.md#oauth-20--consent-flow-multi-tenant)). Sandbox works directly with `working_token`.

### 1. Create statement (async)
```
POST /open-banking/v2.0/statements
```
```json
{"Data": {"Statement": {
  "accountId": "40702810900000000001/044525225",
  "startDateTime": "2026-03-21",
  "endDateTime": "2026-04-20"}}}
```

Gotchas:
- `startDateTime` / `endDateTime` — plain `YYYY-MM-DD`. ISO datetime with non-zero time is rejected.
- OAuth requests also need header `CustomerCode: <your customerCode>`.

Returns `Data.Statement.statementId` + `status: "Created"`. Usually ready in seconds; up to 24h on long ranges.

Full `StatementStatus` enum: `Created` → `Processing` → `Ready` (success) or `Error` (terminal). Poll until `Ready` / `Error`; don't hang on `Processing` indefinitely.

### 2. Poll via list
```
GET /open-banking/v2.0/statements
```
Returns ALL statements for the customer. **Find yours by `statementId`** and wait for `status: "Ready"`. A per-id status endpoint `GET /statements/{statementId}/status` exists, but the list is simpler and includes the payload as soon as it's ready.

### 3. Transactions inline in the response

When `status == "Ready"`, `Transaction[]` is already nested inside the Statement object. There's **no** separate `/accounts/{id}/statements/{id}/payments` endpoint — that path returns 501 on prod. The field is called **`Transaction`** (singular, an array), not `Payment`.

Example transaction (prod, redacted):
```json
{"Amount": {"amount": 195.6, "currency": "RUB", "amountNat": 195.6},
 "status": "Booked",
 "paymentId": "cbs-tb-XX-0000000000",
 "transactionId": "cbs-tb;0000000000;1",
 "documentNumber": "12345",
 "documentProcessDate": "2026-04-20",
 "transactionTypeCode": "Платежное поручение",
 "creditDebitIndicator": "Credit",
 "description": "Оплата по договору. Без НДС",
 "DebtorAgent": {"name": "ООО \"Банк-Пример\"", "identification": "044525999"},
 "DebtorParty": {"inn": "7700000000", "kpp": "770001001", "name": "ООО \"Контрагент-Пример\""},
 "DebtorAccount": {"identification": "40702810000000000000"}}
```

Field reference (**not** OBIE-named, despite what older guides claim):

| Field | Meaning |
|---|---|
| `creditDebitIndicator` | `Credit` (incoming) or `Debit` (outgoing) |
| `Amount.amount` | in rubles (not kopecks) |
| `description` | free-text payment purpose (NOT `purpose` / `paymentPurpose`) |
| `documentProcessDate` | ISO date of booking |
| `documentNumber` | bank document number |
| `transactionTypeCode` | Cyrillic: `"Платежное поручение"`, `"Банковский ордер"` |
| `DebtorParty.{name,inn,kpp}` | for Credit — who sent the money |
| `CreditorParty.{name,inn,kpp}` | for Debit — who received it |
| `DebtorAccount.identification` / `CreditorAccount.identification` | counterparty's account number |
| `status` | `Booked` (posted) or `Pending` (provisional) |
| `TaxFields` | optional for tax payments: `originatorStatus`, `kbk`, `oktmo`, `base`, `documentNumber`, `documentDate`, `type`, `field107` |
| `paymentId`, `transactionId` | bank-internal IDs |

The Statement wrapper has `startDateBalance` / `endDateBalance` (period-bracket balances).

## IE vs LLC — field differences in invoices and payment orders

`SecondSide.type` (invoices, closing docs) and counterparty requisites (payment orders):

| Field | LLC / JSC (`type: "company"`) | IE (`type: "ip"`) |
|---|---|---|
| `taxCode` (ИНН) | 10 digits | **12 digits** |
| `KPP` | required | **not sent** (or `"0"` in payment order) |
| `legalName` | `"ООО \"Name\""` | `"ИП Last First Middle"` |
| Typical `ndsKind` | `nds_22` (general taxation) or `without_nds` (USN) | `without_nds` (USN without VAT preference), `nds_5` / `nds_7` (USN with preference), `nds_22` (general) |
| `taxInfo` in payment order | for budget payments | same; an IE on USN pays contributions and tax through the same fields (different КБК codes) |

For a token holder = **IE on USN**, Tochka imposes no special endpoint restrictions — IE customers use the same JWT/OAuth scheme as legal entities. Typical use case: statement + invoices with `ndsKind: "without_nds"`. Preference rates `nds_5` / `nds_7` apply only if the IE explicitly opted into USN-with-VAT.

The old value `"entrepreneur"` (instead of `"ip"`) is no longer accepted.

## Invoices (OAuth-only on prod)

> ⚠️ **501 with personal JWT on prod.** Works in sandbox (handy for schema dev/debug), but OAuth+Consent on prod.

> ⚠️ **No list endpoint.** `GET /invoice/v2.0/bills` without an id — doesn't exist. Any of `/bills`, `/bills-list`, `/invoices`, `/documents`, `/outgoing`, `/my` returns 501. Invoices are only accessible by a known `documentId` (returned from `POST /bills`, or copied manually from online banking). If an invoice was created via the web UI and `documentId` is unknown — **you can't find it via API**. Confirmed 2026-04-20.

### Create
```
POST /invoice/v2.0/bills
```

> **v1.0 vs v2.0 schema split.** Swagger v1.90.4-stable documents only `/invoice/v1.0/bills` with different field names: `SecondSide.secondSideName` (not `legalName`), `SecondSide.kpp` (not `KPP`), `Positions[].positionName` (not `name`), and `Content.Invoice.{number,date,paymentExpiryDate}` — number/date/due-date are **nested inside `Content.Invoice`**. On prod the v2.0 shape works (flat top-level `documentNumber` / `documentDate` / `paymentExpirationDate`, capitalised `legalName` / `KPP`, `Positions[].name`) — that's what this skill uses. If you ever hit v1.0 directly, switch to the swagger-canonical names.

Body (exact shape — capitalisation matters):
```json
{
  "Data": {
    "customerCode": "100000001",
    "accountId": "40702810900000000001/044525225",
    "documentDate": "2026-04-20",
    "documentNumber": "INV-2026-001",
    "paymentExpirationDate": "2026-05-20",
    "SecondSide": {"taxCode": "7700000000", "KPP": "770001001",
                   "type": "company", "legalName": "ООО \"Покупатель\""},
    "Content": {"Invoice": {
      "number": "INV-2026-001", "totalAmount": 50000.00,
      "Positions": [{"name": "Консультационные услуги", "price": 50000.00,
        "quantity": 1, "totalAmount": 50000.00,
        "unitCode": "услуга.", "ndsKind": "without_nds"}]}}
  }
}
```

Fields:
- `SecondSide.type`: `"company"` (LLC / JSC — 10-digit `taxCode` + `KPP`) or `"ip"` (IE — 12-digit `taxCode`, no KPP). The old `"entrepreneur"` is rejected.
- `Positions[].unitCode` — Russian short form with a trailing dot (not the OKEI code). Accepted: `'шт.'`, `'тыс.шт.'`, `'компл.'`, `'пар.'`, `'усл.ед.'`, `'упак.'`, `'услуга.'`, `'пач.'`, `'мин.'`, `'ч.'`, `'сут.'`, `'г.'`, `'кг.'`, `'л.'`, `'м.'`, `'м2.'`, `'м3.'`, `'км.'`, `'га.'`, `'кВт.'`, `'кВт.ч.'`.
- `Positions[].ndsKind` (2026 VAT rates):
  - `without_nds` — no VAT (IE on USN without the VAT preference; or art. 145 NK exemption)
  - `nds_0` — 0% (exports, special operations)
  - `nds_5` — 5% (USN preference for revenue 20–272.5M ₽/year, no deduction)
  - `nds_7` — 7% (USN preference for revenue 272.5–490.5M ₽/year, no deduction)
  - `nds_10` — 10% (children's goods, groceries, medicines, books)
  - `nds_22` — 22% (standard since 2026-01-01, replaced 20%)

The old enum (`vat_20`, `vat_10`, `without_vat`) is no longer accepted.

**Position constraints (swagger `PositionModel`):**
- `price`: number, `>= 0`
- `quantity`: number, `0 < q < 10000000` (not zero)
- `totalAmount`: number, `>= 0`
- `totalNds` — optional, VAT per position (server usually computes it)

Optional `Content.Invoice`: `basedOn` (free-text «на основании»), `comment`, `totalNds`.

Response: `Data.documentId`.

### Get PDF
```
GET /invoice/v1.0/bills/{customerCode}/{documentId}/file
```
`application/pdf` binary (~50 KB). **Can take 30+ seconds on the first call** (server renders on demand) — set timeout ≥ 60s.

### Check payment status
```
GET /invoice/v1.0/bills/{customerCode}/{documentId}/payment-status
GET /invoice/v2.0/bills/{customerCode}/{documentId}/payment-status
```
```json
{"Data": {"paymentStatus": "payment_paid"}}
```
Swagger `PaymentStatusEnum`: exactly three values — `payment_waiting`, `payment_paid`, `payment_expired`. Older speculation about `payment_cancelled` — **not in the API**. Deleting an invoice manually in online banking simply DELETEs it; there's no cancelled-paid state.

### Send to email
```
POST /invoice/v1.0/bills/{customerCode}/{documentId}/email
```
Body: `{"Data": {"email": "buyer@example.com"}}`

The old `/send-to-email` suffix returns 404. Swagger uses the bare `/email`. v2.0 isn't documented for this operation — stay on v1.0.

### Full Invoice API — exactly 5 operations

**All take `customerCode` in the path** (not just in the body):

| Verb | Path | Purpose |
|---|---|---|
| POST | `/invoice/v2.0/bills` | Create (returns `documentId`) |
| DELETE | `/invoice/v2.0/bills/{customerCode}/{documentId}` | Delete |
| POST | `/invoice/v1.0/bills/{customerCode}/{documentId}/email` | Send to email |
| GET | `/invoice/v1.0/bills/{customerCode}/{documentId}/file` | Get rendered PDF |
| GET | `/invoice/v1.0/bills/{customerCode}/{documentId}/payment-status` | Payment status |

**Pro-tip** for finding `documentId` of an invoice created via the web UI: open the invoice in online banking at https://i.tochka.com/bank/m/document_flow/document/{documentId} — the UUID in the URL IS the `documentId` you need for API calls.

Older guides and agent guesses used `/invoice/v2.0/bills/{documentId}/*` (without customerCode) — **wrong**, returns 501 on prod. `customerCode` in the path is mandatory for per-invoice operations.

## SBP QR codes

### Register retailer (one-time)
```
POST /sbp/v2.0/register-retailer/account/{accountId}
```
Returns `merchantId`.

### Register QR code
```
POST /sbp/v2.0/qr-code/merchant/{merchantId}/account/{accountId}
```
```json
{"Data": {"qrcType": "02", "amount": 150000, "currency": "RUB",
          "paymentPurpose": "Заказ №1234"}}
```

`qrcType`: `"01"` static, `"02"` dynamic.

**⚠️ `amount` is in kopecks, not rubles.** Swagger `RegisterQRCode` titles the field "Сумма в копейках" explicitly. 1500 ₽ → `150000`. Different from acquiring `/payments` where `amount` is float rubles — easy to confuse. Also `paymentPurpose` in the SBP QR has `maxLength: 140` (tighter than the 210 for payment orders).

**SBP NSPK v9.1 (since April 2026):** `bankCode` (БИК) is a required parameter for retailer registration and some SBP endpoints. The `incomingSbpPayment` webhook signature also changed — refresh JWKS before verification.

### List QR codes
```
GET /sbp/v2.0/qr-codes/{merchantId}
```

Permissions: `EditSBPData` (write), `ReadSBPData` (read). ReDoc: https://enter.tochka.com/doc/v2/redoc#tag/SBP-API.

## Closing documents (акты / УПД / ТОРГ-12 / счёт-фактура)

OAuth-gated, same `ManageInvoiceData` permission as invoices. Natural follow-up once an invoice is paid.

| Verb | Path | Purpose |
|---|---|---|
| POST | `/invoice/v1.0/closing-documents` | Create act / УПД / ТОРГ-12 / счёт-фактура |
| GET | `/invoice/v1.0/closing-documents/{customerCode}/{documentId}/file` | Download PDF |
| POST | `/invoice/v1.0/closing-documents/{customerCode}/{documentId}/email` | Send to email |
| DELETE | `/invoice/v1.0/closing-documents/{customerCode}/{documentId}` | Delete |

Body (swagger `ClosingDocumentCreateRequestModel`):
```json
{
  "Data": {
    "customerCode": "100000001",
    "accountId": "40702810900000000001/044525225",
    "SecondSide": {"taxCode": "7700000000", "kpp": "770001001", "type": "company", "secondSideName": "ООО \"Покупатель\""},
    "Content": {
      "Act": {
        "date": "2026-04-30",
        "number": "A-1",
        "totalAmount": 50000,
        "Positions": [{"name": "Услуга", "price": 50000, "quantity": 1, "totalAmount": 50000, "unitCode": "шт.", "ndsKind": "without_nds"}]
      }
    },
    "documentId": "<optional parent-invoice documentId>"
  }
}
```

`Content` is a discriminated union — one of `Act` (акт выполненных работ), `PackingList` (ТОРГ-12), `Invoicef` (счёт-фактура), `Upd` (УПД). Position shape matches invoice `PositionModel`. Optional `documentId` links the closing doc to a parent invoice — online banking groups them in one thread.

**⚠️ Critical v1.0 vs v2.0 shape differences (verified on prod 2026-04-21):**
- **`SecondSide` uses lowercase `kpp` and `secondSideName`** — NOT the v2.0 invoice shape (`KPP` / `legalName`). If you send `KPP` here, Tochka silently drops it → buyer КПП missing in the rendered PDF → ЭДО signing fails with `"Не получилось подписать документ. Проверьте ИНН или КПП контрагента."` The doc still gets a `documentId` at creation time, so the bug only surfaces at signing.
- **`Content.Act.totalAmount` is required** at the block level (not just inside each position). Missing it returns `400 Validation Error: Field Content-ContentAct-Act-totalAmount : Field required`.

Helpers: `create-closing-doc`, `get-closing-doc`, `send-closing-doc`, `delete-closing-doc`.

## Balances

```
GET /open-banking/v1.0/balances
GET /open-banking/v1.0/accounts/{accountId}/balances
```

Live snapshot. Works with personal JWT (`ReadBalances`). The `accountId` slash in the URL must be encoded as `%2F`.

Helpers: `get-balance` (all accounts) / `get-balance --account-id …` (one).

## Webhooks

Types (swagger `WebhookTypeEnum`): `incomingPayment`, `outgoingPayment`, `incomingSbpPayment`, `acquiringInternetPayment`, `incomingSbpB2BPayment`.

| Verb | Path | Purpose |
|---|---|---|
| PUT | `/webhook/v1.0/{client_id}` | Register URL + events |
| POST | `/webhook/v1.0/{client_id}` | Edit existing |
| GET | `/webhook/v1.0/{client_id}` | Get config |
| DELETE | `/webhook/v1.0/{client_id}` | Delete |
| POST | `/webhook/v1.0/{client_id}/test_send` | Test event to the registered URL |

PUT body: `{"webhooksList": ["incomingPayment", "incomingSbpPayment"], "url": "https://your-endpoint/hook"}`. `url` must be `https://`, max 2083 chars. Requires `ManageWebhookData`.

`{client_id}` is the OAuth app `client_id` (for personal JWT — the `client_id` shown when the JWT was generated in online banking). Use `test_send` immediately after registering to confirm the receiver is wired up.

## Payment links (internet acquiring)

```
POST /acquiring/v1.0/payments                  # plain
POST /acquiring/v1.0/payments_with_receipt     # with fiscal receipt (ОФД)
GET  /acquiring/v1.0/payments/{operationId}
POST /acquiring/v1.0/payments/{operationId}/capture    # two-stage capture
POST /acquiring/v1.0/payments/{operationId}/refund     # full/partial
```

Required (plain): `customerCode` (9 chars), `amount` (rubles — NOT kopecks!), `purpose`, `paymentMode` (`["sbp", "card"]`). Optional: `merchantId`, `paymentLinkId` (your external ref), `redirectUrl`, `failRedirectUrl`, `ttl` (1–44640 min, default 10080 = 7 days), `preAuthorization` (`true` → must call `/capture` to settle), `saveCard`, `consumerId`.

Returns `Data.Operation.{operationId, paymentLink}` — share `paymentLink` with the client. Requires `MakeAcquiringOperation`.

Helper: `create-payment-link`. For pre-auth flows the skill does NOT auto-capture — call `/capture` manually when shipping the goods.

## Acquiring daily registry

```
GET /acquiring/v1.0/registry?customerCode=...&merchantId=...&date=YYYY-MM-DD
```

Daily settlement roll-up (реестр интернет-эквайринга): totals, commissions, net transferred. Useful for month-end reconciliation against the bank statement. Optional `paymentId` for drill-down. Requires `ReadAcquiringData`.

Helper: `list-registry --date YYYY-MM-DD`.

## Consents (OAuth introspection)

```
GET  /consent/v1.0/consents                      # all consents for the app token
GET  /consent/v1.0/consents/{consentId}          # single consent details
GET  /consent/v1.0/consents/{consentId}/child    # child consents (multi-account)
POST /consent/v1.0/consents                      # create new (setup)
```

Read-only calls. Useful when OAuth starts returning 403 — check the active consent's scopes before assuming token expiry.

Helpers: `list-consents`, `get-consent <consentId>`.

## accountId format — critical pitfall

`accountId` is NOT a plain account number. Format: `{20-digit-account}/{9-digit-BIC}` — with an embedded forward slash:

```
"accountId": "40702810900000000001/044525225"
```

In request **bodies** pass it as-is. In **URLs** the slash must be URL-encoded as `%2F`:

```
GET /open-banking/v2.0/accounts/40702810900000000001%2F044525225/statements/{statementId}/payments
```

Forget to encode it — `404 HTTPNotFound: Not found statement under account`.

## Operational pitfalls

Not error messages — architectural caveats to factor into planning:

- **No idempotency keys:** payment endpoints don't accept `Idempotency-Key`. Track your own `paymentNumber` / external IDs to avoid duplicates on retry.
- **Cyrillic in responses:** some fields and enum values are Cyrillic strings (`"Исполнен"`, `unitCode` `"услуга."`). Match on the exact string from ReDoc, not a translation.
- **Sandbox ≠ prod:** sandbox accepts `working_token` for everything (including Invoice + Open Banking). Prod splits by auth tier. Schema debugging → sandbox; access verification → prod with a low-scope token.
- **Divergence from Russian Central Bank Open Banking standards:** the current Tochka API does **not** conform to the central bank's approved AFT/OBR standards (wiki.openbankingrussia.ru, wiki.opendatarussia.ru) — neither Accounts v2.0/v3.0, nor Payment Initiation v1.0.0, nor FAPI Advanced v2.0. The nearest watershed is **2026-10-01** (the introduction date of the v2.0 CB standards package): expect a possible OAuth-flow reissue and field-name changes in `/accounts` / `/payment`. Don't copy OBR/OBIE examples assuming they'll work against Tochka.
