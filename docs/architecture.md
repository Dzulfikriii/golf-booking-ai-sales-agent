# Architecture

Three services, one database. The booking status is the source of truth for every other piece of behaviour in the system.

## Services

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   WhatsApp                                                              │
│  (Meta Cloud API)                                                       │
│        │                                                                │
│        │  inbound msg / status callback                                 │
│        ▼                                                                │
│  ┌─────────────────────┐                                                │
│  │ FastAPI ingress     │  verify HMAC, normalise payload, forward       │
│  │  /webhook/whatsapp  │                                                │
│  └──────────┬──────────┘                                                │
│             │  POST /ingress/whatsapp (Convex HTTP action)              │
│             ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Convex                                                          │   │
│  │   schema, mutations, internal actions, crons, file storage       │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │  live-query push                      │
│                                 ▼                                       │
│                       ┌───────────────────────┐                         │
│                       │  Next.js dashboard    │                         │
│                       └───────────────────────┘                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

The webhook ingress is Python because HMAC-SHA256 verification has to happen against the raw request body, and keeping that on its own service shrinks the public-internet attack surface to a couple of hundred lines of code. The ingress does no business logic; it validates, normalises to a flat shape, and forwards.

Convex holds all state. The dashboard never talks to the FastAPI service; it subscribes to Convex queries directly. When the AI mutates a row, the operator's UI updates without a refresh. When the operator flips a status, the AI sees it through the same query layer.

## Booking status machine

Twelve statuses. Every transition is either golfer-driven (through the AI) or operator-driven (through the dashboard). The status drives the dashboard (which action buttons render) and the AI (what message to send when the status changes).

```
                     ┌─────────┐
   golfer message ──▶│ pending │
                     └────┬────┘
                          │  operator: Mark Checking Availability
                          ▼
              ┌───────────────────────┐
              │ checking_availability │
              └────────┬──────────────┘
        operator:     │     operator:
        Slot Available│     Slot Unavailable
                ▼     │     ▼
   ┌────────────────┐ │ ┌────────────────────┐
   │ slot_available │ │ │ slot_unavailable   │
   └───────┬────────┘ │ └───────┬────────────┘
           ▼          │         ▼
┌──────────────────────────┐  AI offers alternatives
│ awaiting_golfer_         │  → new pending booking
│ confirmation             │
└────────┬─────────────────┘
   golfer replies yes (AI detects intent)
         ▼
  ┌──────────────────┐
  │ golfer_confirmed │
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ awaiting_payment │   ← AI sends bank details + payment-instructions email
  └────────┬─────────┘
   golfer sends image (AI stores it, transitions status)
         ▼
  ┌──────────────────┐
  │ payment_received │
  └────────┬─────────┘
   operator: Verify Payment
         ▼
  ┌──────────────────┐
  │ payment_verified │
  └────────┬─────────┘
   operator: Mark Club Notified
         ▼
  ┌──────────────────┐
  │ club_notified    │
  └────────┬─────────┘
   operator: Mark Completed
         ▼
  ┌──────────────────┐
  │ completed        │   ← AI sends final confirmation
  └──────────────────┘     + booking confirmation email + club notification email

Any state ──▶ cancelled (Cancel Booking button, always available)
```

Each transition runs through one mutation, `bookings.transitionStatus`. That mutation validates the move is legal, writes the new status, appends to `activityLog`, and schedules two side-effect actions: one to send the outbound WhatsApp message, one to send the right email (if any). Both branches share the same path whether the transition came from the AI or the operator, so there is no second copy of the side-effect logic.

## Webhook contract

The Meta WhatsApp Cloud API hits the FastAPI ingress in two shapes.

### Verification (GET)

```
GET /webhook/whatsapp?hub.mode=subscribe
                     &hub.verify_token=…
                     &hub.challenge=…
```

The verify token is compared to a configured value; on match we echo back `hub.challenge`.

### Delivery (POST)

For every inbound message and every outbound status callback. The handler:

