# WhatsApp Business Coexistence: #3441034, interrupted signup #1690130, and sync error 131000/10

**Martín Muñoz · November 2, 2026**  
*Meta Cloud API · WhatsApp Business App Coexistence · Embedded Signup · Webhooks*

> **TL;DR:** In my case, Embedded Signup error **#3441034** stopped occurring after I disconnected **Meta Business Suite Inbox** under *WhatsApp Business → Settings → Account → Business Platform*. Later, an interrupted onboarding repeatedly showed **#1690130**, although the business portfolio existed and was verified. Graph API indicated the number was already `CONNECTED` / `CLOUD_API` / `is_on_biz_app: true`. My app wasn't subscribed to the WABA and had no local channel, so I completed **that missing integration** with a dedicated system-user token. **Incoming messages then reached both the WhatsApp Business app and my SaaS**. But `smb_app_data` **rejected history and contact sync** with **131000, subcode 10**. I do **not** claim Meta's interrupted onboarding state was fully repaired.

This is a real, first-person incident report, not official Meta documentation or a universal fix. Business identifiers, phone numbers, session IDs, and secrets are intentionally omitted.

## Error timeline

I was connecting an existing WhatsApp Business app number to the official Cloud API, **without moving it out of the WhatsApp Business mobile app**. Multiple different errors appeared:

| Error or symptom | What I observed | Result |
| --- | --- | --- |
| **#3441034** — not eligible to register/migrate | The number was already connected to **Meta Business Suite Inbox** | Removing *only that platform connection* allowed Embedded Signup to advance |
| **#2494064** — already registered | An attempt treated the number as already registered | Returned to coexistence rather than trying to delete or migrate it |
| **#1690130** — business identifier not valid | Repeated after I interrupted onboarding around the business/time-zone step | **Exact cause unknown**; I recovered the separate SaaS-side integration using Graph API |
| WhatsApp Manager **Offline** while Graph said **CONNECTED** | Two conflicting signals | Verified delivery with a real incoming message |
| `subscribed_apps` empty | No WABA webhooks to my app | Subscribed the existing app to the existing WABA |
| **131000 / 10** on `smb_app_data` | History and contacts sync requests rejected | **Still unresolved**; needs Meta support |
| **141006** | Billing/payment issue affecting outbound initiation | Separate issue; not the reason incoming webhook events were missing |

## 1. What unblocked #3441034

The important setting was in the WhatsApp Business app on the device hosting the account, not the Facebook Login for Business configuration:

1. Open **WhatsApp Business → Settings → Account → Business Platform** (labels may vary).
2. Look for **Meta Business Suite Inbox** as the connected platform.
3. If you're switching to a different platform, read the warning and disconnect **only this Business Platform integration**.
4. Return to your vendor's **“Keep using WhatsApp Business” / coexistence** signup flow.
5. Finish both the browser and device authorization steps.

That is precisely what allowed **my** phone to advance past #3441034. It does **not** mean every #3441034 is caused by Meta Business Suite Inbox. Troubleshooting documentation from Bird covers other eligibility or setup conditions.

I also got “Something went wrong. Try again” when tapping Disconnect. I restarted the device; a later attempt worked. **I cannot prove that rebooting caused the fix.** It may have been a temporary Meta-side problem.

**Do not blindly click “Remove from business portfolio” or “Delete WhatsApp account.”** Those are not equivalent to disconnecting Business Platform and can have serious consequences.

## 2. #2494064: an existing number is not a new number

The message that a number is “already registered” is not a reason to delete it. My requirement was to **retain WhatsApp Business on the device**. Standard new-number registration or migration is not interchangeable with coexistence.

## 3. #1690130 after an interrupted Embedded Signup

I advanced through the initial signup screens, selected the wrong time zone, went back, and disconnected the platform from my device before completing the browser flow. Subsequent attempts failed at business confirmation with **#1690130** (“[portfolio ID] is not a valid business identifier”).

The portfolio existed and was verified. Reopening the browser, signing in again, disconnecting/reconnecting from the device, and checking permissions did **not** fix this error.

**I cannot prove whether the interruption, time zone, asset ownership, or another Meta-side condition caused #1690130.** Please do not treat those theories as confirmed facts.

The useful next step was to determine **how much of the onboarding Meta had already completed**.

## 4. Read-only diagnosis using Graph API

With an authorized token, I checked the existing number, its WhatsApp Business Account (WABA), subscriptions, and my own database.

| Evidence | Observed value |
| --- | --- |
| Graph number status | `CONNECTED` |
| Platform | `CLOUD_API` |
| WhatsApp Business app indicator | `is_on_biz_app: true` |
| WABA | Exists and contains the number |
| `GET /<WABA_ID>/subscribed_apps` | Empty |
| SaaS database | No channel for the number |
| App webhook field subscriptions | `messages`, `smb_message_echoes`, `history`, `smb_app_state_sync`, `account_update` already configured |

