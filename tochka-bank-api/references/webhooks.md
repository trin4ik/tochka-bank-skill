# Tochka Bank API — Webhooks

Real-time event delivery from Tochka to your server. Strongly recommended over polling for incoming-payment notifications.

## Setting up a webhook URL

In online banking → **Интеграции и API** → existing JWT/OAuth integration → set webhook URL. The URL must:
- Be HTTPS
- Respond with HTTP 200 within ~10 seconds
- Accept `POST` with `Content-Type: text/plain` (Tochka sends raw JWT-signed bodies)

The `client_id` shown alongside the JWT is needed to verify the webhook signature.

## Available events

| Event name | Fires when |
|------------|------------|
| `incomingPayment` | Incoming non-SBP payment credited to one of your accounts |
| `incomingSbpB2BPayment` | Incoming SBP payment (B2B or QR-driven C2B) |
| `outgoingPayment` | Outgoing payment executed by the bank |
| `acquiringInternetPayment` | Card payment via internet acquiring completed |
| `invoiceStatusChanged` | Invoice status transition (paid, expired, etc.) |
| `sbpInboundC2BPayment` | C2B SBP payment via dynamic QR |

Latency: typically under 20 seconds from the bank-side event to webhook delivery.

Authoritative list: https://developers.tochka.com/docs/tochka-api/opisanie-metodov/vebhuki

## Request format

The body is a JWS-signed JWT (Tochka calls this their "webhook payload"). Decoding gives a JSON object similar to:

```json
{
  "webhookType": "incomingPayment",
  "client_id": "your_client_id",
  "data": {
    "accountId": "40802810500000123456",
    "amount": 15000.00,
    "currency": "RUB",
    "counterpartyName": "ООО \"Плательщик\"",
    "counterpartyINN": "7700000000",
    "paymentPurpose": "Оплата по счёту №7",
    "paymentDate": "2026-04-20",
    "paymentId": "..."
  }
}
```

The exact field set depends on the event type — log the first few payloads in development to confirm shape against the actual messages your account receives.

## Verifying the signature

The webhook body is a JWT signed by Tochka. Steps to verify:

1. Resolve JWKS URL from the OIDC discovery document (surviving future URL changes): fetch `https://enter.tochka.com/connect/.well-known/openid-configuration` and follow `jwks_uri`
2. Fetch the JWKS from the resolved URL
3. Decode the JWT header to get `kid` (key id)
4. Find the matching public key in JWKS
5. Verify the signature with that key
6. Reject if the `client_id` claim doesn't match your registered `client_id`

Reference Python (using `pyjwt[crypto]` and `requests`):

```python
import jwt, requests

# Resolve JWKS via discovery — hardcoding /connect/jwks breaks when Tochka rotates the path.
DISCOVERY = requests.get("https://enter.tochka.com/connect/.well-known/openid-configuration").json()
JWKS = requests.get(DISCOVERY["jwks_uri"]).json()

def verify_webhook(raw_body: bytes, expected_client_id: str) -> dict:
    token = raw_body.decode("ascii").strip()
    unverified_header = jwt.get_unverified_header(token)
    key = next(k for k in JWKS["keys"] if k["kid"] == unverified_header["kid"])
    public_key = jwt.algorithms.RSAAlgorithm.from_jwk(key)
    payload = jwt.decode(token, public_key, algorithms=[unverified_header["alg"]],
                        audience=expected_client_id)
    return payload
```

Confirm the audience claim against current Tochka docs before deploying — the JWKS URL is now resolved dynamically, but the `aud` contract may evolve.

## Idempotency and retries

Tochka retries failed deliveries (non-2xx responses) with exponential backoff for up to ~24 hours. Treat events as **at-least-once**:
- Store `paymentId` (or equivalent unique field per event type) in a deduplication table
- Make handler effectively idempotent — on duplicate, return 200 without re-processing

If you ever return 5xx for a real bug, fix and **request manual replay** via Tochka support — there's no self-serve replay endpoint.
