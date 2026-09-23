# Cancel Booking (TRAVALA)

**Summary**

- Cancels a completed (`COMPLETED`) TRAVALA hotel booking. **Irreversible**.
- Two-stage email verification code flow, calling the same endpoint twice:
  1. First call **without `otp`** — Travala sends a 6-digit verification code to the user's login email and returns `state=OTP_SENT`;
  2. Once the user provides the code, **call again with `otp`** — this actually executes the cancellation and returns `state=CANCELLED` on success.
- The first stage can be called repeatedly = resend the code (the old code is invalidated immediately). If the user receives multiple verification emails, use the latest one (a new email may be delayed by a minute or two).
- **Refund destination (you must inform the user truthfully)**: any refund from the cancellation (if any) is issued only as **Travel Credits** to the user's own Travala account (the account is the booking email, used by logging in at travala.com); it is **never refunded to this platform's wallet, and there is no cash refund**. The refund amount and arrival time are determined by Travala's actual processing per that booking's cancellation policy (a cancellation fee may be charged).
- Idempotent: repeated calls on an already-cancelled order return `state=CANCELLED` directly, without sending another code.

**Request**

- POST `/orders/booking/cancel`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for generation rules |
| orderNo | Yes | string | System order number returned by the pay response (must be a `COMPLETED` TRAVALA order) |
| lastName | No | string | Guest's last name. Defaults to the guest last name from the order receipt; if upstream verification fails, explicitly pass the lastName entered at booking time |
| otp | No | string | 6-digit email verification code. Omit in the first stage (code sending); required in the second stage |

**Request Examples**

First stage (send the code):

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "orderNo": "AIS17889..." }
```

Second stage (confirm cancellation):

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "orderNo": "AIS17889...", "otp": "123456" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| state | string | `OTP_SENT` = verification code has been sent to the user's login email; `CANCELLED` = booking has been cancelled |
| bookingId | string | Travala booking number (Booking ID) |
| email | string | Email that receives the code (the user's login email; both the verification code and the cancellation confirmation email are sent to it) |
| refundNote | string | Fixed statement of the refund destination (English): Travel Credits go to the user's Travala account, not refunded to the platform wallet |
| message | string | English summary returned by upstream (cancellation fee / cancellation policy, etc.); relay it to the user truthfully |

**Response Examples**

```json
{ "state": "OTP_SENT", "bookingId": "MN5V9DWQ", "email": "user@example.com",
  "refundNote": "Any refund is issued as Travala Travel Credits to the traveler's Travala account (the booking email) on travala.com, not to the platform wallet. Refund amount and timing are determined by Travala per the booking's cancellation policy.",
  "message": "A verification code has been sent to user@example.com." }
```

```json
{ "state": "CANCELLED", "bookingId": "MN5V9DWQ", "email": "user@example.com",
  "refundNote": "Any refund is issued as Travala Travel Credits to the traveler's Travala account (the booking email) on travala.com, not to the platform wallet. Refund amount and timing are determined by Travala per the booking's cancellation policy.",
  "message": "Booking MN5V9DWQ has been successfully cancelled." }
```

**Error Handling**

| code | Scenario / client action |
|---|---|
| 91027 | Order not cancellable: not a TRAVALA order / not `COMPLETED` / upstream refused per the cancellation policy (original text in `msg`) → do not retry as-is |
| 91019 | Verification code wrong or expired → resend the code without `otp` and have the user retry with the code from the latest email (do not repeatedly re-check the order number) |
| 91017 | Parameter missing (e.g. the receipt has no guest last name and `lastName` was not explicitly passed) → supply it per `msg` and retry |
| 91018 | Order does not exist → verify `orderNo` |
| 91021 | Upstream temporarily unavailable → retry later |

**Notes**

- After a successful cancellation, the `receipt` in "Order Detail" gains two new fields, `cancelledAt` (cancellation time) and `refundNote`; the order `status` remains `COMPLETED` unchanged (no funds move on the platform side; the refund is handled by Travala on its side).
- This endpoint is in the funds rate-limit tier (5 req/min); verification code sending relies on this limit to prevent email bombing.
- Before submitting the cancellation, the user must be made aware of the refund destination (`refundNote`) and explicitly confirm — cancellation is irreversible.