Illustrative **read-only** requests (choose a supported Graph version; use placeholders, not anyone else's IDs):

```http
GET https://graph.facebook.com/vXX.X/<PHONE_NUMBER_ID>?fields=platform_type,is_on_biz_app
GET https://graph.facebook.com/vXX.X/<WABA_ID>/phone_numbers
GET https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
Authorization: Bearer <AUTHORIZED_TOKEN>
```

A Graph 403 or code 100 can mean the token lacks access; it does **not** automatically mean the asset does not exist.

Also, `CONNECTED` is **not** proof that every coexistence feature is working. My WhatsApp Manager displayed *Offline* during the incident.

## 5. Recovering the integration without re-registering the phone

Because Graph showed an existing number and WABA, I did **not** invoke `/<PHONE_NUMBER_ID>/register` or try deleting, migrating, or moving them.

I instead:

1. Created a **dedicated system user** with access only to my WABA and app, using `whatsapp_business_management` and `whatsapp_business_messaging`.
2. Checked this token's access and that it could not read the four other WABAs known to my integration.
3. Stored the token in a **server-side secret vault**, not browser JavaScript or a broadly readable database column.
4. Tested local channel creation, tenant isolation, idempotency and rollback before touching production.
5. Created the missing **local channel** for that existing phone number.
6. Subscribed the existing app to its WABA using Meta's documented endpoint:

```http
POST https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
Authorization: Bearer <SCOPED_SYSTEM_USER_TOKEN>
```

7. Read back `subscribed_apps` and the channel state to confirm both steps were effective.

**Observed outcome, November 2, 2026:** the channel appeared in my SaaS, the app was subscribed to the WABA, and **a new message sent from another phone arrived both in WhatsApp Business and in my SaaS inbox**.

That recovered **inbound routing** without repeating Embedded Signup. It **does not** establish that #1690130 is fixed in Meta, that outbound sends and echoes have been verified, or that history sync is supported.

This is an **advanced developer recovery approach**, appropriate only when you can verify the real asset state, have proper permissions, and safely isolate your integration. It is **not** a general-purpose reset button for WhatsApp users.

## 6. History/contact sync returned 131000 / error_subcode 10

After inbound delivery worked, I requested the app-state/contact sync and the history sync through `/<PHONE_NUMBER_ID>/smb_app_data`. Both were **rejected**:

```text
code: 131000
error_subcode: 10
This API is only available to WhatsApp Business App phone numbers
onboarded on to WhatsApp Business Platform
```

At the same time, the Graph phone indicators still showed `CONNECTED`, `CLOUD_API`, `is_on_biz_app: true`.

**The important lesson:** receiving a Cloud API webhook and seeing `is_on_biz_app: true` **does not guarantee** that Meta considers the number fully onboarded **for the `smb_app_data` sync feature**.

This is **consistent with**, but does not prove, the incomplete-onboarding hypothesis. The error did **not** say the 24-hour sync window had expired or that the user denied history sharing. I did not retry.

I kept the relevant `fbtrace_id` and signup session ID privately for Meta support. **No historical chats or contacts were imported at the time of writing.** Separately, changing from an old phone to a tablet without restoring a backup can leave old device chats unavailable; importing Cloud API history and restoring a device backup are two different processes.

## 7. Diagnostic checklist

- **#3441034:** check WhatsApp Business → Settings → Account → Business Platform, especially an existing **Meta Business Suite Inbox** connection. Don't assume that is the cause in every case.
- **#2494064:** confirm that you are in the coexistence flow rather than standard registration or migration.
- **#1690130:** verify the portfolio and check whether Graph already knows the phone, WABA and subscriptions. Avoid blindly repeating a half-finished signup.
- **Number CONNECTED but inbox empty:** inspect `subscribed_apps`, webhook configuration, local channel persistence and real delivery.
- **131000 / 10 on `smb_app_data`:** retain the trace IDs and ask Meta to explain the sync eligibility/onboarding-state mismatch. Don't claim history sync succeeded just because incoming messages work.
- **141006:** investigate billing restrictions separately from webhook receipt.
- **Security:** use scoped credentials and server-side storage; don't log or publish tokens, phone numbers, portfolio IDs or session identifiers.

## What remains unresolved

The exact internal reason for **#1690130** is still unknown. Meta still needs to clarify why this number accepted incoming Cloud API messages but rejected `smb_app_data` as ineligible for WhatsApp Business App history/contact sync. I am **not** claiming that changing the app's portfolio ownership would fix it or that I recovered historical chats.

## References

- [Meta for Developers — Onboarding WhatsApp Business app users](https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users).
- [Meta's Postman collection — Subscribe to your WABA](https://www.postman.com/meta/whatsapp-business-platform/request/c1ai24q/subscribe-to-your-waba).
- [Bird — WhatsApp coexistence troubleshooting](https://bird.com/docs/guides/whatsapp/coexistence/troubleshooting).
- [WACO — Existing WhatsApp Business app number](https://docs.waco.id/en/existing-number).

---

**By Martín Muñoz** · [GitHub](https://github.com/martinmunozb). If you encountered these errors, consider sharing the **error code, step and sanitized Graph/WABA subscription state** rather than posting private account identifiers.
