# ZiCharge Merchant Payment API — Developer Documentation

**Version:** v3 (Java engine)
**Audience:** Merchant developers, integration engineers, partner solution architects
**Last reviewed:** 2026-05-14
**Status:** Production

---

Welcome to the ZiCharge Merchant Payment API. The API lets your platform accept payments from ZiCharge wallet holders through a **hosted payment page**, **direct API charge**, or **QR-code checkout**. It is the same gateway your customers already know — re-engineered on a Java 21 / Spring Boot 3.2 backend with stronger validation, idempotent state machines, race-free token lifecycle, signed-callback delivery with automatic retries, and a low-latency hot path designed for high-throughput merchant traffic.

The API is **wire-compatible with the previous gateway version** so that merchants who are live today require no client changes to keep working. New capabilities are additive and opt-in.

---

## What's in this folder

| # | Document | What you'll find |
|---|----------|------------------|
| 1 | [01-overview.md](01-overview.md) | What the gateway does, supported flows, environments, base URLs, key concepts |
| 2 | [02-authentication.md](02-authentication.md) | Merchant credentials, store password, IP allow-listing, IPN HMAC, security model |
| 3 | [03-payment-flow.md](03-payment-flow.md) | End-to-end checkout flow, sequence diagrams, state machine, timing & expiry |
| 4 | [04-api-reference.md](04-api-reference.md) | Every endpoint: URL, request schema, response schema, errors, examples |
| 5 | [05-ipn-callbacks.md](05-ipn-callbacks.md) | IPN (Instant Payment Notification) contract, payload, retries, signature verification, idempotency on your side |
| 6 | [06-error-codes.md](06-error-codes.md) | Catalogue of `code` values, when they occur, recommended client handling |
| 7 | [07-testing-and-go-live.md](07-testing-and-go-live.md) | Sandbox, test data, go-live checklist, rotation, observability you can expect |
| 8 | [08-migration-notes.md](08-migration-notes.md) | Differences vs. the previous gateway, blocker check, backwards-compat guarantees |

---

## Quick start

If you have already integrated with the previous version of the gateway, you can keep using the same endpoint paths and payloads. Read [08-migration-notes.md](08-migration-notes.md) for the small list of behavioural changes you should be aware of.

For a brand new integration, the shortest path is:

1. Provision a merchant account and a store password from your ZiCharge merchant dashboard.
2. Configure an IPN listener URL and (optionally) success / cancel / fail redirect URLs.
3. Call `POST /merchant/generate-payment-token` from your server to mint a token.
4. Redirect the customer to `https://<gateway-host>/merchant/payment?token=<token>`.
5. Wait for the IPN callback on your listener (signed, retried), then call `POST /merchant/validate-payment` or `POST /merchant/fetch-payment-status` from your server to confirm before fulfilling the order.

That is the entire happy path. Everything else in this folder is here to make sure the unhappy paths are also boring.
