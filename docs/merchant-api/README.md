# Merchant API — Agentic Shopping & Payment

A server-to-server REST API that lets a merchant system (or an agent platform) search products and complete purchases **on behalf of its users**. Funds always come from the user's own AEON wallet; payment is executed server-side — the merchant never touches card data.

Channels: **SHOPIFY** (physical goods, card payment) · **AMAZON** (physical goods, card payment, US only) · **TRAVALA** (hotel booking, wallet payment).

> The merchant must ensure it has the user's authorization for purchase-on-behalf / pay-on-behalf operations.

## Protocol at a glance

| | |
|---|---|
| Base URL | `https://aipay-api.aeon.xyz/agentapi/v1` |
| Transport | All endpoints are **POST + JSON** (UTF-8), timestamps in ISO-8601 UTC |
| Auth | `appId` / `userId` / `sign` at the top level of every request body — per-request MD5 signature, see [Signature](shopping/01-signature.md) |
| Result | HTTP status is always 200; success/failure is decided solely by the envelope `code` (`"0"` = success), see [Error Codes](shopping/10-error-codes.md) |
| Credentials | `appId` + `secret` issued offline or via the merchant console. `secret` must never leave your server |

## Quick start: your first purchase

```
⓪ Initialize account   POST /account/init          (once per new userId — mandatory first call)
⓪ Initialize AI Card   POST /wallet/card/init      (paid card creation; required for SHOPIFY / AMAZON)
                         → poll POST /wallet/card/status until cardStatus = ACTIVE
① Shipping address     POST /address/get           (POST /address/save if missing)
② Search               POST /search
③ Product detail       POST /items/detail          → pick variantId / asin / packageId
④ Check balance        POST /wallet/assets
⑤ Create order         POST /orders/create         → channelOrderId + quote (locks the price, no funds move)
        ── echo the item & total back to the user, get explicit consent ──
⑥ Pay                  POST /orders/pay            → orderNo (pass your own outTradeNo as the idempotency key;
                                                      pass webhookUrl to receive terminal-state callbacks)
⑦ Poll                 POST /orders/detail every 5–10 s
                         ├─ pendingAction != null → ask the user for the code → POST /orders/actions
                         ├─ COMPLETED            → report the merchant order number
                         ├─ FAIL                 → handle by failCode, show `reason` verbatim
                         └─ PROCESSING           → keep polling
⑧ Webhook              on terminal state (COMPLETED / FAIL / TIMEOUT) the server POSTs to your webhookUrl
                         (verify the signature + dedupe by outTradeNo; retries with decay for ~12 h)
```

Polling and webhooks are complementary: the webhook frees you from long polling, but delivery may be delayed or lost — **the authoritative status is always [Payment Detail](shopping/06-payment-detail.md)**; re-query once after receiving a webhook before booking the result.

## Reference

### AI Shopping ([shopping/](shopping/))

| Doc | Endpoint(s) |
|---|---|
| [Integration Overview](shopping/00-overview.md) | envelope, call sequence, order-number glossary, rate limits, integration rules |
| [Signature](shopping/01-signature.md) | signing algorithm + Java sample |
| [Product Search](shopping/02-product-search.md) | `POST /search` |
| [Product Detail](shopping/03-product-detail.md) | `POST /items/detail` |
| [Create Order](shopping/04-create-order.md) | `POST /orders/create` — locks the quote, no funds move |
| [Pay](shopping/05-pay.md) | `POST /orders/pay` — idempotent by `outTradeNo` |
| [Payment Detail](shopping/06-payment-detail.md) | `POST /orders/detail` — the authoritative status |
| [Manual Verification](shopping/07-manual-verification.md) | `POST /orders/actions` — 3DS / OTP relay |
| [Order List](shopping/08-order-list.md) | `POST /orders/list` |
| [Payment Webhook](shopping/09-payment-webhook.md) | terminal-state callback, signature verification, retry policy |
| [Error Codes](shopping/10-error-codes.md) | full `code` reference |
| [Cancel Booking](shopping/11-cancel-booking.md) | `POST /orders/booking/cancel` — TRAVALA only |

### Address Management ([address/](address/))

| Doc | Endpoint(s) |
|---|---|
| [Get Address](address/01-get-address.md) | `POST /address/get` |
| [Save Address](address/02-save-address.md) | `POST /address/save` |
| [Country List](address/03-country-list.md) | `POST /address/countries` |
| [Phone Code List](address/04-phone-code-list.md) | `POST /address/phoneCodes` |
| [State & Province List](address/05-state-list.md) | `POST /address/states` |

### Account & Assets ([account/](account/))

| Doc | Endpoint(s) |
|---|---|
| [Initialize Account](account/01-init-account.md) | `POST /account/init` — mandatory first call per userId |
| [Asset Balance](account/02-asset-balance.md) | `POST /wallet/assets` |
| [Transfer AI Card to Wallet](account/03-transfer-card-to-wallet.md) | `POST /wallet/card/redeem` |
| [Transfer Wallet to AI Card](account/04-transfer-wallet-to-card.md) | `POST /wallet/card/topup` |

**Wallet** ([account/wallet/](account/wallet/)): [Get Token Recharge Address](account/wallet/01-recharge-address.md) · [Recharge & Redeem Network List](account/wallet/02-network-list.md) · [Redeem Quote](account/wallet/03-redeem-quote.md) · [Token Redeem](account/wallet/04-token-redeem.md) · [Wallet Transactions](account/wallet/05-wallet-transactions.md)

**AI Card** ([account/card/](account/card/)): [Initialize AI Card](account/card/01-init-card.md) · [Card Status](account/card/02-card-status.md) · [Card Transactions](account/card/03-card-transactions.md) · [Card Consumption Records](account/card/04-card-consumption.md)

## Money-safety model (what you do NOT need to build)

- **Idempotency**: re-submitting `pay` with the same `outTradeNo` or the same `channelOrderId` returns the existing order — funds are never charged twice.
- **Auto refund**: a definitive payment failure is refunded automatically via the original route.
- **Unknown outcomes are frozen, never guessed**: while an order is `PROCESSING`, do not promise refunds or invent results to the user — poll or wait for the webhook.
- **Spending limits**: per-user single-purchase and daily USD caps are enforced *before* any funds move (`91011`).

## Integration rules (hard requirements)

1. Keep `secret` on your server only — never ship it to apps, web pages, or devices; never write `sign` or raw request params into publicly accessible logs. If leaked, contact operations immediately to rotate.
2. Before calling `pay`, echo the product, variant, and quoted total back to the user (for `ESTIMATE`-type channels, note that shipping/tax may be added) and obtain explicit consent.
3. Show failure `reason` to the end user verbatim; never fabricate outcomes while `PROCESSING`.
4. Poll every 5–10 seconds; on `91014` (rate limit) back off exponentially.
5. Compatibility: breaking changes ship under a new version path (`/agentapi/v2`). New response fields or new error codes within v1 are **not** breaking — your deserializer must ignore unknown fields, and unknown error codes should fall back to generic-failure handling.

## Rate limits (per user)

| Tier | Quota | Endpoints |
|---|---|---|
| Catalog | 30 /min | `/search`, `/items/detail` |
| Funds | 5 /min | `/orders/create`, `/orders/pay`, `/orders/actions`, `/orders/booking/cancel`, `/wallet/card/init`, `/wallet/card/topup`, `/wallet/card/redeem`, `/wallet/token/redeem` |
| Query | unlimited | all other read-only endpoints |
