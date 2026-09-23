# Error Codes

The HTTP status code is always 200; errors are always determined by the envelope `code` (a numeric code as a string), with `msg` being the default English message.

| code&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Meaning / client action |
|:---------------------|:---|
| 91001 | Missing common parameters (appId / userId / sign) → supply them and retry |
| 91002 | Transient server error → retry the original request once |
| 91003 | `appId` invalid or not enabled → verify onboarding info or contact operations |
| 91004 | No saved address → confirm with the user, then call "Save Address" |
| 91005 | Quote expired or `channelOrderId` does not exist → create the order again |
| 91006 | Actual total exceeded the quote tolerance; no charge made → suggest switching merchants, or retry with the user informed |
| 91007 | Insufficient balance; no charge made → CARD: card balance insufficient (the card is not automatically topped up from the wallet); first call "Transfer Wallet to AI Card" to top up, then retry. When the pay endpoint returns this **synchronously**, no order was created (blocked by the acceptance pre-check; `outTradeNo` was not consumed); when it appears as an order `failCode`, the checkout's actual total (including shipping and taxes) exceeded the card balance — retry with a new `outTradeNo` |
| 91008 | `payMethod` does not match the channel (SHOPIFY/AMAZON only CARD, TRAVALA only WALLET) → correct and retry |
| 91009 | AI Card not active; no funds moved → call "Card Status" to check `cardStatus`: `NOT_CREATED` — call "Initialize AI Card" first; `PENDING` — retry later; `UNAVAILABLE` — contact operations |
| 91010 | Transfer failed; no funds moved → retryable, `msg` explains the reason |
| 91011 | Per-transaction / daily limit exceeded; no charge made |
| 91013 | The order currently has no pending action |
| 91014 | Rate limit triggered → slow down, retry with backoff |
| 91015 | This channel is temporarily disabled in this environment → retry later or contact operations |
| 91016 | Payment failed, nothing purchased (any funds charged have been automatically refunded via the original route) → check `reason`, switch product/merchant or fix and create the order again. As an order `failCode` it is in the retryable/correctable category (card declined / address rejected / shipping-rate loading timeout / anti-automation block, etc., as indicated by `reason`) |
| 91017 | Request parameter missing or invalid (`msg` names the specific field and reason) → correct and retry |
| 91018 | Resource does not exist (order number / product / channel order ID / `sessionId` invalid) → verify the identifier; do not retry the same identifier |
| 91019 | Verification code wrong or expired (second stage of "Cancel Booking") → resend the code without `otp`, then retry with the latest code |
| 91020 | Product currently not purchasable (sold out / delisted / card payment not supported / does not ship to the destination country / agent checkout not enabled) → switch product, variant, or merchant. As an order `failCode` it is a merchant-level permanent constraint (checkout empirically rejects cards / requires login / refuses the session); retrying the same merchant will fail identically |
| 91021 | Channel upstream service temporarily unavailable → retry later. As an order `failCode` it includes "pre-payment card balance query failed" (card-issuing channel outage, not insufficient balance; no charge made): **do not top up the card**; simply retry later with a new `outTradeNo` |
| 91024 | Signature verification failed → check the string to sign and the secret per "Signature" |
| 91025 | Merchant does not ship to the delivery address (empirically confirmed rejection at checkout; only appears as an order `failCode`) → do not retry the same merchant; switch merchants or change the delivery address |
| 91026 | Order amount below the merchant's minimum order threshold (checkout empirically rejected this order; not a merchant-level constraint; only appears as an order `failCode`) → add quantity/items to reach the threshold and create the order again; retrying the same cart unchanged will definitely fail |
| 91027 | Booking not cancellable (not a TRAVALA order / not `COMPLETED` / upstream refused per the cancellation policy, original text in `msg`) → do not retry as-is; see "Cancel Booking" |
| 91028 | This `appId` + `userId` has not initialized an account yet → call "Initialize Account" first (note that the same email under different `appId`s is a different account; assets are not shared) |
| 1 | Unexpected error; query endpoints can be safely retried |
