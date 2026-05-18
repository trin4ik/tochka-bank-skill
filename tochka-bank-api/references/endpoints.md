# Tochka Bank API — Endpoints catalogue

Reference for the most useful endpoints. All paths use **v2.0** where the bank offers it (Open Banking, Acquiring, Payment, SBP). Invoice API has v1.0/v2.0 on prod, both 501 for personal JWT.

Always confirm against the live ReDoc at https://enter.tochka.com/doc/v2/redoc — paths and field names occasionally change.

## Table of Contents

- [What works with personal JWT vs. what needs OAuth](#what-works-with-personal-jwt-vs-what-needs-oauth)
- [Accounts](#accounts)
- [Incoming payments (acquiring / SBP)](#incoming-payments-acquiring--sbp)
- [Outgoing payment orders — "На подпись"](#outgoing-payment-orders--на-подпись)
- [Full bank statement (Open Banking)](#full-bank-statement-open-banking)
- [Invoices (OAuth-only on prod)](#invoices-oauth-only-on-prod)
- [SBP QR codes](#sbp-qr-codes)
- [Closing documents (акты / УПД / накладные / счета-фактуры)](#closing-documents-акты--упд--накладные--счета-фактуры)
- [Balances (текущий баланс)](#balances-текущий-баланс)
- [Webhooks](#webhooks)
- [Payment links (интернет-эквайринг)](#payment-links-интернет-эквайринг)
- [Acquiring daily registry](#acquiring-daily-registry)
- [Consents (OAuth introspection)](#consents-oauth-introspection)
- [accountId format — critical pitfall](#accountid-format--critical-pitfall)

All paths relative to `https://enter.tochka.com/uapi` (prod) or `https://enter.tochka.com/sandbox/v2` (sandbox).

Headers:
```
Authorization: Bearer <token>
Content-Type: application/json     # for POST/PUT
```

## What works with personal JWT vs. what needs OAuth

**Verified on prod 2026-04-20** against a personal JWT generated via ЛК → «Интеграции и API» → «Сгенерировать JWT-ключ»:

| Area | Endpoint | Personal JWT | Notes |
|------|----------|:---:|---|
| Accounts | `GET /open-banking/v2.0/accounts` | ✅ | returns all accounts + customerCode |
| Incoming acquiring | `GET /acquiring/v2.0/payments?customerCode=X` | ✅ | SBP + card payments via tochka merchant |
| Outgoing drafts | `GET /payment/v2.0/for-sign?customerCode=X` | ✅ | only payments currently in "На подпись" |
| Payment status | `GET /payment/v2.0/status/{requestId}` | ✅ | per-order status |
| Create payment order | `POST /payment/v2.0/for-sign` | ✅* | *needs `CreatePaymentForSign` scope |
| **Full statement** | `POST /open-banking/v2.0/statements` | ❌ 501 | **needs OAuth + Consent** |
| **Invoices** | `POST /invoice/v2.0/bills` | ❌ 501 | **needs OAuth + Consent** on prod (OK in sandbox) |
| SBP merchant | `POST /sbp/v2.0/qr-code/legal-entity` | ✅* | *needs `EditSBPData` |

Personal JWT **does NOT** expose signed/executed payment orders (they fall out of `/for-sign` once signed) and **does NOT** expose the full account statement. Anything beyond the table requires OAuth 2.0 + Consent — see [auth.md](auth.md#oauth-20--consent-flow-multi-tenant).

## Accounts

### List accounts
```
GET /open-banking/v2.0/accounts
```
Returns `Data.Account[]` with `accountId`, `customerCode`, `currency`, `accountType`, `accountSubType`, `status`.

The `customerCode` is Tochka's internal ID for your legal entity/ИП — you'll need it for almost every other endpoint. Save it once from this response; it doesn't change.

Example response (verified prod, redacted):
```json
{
  "Data": {
    "Account": [{
      "customerCode": "100000001",
      "accountId": "40702810900000000001/044525225",
      "status": "Enabled",
      "currency": "RUB",
      "accountType": "Business",
      "accountSubType": "CurrentAccount",
      "registrationDate": "2025-01-15"
    }]
  }
}
```

## Incoming payments (acquiring / SBP)

Payments received via Tochka's merchant tools (SBP QR, pay-by-link, internet acquiring).

### List incoming payments
```
GET /acquiring/v2.0/payments?customerCode={code}
```

Optional query params: `status` (e.g. `APPROVED`, `REJECTED`), `fromDate`, `toDate`, `page`, `perPage` (default 1000 per swagger v1.90.4-stable).

Verified-working response (prod):
```json
{
  "Data": {
    "Operation": [{
      "customerCode": "100000001",
      "paymentType": "sbp",
      "paymentId": "B0000000...",
      "transactionId": "00000000-0000-0000-0000-000000000000",
      "createdAt": "2026-02-10T14:09:30+05:00",
      "paidAt": "2026-02-10T14:50:24+03:00",
      "purpose": "Оплата услуг по договору",
      "amount": 1000.0,
      "status": "APPROVED",
      "operationId": "00000000-0000-0000-0000-000000000000",
      "paymentLink": "https://merch.tochka.com/order/?uuid=...",
      "consumerId": "00000000-0000-0000-0000-000000000000",
      "paymentLinkId": "1"
    }]
  }
}
```

Observed statuses on prod (v2.0): `APPROVED`, `REJECTED`, `PENDING`, `REFUNDED`.

Full `AcquiringPaymentStatus` enum per swagger (v1.0): `CREATED`, `APPROVED`, `ON-REFUND`, `REFUNDED`, `EXPIRED`, `REFUNDED_PARTIALLY`, `AUTHORIZED`, `WAIT_FULL_PAYMENT`. The v2.0 stream additionally surfaces `PENDING` and `REJECTED` (not in the v1.0 enum).

Does **not** include wire transfers to your settlement account from non-merchant sources — those are only in the full Open Banking statement (OAuth).

## Outgoing payment orders — "На подпись"

### List drafts awaiting signature
```
GET /payment/v2.0/for-sign?customerCode={code}
```

Returns payments currently in the "На подпись" section of online banking — regardless of whether they were created via API or manually in the ЛК. Once a payment is signed and dispatched, it drops out of this list (there is no "executed list" accessible to a personal JWT).

Optional `status` query param. Observed for drafts: `Created`, `AwaitingSignature`.

### Create a payment order
```
POST /payment/v2.0/for-sign
```

**Body shape — flat `Data` object** (verified prod 2026-04-20). Older docs and agent-generated examples show `Data.Payment: [...]` (array) or `Data.Payment: {...}` (nested object) — **both are wrong**, the API ignores the fields and returns `"Field X : Field required"` for every mandatory field.

Correct payload:
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
    "taxInfo": {
      "status": "0", "kbk": "0", "oktmo": "0", "reasonCode": "0",
      "taxPeriod": "0", "documentNumber": "0", "documentDate": "0", "payerStatus": "0"
    }
  }
}
```

Required fields (from validation errors): `accountCode`, `bankCode`, `counterpartyBankBic`, `counterpartyAccountNumber`, `counterpartyName`, `paymentAmount`, `paymentDate`, `paymentPurpose` (also `paymentNumber`, `paymentPriority`, `supplierBillId`, `taxInfo`, `counterpartyINN`, `counterpartyKPP` for company counterparties).

**Field-level constraints (from swagger `PaymentForSignRequestModel`):**

| Field | Type / constraint |
|-------|-------------------|
| `accountCode` | exactly 20 chars |
| `bankCode` | exactly 9 chars (БИК своего банка) |
| `counterpartyAccountNumber` | exactly 20 chars |
| `counterpartyBankBic` | exactly 9 chars |
| `counterpartyINN` | 10–12 chars (10 для юр-лица, 12 для ИП) |
| `counterpartyKPP` | up to 9 chars (use `"0"` для ИП/физлиц) |
| `paymentNumber` | integer, `0 < n < 1000000` (range 1–999999) |
| `paymentPurpose` | string, up to 210 chars |
| `paymentDate` | format `date` (ISO `YYYY-MM-DD`, not datetime) |

**Optional fields (per swagger, not commonly used):**
- `counterpartyBankCorrAccount` — корреспондентский счёт банка получателя (20 chars)
- `email` — адрес, на который банк пришлёт PDF платёжки после создания
- `codePurpose` — поле 20 назначения платежа, строка `1–5` или пусто (для переводов физлицам на определённые типы счетов)
- `payerINN` / `payerKPP` — реквизиты фактического плательщика, если платите за третье лицо

For non-tax payments the `taxInfo` block must still be present with all-`"0"` placeholders (required by schema). For tax payments fill actual КБК/ОКТМО/period.

**Verified validation rules (prod):**

- `paymentDate` must be **≤ today**. API rejects future dates with `"Value error, should be less than today or equal"`. For deferred payments, submit draft and let human sign on the target date.
- `paymentPurpose` **cannot contain long dash `—`** (U+2014). API rejects with `"Value error, forbidden symbols: —"`. Use regular hyphen `-` or comma. Likely other non-ASCII punctuation is also restricted — Cyrillic letters OK.
- `counterpartyBankBic` + `counterpartyAccountNumber` are validated against the **ЦБ registry**. Fake account numbers return `"Проверьте номер счёта — в выбранном банке такого счёта нет"`. Use real counterparty reqs.
- `accountCode` is the **20-digit account number alone** (without the BIC-slash suffix used in `accountId`). Split `accountId = "40702810900000000001/044525225"` into `accountCode = "40702810900000000001"` and `bankCode = "044525225"`.

### Signing

The API creates a *draft* in "На подпись". Actual signing and dispatch happen via mobile/online-banking with SMS confirmation, unless the JWT holds `SignPaymentOrder` (rare, requires hardware-key setup with Tochka).

### Get payment status
```
GET /payment/v2.0/status/{requestId}
```

**Caveat from docs:** "If a payment was edited in internet-banking after API creation, actual status cannot be determined via this method — it displays as `Created`."

### Permissions
- `CreatePaymentForSign` — draft outgoing
- `SignPaymentOrder` — auto-sign (rare)

## Full bank statement (Open Banking)

> ⚠️ **501 with a personal JWT on prod.** Requires OAuth 2.0 + Consent (see [auth.md](auth.md#oauth-20--consent-flow-multi-tenant)). Sandbox supports it directly via `working_token`.

**Verified flow on prod with OAuth (2026-04-20):**

### 1. Create a statement (async)
```
POST /open-banking/v2.0/statements
```
Body:
```json
{
  "Data": {
    "Statement": {
      "accountId": "40702810900000000001/044525225",
      "startDateTime": "2026-03-21",
      "endDateTime": "2026-04-20"
    }
  }
}
```

**Gotchas:**
- `startDateTime` / `endDateTime` must be plain `YYYY-MM-DD` — ISO datetime with non-zero time is rejected.
- OAuth requests also need header `CustomerCode: <your customerCode>`.

Returns `Data.Statement.statementId` and `status: "Created"`. Processing normally completes in seconds (can take up to 24h for long ranges).

Full `StatementStatus` enum per swagger: `Created` → `Processing` → `Ready` (success path) or `Error` (terminal failure). Poll until `Ready` or `Error`; don't wait on `Processing`/`Created` forever — surface failures so the caller knows to retry.

### 2. Poll by fetching the whole statements list
```
GET /open-banking/v2.0/statements
```
Returns ALL statements on record for the customer. **Find yours by `statementId`** and watch for `status: "Ready"`. A single per-id status endpoint `GET /statements/{statementId}/status` exists but the list endpoint is simpler and also returns the full payload when ready — no separate fetch step needed.

### 3. Extract transactions — inline in the list response

When `status == "Ready"`, `Transaction[]` is already embedded in the Statement object. There is **no** separate `/accounts/{id}/statements/{id}/payments` endpoint — that path returns 501 on prod. The field is called **`Transaction`** (singular, an array), not `Payment`.

Example entry (real prod data, redacted):
```json
{
  "Amount": { "amount": 195.6, "currency": "RUB", "amountNat": 195.6 },
  "status": "Booked",
  "paymentId": "cbs-tb-XX-0000000000",
  "transactionId": "cbs-tb;0000000000;1",
  "documentNumber": "12345",
  "documentProcessDate": "2026-04-20",
  "transactionTypeCode": "Платежное поручение",
  "creditDebitIndicator": "Credit",
  "description": "Оплата по договору. Без НДС",
  "DebtorAgent": { "name": "ООО \"Банк-Пример\"", "identification": "044525999" },
  "DebtorParty": { "inn": "7700000000", "kpp": "770001001", "name": "ООО \"Контрагент-Пример\"" },
  "DebtorAccount": { "identification": "40702810000000000000" }
}
```

Field reference (**not** OBIE-named as older guides say):

| Field | Meaning |
|------|---------|
| `creditDebitIndicator` | `"Credit"` (incoming) or `"Debit"` (outgoing) |
| `Amount.amount` | value in rubles (not kopecks) |
| `description` | free-text payment purpose (NOT `purpose` / `paymentPurpose`) |
| `documentProcessDate` | ISO date of booking |
| `documentNumber` | bank document number |
| `transactionTypeCode` | Cyrillic type, e.g. `"Платежное поручение"`, `"Банковский ордер"` |
| `DebtorParty.{name,inn,kpp}` | for Credit — who sent the money |
| `CreditorParty.{name,inn,kpp}` | for Debit — who received the money |
| `DebtorAccount.identification` / `CreditorAccount.identification` | counterparty's account number |
| `status` | `"Booked"` (posted) or `"Pending"` (provisional); both listed in swagger `ExternalTransactionStatusEnum` |
| `TaxFields` | optional sub-block for tax payments: `originatorStatus`, `kbk`, `oktmo`, `base`, `documentNumber`, `documentDate`, `type`, `field107` |
| `paymentId`, `transactionId` | bank-internal IDs |

The Statement wrapper itself has `startDateBalance` and `endDateBalance` (period-bracket balances).

## ИП vs ООО — field differences in invoices and payment orders

Tochka's `SecondSide.type` (invoices, closing documents) and counterparty reqs (payment orders) differ for legal entities vs individual entrepreneurs:

| Field | ООО / АО (`type: "company"`) | ИП (`type: "ip"`) |
|-------|------------------------------|--------------------|
| `taxCode` (ИНН) | 10 цифр | **12 цифр** |
| `KPP` | обязателен | **не передаётся** (или `"0"` в платёжке) |
| `legalName` | `"ООО \"Название\""` | `"ИП Фамилия Имя Отчество"` |
| Типичный `ndsKind` | `nds_22` (ОСНО) или `without_nds` (УСН) | `without_nds` (УСН без льготы), `nds_5` / `nds_7` (УСН с льготой), `nds_22` (ОСНО) |
| Налоговые поля в payment order (`taxInfo`) | заполнять для бюджетных платежей | аналогично; ИП на УСН платит взносы и налог через те же поля (КБК отличаются) |

**Для держателя токена = ИП на УСН** специальных ограничений на endpoints у Точки нет — ИП-клиенты пользуются той же JWT/OAuth-схемой что и юрлица. Типичный use-case: выписка + выставление счетов с `ndsKind: "without_nds"`. Льготные ставки `nds_5` / `nds_7` — только если ИП явно перешёл на УСН с НДС.

Старое значение `"entrepreneur"` (вместо `"ip"`) больше не принимается.

## Invoices (OAuth-only on prod)

> ⚠️ **501 with a personal JWT on prod.** Invoice API works in sandbox (useful for schema dev/debug) but needs OAuth+Consent on prod.

> ⚠️ **No list endpoint.** There is **NO** `GET /invoice/v2.0/bills` returning all invoices — any path like `/bills`, `/bills-list`, `/invoices`, `/documents`, `/outgoing`, `/my` returns 501. Invoices can only be queried by a known `documentId` (returned from `POST /bills` at creation time, or copied manually from ЛК). If an invoice was created via the web UI in online banking and you don't have its `documentId` — **you cannot find it via API**. Confirmed via docs (developers.tochka.com/docs/tochka-api/api/rabota-s-vystavleniem-schetov) and live-probed 2026-04-20.

### Create invoice
```
POST /invoice/v2.0/bills
```

> **v1.0 vs v2.0 schema split.** Swagger v1.90.4-stable documents only `/invoice/v1.0/bills` and uses different field names: `SecondSide.secondSideName` (not `legalName`), `SecondSide.kpp` (not `KPP`), `Positions[].positionName` (not `name`), with `Content.Invoice.{number, date, paymentExpiryDate}` — i.e. number/date/due-date are **nested inside `Content.Invoice`** rather than at the top `Data` level. On prod the v2.0 shape (flat top-level `documentNumber`/`documentDate`/`paymentExpirationDate`, capitalised `legalName`/`KPP`, `Positions[].name`) is what works and what this skill uses. If you ever hit v1.0 directly, switch to the swagger-canonical names.

Body (exact shape — capitalisation matters):
```json
{
  "Data": {
    "customerCode": "100000001",
    "accountId": "40702810900000000001/044525225",
    "documentDate": "2026-04-20",
    "documentNumber": "INV-2026-001",
    "paymentExpirationDate": "2026-05-20",
    "SecondSide": {
      "taxCode": "7700000000",
      "KPP": "770001001",
      "type": "company",
      "legalName": "ООО \"Покупатель\""
    },
    "Content": {
      "Invoice": {
        "number": "INV-2026-001",
        "totalAmount": 50000.00,
        "Positions": [{
          "name": "Консультационные услуги",
          "price": 50000.00,
          "quantity": 1,
          "totalAmount": 50000.00,
          "unitCode": "услуга.",
          "ndsKind": "without_nds"
        }]
      }
    }
  }
}
```

Field reference:

- `SecondSide.type`: `"company"` (ООО, АО — 10-digit `taxCode` + `KPP` required) or `"ip"` (ИП — 12-digit `taxCode`, no KPP). The old `"entrepreneur"` is rejected.
- `Positions[].unitCode` — Russian short form with trailing dot (not ОКЕИ code). Accepted:
  `'шт.'`, `'тыс.шт.'`, `'компл.'`, `'пар.'`, `'усл.ед.'`, `'упак.'`, `'услуга.'`, `'пач.'`, `'мин.'`, `'ч.'`, `'сут.'`, `'г.'`, `'кг.'`, `'л.'`, `'м.'`, `'м2.'`, `'м3.'`, `'км.'`, `'га.'`, `'кВт.'`, `'кВт.ч.'`.
- `Positions[].ndsKind` (ставки 2026 года):
  - `without_nds` — без НДС (типовой для ИП на УСН без перехода на НДС-льготу, и для освобождённых по ст. 145 НК РФ)
  - `nds_0` — 0% (экспорт и спецоперации)
  - `nds_5` — 5% (льгота УСН при доходах 20–272,5 млн ₽ в год, без права вычета)
  - `nds_7` — 7% (льгота УСН при доходах 272,5–490,5 млн ₽ в год, без права вычета)
  - `nds_10` — 10% (детские товары, продукты, лекарства, книги)
  - `nds_22` — 22% (стандарт с 01.01.2026, заменил 20%)

Prior enum (`vat_20`, `vat_10`, `without_vat`) is no longer accepted.

**Position constraints (swagger `PositionModel`):**
- `price`: number, `>= 0`
- `quantity`: number, `0 < q < 10000000` (cannot be zero)
- `totalAmount`: number, `>= 0`
- `totalNds` — optional, НДС по позиции (server normally computes if omitted)

**Optional `Content.Invoice` fields (not commonly used but in swagger):**
- `basedOn` — свободный текст «на основании» (договор №..., счёт-оферта и т.п.)
- `comment` — комментарий к позициям
- `totalNds` — общая сумма НДС по всему счёту

Response: `Data.documentId`.

### Get invoice PDF (verified live on prod)
```
GET /invoice/v1.0/bills/{customerCode}/{documentId}/file
```
Returns `application/pdf` binary (~50 KB). **Can take 30+ seconds on first call** (server renders the PDF on demand) — set timeout ≥ 60s.

### Check invoice payment status (verified live on prod)
```
GET /invoice/v1.0/bills/{customerCode}/{documentId}/payment-status
GET /invoice/v2.0/bills/{customerCode}/{documentId}/payment-status
```
Response shape:
```json
{ "Data": { "paymentStatus": "payment_paid" } }
```
Swagger `PaymentStatusEnum` enumerates exactly three values: `payment_waiting`, `payment_paid`, `payment_expired`. Earlier guides speculated about a fourth `payment_cancelled` — it's **not** in the API. Manual deletion in ЛК simply removes the invoice (DELETE); there is no "cancelled" paid-state.

### Send invoice to email

**v1.0 path (canonical per swagger):**
```
POST /invoice/v1.0/bills/{customerCode}/{documentId}/email
```
Body: `{ "Data": { "email": "buyer@example.com" } }`

The `/send-to-email` suffix used in some older docs returns 404. Swagger's `send_invoice_to_email_...` operation uses the bare `/email` suffix. The helper script's `send-invoice` subcommand now hits this path. v2.0 is not documented for this specific operation — stay on v1.0.

### Full list of Invoice API operations

There are exactly **5** operations — no more, no less. **All take `customerCode` in the path** (not just body):

| Verb | Path | Purpose | Verified? |
|------|------|---------|:---------:|
| POST | `/invoice/v2.0/bills` | Create invoice (returns `documentId`) | sandbox only |
| DELETE | `/invoice/v2.0/bills/{customerCode}/{documentId}` | Delete | inferred |
| POST | `/invoice/v1.0/bills/{customerCode}/{documentId}/email` | Send to email | ✅ swagger v1.90.4-stable |
| GET | `/invoice/v1.0/bills/{customerCode}/{documentId}/file` | Get rendered PDF | ✅ prod |
| GET | `/invoice/v1.0/bills/{customerCode}/{documentId}/payment-status` | Payment status | ✅ prod |

**Pro-tip for finding `documentId` of an invoice created via ЛК:** open the invoice in online banking at https://i.tochka.com/bank/m/document_flow/document/{documentId} — the UUID in the URL path IS the `documentId` you need for API calls.

Earlier docs and agent guesses used `/invoice/v2.0/bills/{documentId}/*` (without customerCode) — those are **wrong** and return 501 on prod. The customerCode is always required in the path for per-invoice operations.

## SBP QR codes

### Register retailer (one-time)
```
POST /sbp/v2.0/register-retailer/account/{accountId}
```
Returns `merchantId` for subsequent QR operations.

### Register a QR code
```
POST /sbp/v2.0/qr-code/merchant/{merchantId}/account/{accountId}
```

Body:
```json
{
  "Data": {
    "qrcType": "02",
    "amount": 150000,
    "currency": "RUB",
    "paymentPurpose": "Заказ №1234"
  }
}
```

`qrcType`: `"01"` static, `"02"` dynamic.

**⚠️ `amount` is in kopecks, not rubles.** Swagger `RegisterQRCode` schema explicitly titles the field "Сумма в копейках". To bill 1500 ₽ pass `150000`. Different from acquiring `/payments` where `amount` is float rubles — easy to confuse. Also note: `paymentPurpose` in the SBP QR has `maxLength: 140` (tighter than the 210 limit for payment orders).

**SBP NSPK v9.1 (since April 2026):** `bankCode` (БИК) is a required parameter in retailer registration and some other SBP endpoints. Also the signature of the `incomingSbpPayment` webhook changed — refresh JWKS before verifying.

### List QR codes
```
GET /sbp/v2.0/qr-codes/{merchantId}
```

### Permissions
- `EditSBPData` (register/modify)
- `ReadSBPData` (read)

ReDoc: https://enter.tochka.com/doc/v2/redoc#tag/SBP-API

## Closing documents (акты / УПД / накладные / счета-фактуры)

OAuth-gated, same `ManageInvoiceData` permission as invoices. Natural follow-up when an invoice is paid and the customer needs a closing document.

| Verb | Path | Purpose |
|------|------|---------|
| POST | `/invoice/v1.0/closing-documents` | Create act/УПД/ТОРГ-12/счёт-фактура |
| GET | `/invoice/v1.0/closing-documents/{customerCode}/{documentId}/file` | Download rendered PDF |
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

`Content` is a discriminated union — use one of: `Act` (акт выполненных работ), `PackingList` (товарная накладная ТОРГ-12), `Invoicef` (счёт-фактура), `Upd` (УПД). Position shape matches invoice `PositionModel` (same constraints). Optional `documentId` links the closing doc to a parent invoice so ЛК groups them in one thread.

**⚠️ Critical v1.0 vs v2.0 shape differences (verified on prod 2026-04-21):**
- **`SecondSide` uses lowercase `kpp` and `secondSideName`** — NOT the v2.0 invoice shape (`KPP` / `legalName`). If you send `KPP` here, Tochka silently drops it → buyer КПП missing in the rendered PDF → ЭДО signing fails with `"Не получилось подписать документ. Проверьте ИНН или КПП контрагента."` The doc still gets a `documentId` at creation time, so the bug only surfaces at signing.
- **`Content.Act.totalAmount` is required** at the block level (not just inside each position). Missing it returns `400 Validation Error: Field Content-ContentAct-Act-totalAmount : Field required`.

Helper subcommands: `create-closing-doc`, `get-closing-doc`, `send-closing-doc`, `delete-closing-doc`.

## Balances (текущий баланс)

```
GET /open-banking/v1.0/balances
GET /open-banking/v1.0/accounts/{accountId}/balances
```

Returns live balance snapshot. Works with personal JWT (permission: `ReadBalances`). Useful pre-check before creating a payment order. `accountId` URL slash must be encoded as `%2F`.

Helper: `get-balance` (all accounts) / `get-balance --account-id ...` (one).

## Webhooks

Webhook types per swagger `WebhookTypeEnum`: `incomingPayment`, `outgoingPayment`, `incomingSbpPayment`, `acquiringInternetPayment`, `incomingSbpB2BPayment`.

| Verb | Path | Purpose |
|------|------|---------|
| PUT | `/webhook/v1.0/{client_id}` | Register webhook URL + subscribed events |
| POST | `/webhook/v1.0/{client_id}` | Edit existing webhook |
| GET | `/webhook/v1.0/{client_id}` | Get current webhook config |
| DELETE | `/webhook/v1.0/{client_id}` | Delete webhook |
| POST | `/webhook/v1.0/{client_id}/test_send` | Send a test event to your URL |

Body shape (PUT): `{"webhooksList": ["incomingPayment", "incomingSbpPayment"], "url": "https://your-endpoint/hook"}`. `url` must be `https://` (max 2083 chars). Requires `ManageWebhookData`.

`{client_id}` is the OAuth app `client_id` (for personal JWT — the `client_id` shown when the JWT was generated in ЛК). Use `test_send` right after registering to confirm the receiver is wired up before waiting for a real payment.

## Payment links (интернет-эквайринг)

```
POST /acquiring/v1.0/payments                  # plain payment link
POST /acquiring/v1.0/payments_with_receipt     # with fiscal receipt (ОФД)
GET  /acquiring/v1.0/payments/{operationId}
POST /acquiring/v1.0/payments/{operationId}/capture    # two-stage capture
POST /acquiring/v1.0/payments/{operationId}/refund     # full/partial refund
```

Required (plain): `customerCode` (exactly 9 chars), `amount` (number in rubles — unlike SBP QR), `purpose`, `paymentMode` (`["sbp", "card"]`). Optional: `merchantId`, `paymentLinkId` (your external ref), `redirectUrl`, `failRedirectUrl`, `ttl` (1–44640 minutes, default 10080 = 7 days), `preAuthorization` (true → must call `/capture` to charge), `saveCard`, `consumerId`.

Returns `Data.Operation.{operationId, paymentLink}` — share `paymentLink` with the client. Requires `MakeAcquiringOperation`.

Helper: `create-payment-link`. For pre-auth flow the skill does NOT auto-capture — call `/capture` manually when shipping the goods.

## Acquiring daily registry

```
GET /acquiring/v1.0/registry?customerCode=...&merchantId=...&date=YYYY-MM-DD
```

Daily settlement roll-up (реестр интернет-эквайринга): totals, commissions, net transferred amount. Useful for month-end reconciliation against the bank statement. Optional `paymentId` for per-payment drill-down. Requires `ReadAcquiringData`.

Helper: `list-registry --date YYYY-MM-DD`.

## Consents (OAuth introspection)

```
GET  /consent/v1.0/consents                      # all consents for the app token
GET  /consent/v1.0/consents/{consentId}          # single consent details
GET  /consent/v1.0/consents/{consentId}/child    # child consents (multi-account case)
POST /consent/v1.0/consents                      # create a new consent (setup)
```

Read-only calls. Useful when OAuth starts returning 403 — check which scopes the active consent actually carries before assuming token expiry.

Helpers: `list-consents`, `get-consent <consentId>`.

## accountId format — critical pitfall

Tochka's `accountId` is NOT a plain account number. It has the format `{20-digit-account}/{9-digit-BIC}` — with an embedded forward slash. Example:

```
"accountId": "40702810900000000001/044525225"
```

In request **bodies** pass it as-is. In **URL paths** the slash must be URL-encoded as `%2F`:

```
GET /open-banking/v2.0/accounts/40702810900000000001%2F044525225/statements/{statementId}/payments
```

Forgetting to encode the slash produces `404 HTTPNotFound: Not found statement under account`.
