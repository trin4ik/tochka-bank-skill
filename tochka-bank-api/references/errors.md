# Tochka Bank API — Error → fix cheatsheet

Exact error string + cause + fix. If your error isn't listed, fall back to:
1. Live ReDoc https://enter.tochka.com/doc/v2/redoc (source of truth for shapes).
2. Developer docs https://developers.tochka.com/docs/tochka-api/.
3. Telegram support (linked from the developer portal) — Tochka replies fast; their docs sometimes lag the API.

| Exact error / symptom | Cause | Fix |
|---|---|---|
| HTTP 501 on `/invoice/...`, `/open-banking/.../statements`, `/closing-documents` | Personal JWT can't reach this tier | Switch to OAuth (`init --oauth`) |
| HTTP 401 on OAuth endpoints after a quiet period | `refresh_token` expired (30-day window) | Re-run `init --oauth` — there's no silent recovery once the refresh token dies |
| HTTP 403 with a fresh token | Missing scope or consent expired | `list-consents` / `get-consent --consent-id …` to see granted scopes; `init --oauth` if the consent lapsed |
| HTTP 403 with an old token | JWT TTL elapsed | Regenerate JWT in online-banking → Интеграции и API → new key → `init` again |
| `404 Not found statement under account` | Un-encoded `/` in `accountId` URL path | URL-encode as `%2F` (in request bodies, leave the slash as-is) |
| `"Datetimes provided to dates should have zero time"` | Full ISO datetime in statement date field | Pass `YYYY-MM-DD` only |
| `"should be less than today or equal"` on payment | `paymentDate` is in the future | Today or earlier; for deferred sends, submit a draft and have a human sign it on the target date |
| `"forbidden symbols: —"` on `paymentPurpose` | Em-dash U+2014 in the string | Replace with hyphen `-` or comma |
| `Field …-Positions-0-unitCode : Input should be 'шт.', …` on invoice/closing-doc | `unitCode` outside the closed whitelist (e.g. `мес.`, `год.`, an OKEI code, missing trailing dot) | Pick from the whitelist — for subscriptions use `услуга.` or `шт.` with the period inside the position name. `tochka_client.py` enforces this client-side via argparse `choices`. |
| ЭДО signing fails on a closing doc: `"Не получилось подписать документ. Проверьте ИНН или КПП контрагента."` | `cmd_create_closing_doc` was called with the v2.0 invoice shape (`KPP` / `legalName`); Tochka silently drops it for closing-docs → buyer КПП missing in PDF | Closing-doc `SecondSide` requires lowercase `kpp` and `secondSideName`. Re-create with the v1.0 shape; the helper does this correctly since 2026-04-21. |
| `"Проверьте номер счёта — в выбранном банке такого счёта нет"` | Fake/typo counterparty account | Counterparty account is validated against the Russian Central Bank registry — use real numbers |
| `"Field X: Field required"` for every field on payment | Nested `Data.Payment` instead of flat `Data.{…}` | Flatten the body — v2.0 schema is flat, not array/nested |
| Invoice created but response lacks `documentUid` | Looking at the wrong field | Use `Data.documentId` (the `documentUid` alias was removed) |
| `error: init needs a real terminal…` / wizard hangs | No TTY (ran through the agent's Bash tool) | User must run with the `!` prefix: `! python3 .../tochka_client.py init` |
| Invoices/payments "succeed" but don't show up in online banking | `TOCHKA_SANDBOX=1` is stuck in env, calls go to sandbox | `unset TOCHKA_SANDBOX`; verify target via the hook prompt label or by running `list-accounts` against a known prod `customerCode` |
| OAuth callback never arrives (5-min timeout) | Registered Redirect URL ≠ wizard default | `init --oauth --redirect-url <exact URL from online banking>` |
| OAuth: browser shows a cert warning during authorize | `mkcert` not installed; falling back to self-signed | `brew install mkcert && mkcert -install` then re-run `init --oauth` (or click through the warning once) |
| `OSError: [Errno 48] Address already in use` during `init --oauth` | Port 8443 held by a previous wizard run | `lsof -i :8443` → kill the process, or pass `--redirect-url https://127.0.0.1:<other-port>/callback` (must match the registration in online banking) |
| `security: SecKeychainSearchCopyNext` / Keychain denied | macOS Keychain access denied or locked | Unlock Keychain and retry; or `init --storage file` to fall back to a chmod-600 file |
| PDF download hangs / times out | First server-side render takes ~30 s | The client already uses a 90 s timeout — if it still fails, retry once |
| Payment was signed but is no longer in `list-for-sign` | Once signed and dispatched, it leaves the «На подпись» queue | Use `list-statement` (OAuth) to see executed payments |