1. Reads the raw bytes of the request body (not the parsed JSON; the signature is computed over bytes).
2. Computes `HMAC-SHA256(body, META_APP_SECRET)` and compares to `X-Hub-Signature-256` in constant time.
3. Returns 401 on mismatch.
4. Parses, normalises to a flat shape, forwards to a Convex HTTP action.

The normalised payload:

```json
{
  "from": "+60123456789",
  "type": "text" | "image" | "document",
  "text": "Looking for a slot at Saujana on Saturday morning",
  "mediaId": null,
  "providerMessageId": "wamid.HBgL…"
}
```

On the Convex side, `ai.handleIncomingMessage` is scheduled. Image messages are resolved through the Meta Graph API into bytes and stored in Convex file storage; the original media URL from Meta expires after 24 hours, so re-storing immediately is the only safe move.

## Inbound flow (golfer → AI)

```
WhatsApp inbound POST
       │
       ▼  validated by FastAPI
Convex http action /ingress/whatsapp
       │
       ▼  schedules
ai.handleIncomingMessage
       │
       ├─ upsert golfer by WhatsApp number
       ├─ ensure conversation, append the user message
       │
       ├─ if image + booking is awaiting_payment:
       │     download media → file storage → transition to payment_received
       │     reply "got it, we'll confirm shortly"
       │     return
       │
       ├─ one OpenAI call returns { reply, collected, intent }
       │
       ├─ intent confirming_yes & awaiting confirmation:
       │     transition booking through to awaiting_payment
       │     (bank details auto-sent by status-triggered path)
       │     return
       │
       ├─ intent confirming_no & awaiting confirmation:
       │     transition booking → cancelled, send reply
       │
       ├─ all four fields collected & no active booking:
       │     create pending booking, link to conversation, send reply
       │
       └─ default:
             update conversation stage + collected fields, send reply
```

The reply text is generated, never templated, on this path. The model sees the conversation history, the partial booking data collected so far, and the current stage, and writes whatever makes sense next.

## Outbound flow (operator → golfer)

```
Operator clicks "Slot Available"
       │
       ▼  Next.js calls mutation
bookings.transitionStatus({ id, toStatus: "slot_available" })
       │
       ├─ validate legal transition
       ├─ write status, append activityLog
       │
       ├─ schedule ai.handleStatusTransition
       │     composes "Good news! {club} has a slot on {date}…"
       │     sendMessage via WhatsApp Cloud API
       │     append assistant message to conversation
       │
       └─ schedule statusTransitions.sendEmailsForStatus
             pick the right template (slot available, completed, cancelled, …)
             sendEmail
```

Outbound messages on the status-triggered path are templated, not LLM-generated. They carry transactional data (bank account numbers, dates, player counts) where exactness matters more than tone variety.

## Reactive dashboard

Every page in the dashboard wraps a `useQuery(api.bookings.list)` (or similar). Convex pushes new snapshots over a websocket whenever a mutation touches a row the query depends on.

When the AI appends an assistant message to a conversation, the operator's open `BookingDetail` page renders the new message immediately. No polling, no optimistic updates, no manual cache invalidation. The query is the subscription.

This is the largest single reason the dashboard code is small.

## File storage

Payment-proof images move through three places. The image arrives at the FastAPI ingress as a `mediaId` reference. Convex fetches the bytes from Meta's media endpoint using the access token, then writes the blob to Convex file storage and saves the `_storage` ID on the booking. The dashboard renders the image via a Convex storage URL, which is short-lived and scoped.

## Failure surfaces

| Failure | Behaviour |
|---|---|
| Webhook HMAC mismatch | 401, no DB write, no AI call. Logged with a signature-prefix excerpt. |
| OpenAI 5xx / timeout | Action retries via Convex's built-in retry. The inbound user message was persisted before the LLM call, so retries are safe. |
| WhatsApp send fails | The assistant message is only appended to the conversation after a successful send, so the conversation log doesn't get poisoned with messages that never went out. |
| Golfer goes silent 24h | A cron sweeps `awaiting_payment` and `awaiting_golfer_confirmation` bookings older than 24h since the last ping and sends a single follow-up. `lastSilencePingAt` prevents double-pinging. |
