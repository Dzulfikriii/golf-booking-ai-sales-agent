# Architecture

A guided tour of the system. Three deployables, one database, one source of truth for state — the booking status.

## The three services

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   WhatsApp                                                              │
│  (Meta Cloud API)                                                       │
│        │                                                                │
│        │ 1. inbound msg / status callback                               │
│        ▼                                                                │
│  ┌─────────────────────┐                                                │
│  │ FastAPI ingress     │ — verify HMAC, normalise payload, fan out      │
│  │  /webhook/whatsapp  │                                                │
│  └──────────┬──────────┘                                                │
│             │ 2. POST /ingress/whatsapp (Convex HTTP action)            │
│             ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Convex (the system of record)                                   │   │
│  │                                                                  │   │
│  │  • schema (bookings, golfers, clubs, conversations, …)           │   │
│  │  • mutations  (transactional state writes)                       │   │
│  │  • internal actions (LLM calls, WhatsApp send, email send)       │   │
│  │  • cron (24h silence follow-up)                                  │   │
│  │  • storage (payment-proof images)                                │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │ 3. live-query push                    │
│                                 ▼                                       │
│                       ┌───────────────────────┐                         │
│                       │  Next.js dashboard    │                         │
│                       │  (Convex React client)│                         │
│                       └───────────────────────┘                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why three, not one?**
- The webhook ingress is Python because HMAC-SHA256 validation needs the raw request body, and isolating it shrinks the trusted surface for a public-internet endpoint to ~200 LOC.
- Convex is the only place state lives. No shadow tables, no caches that need invalidating.
- The dashboard never talks to FastAPI. It subscribes to Convex queries and renders. State the AI changes shows up on the operator's screen without a refresh.

## Booking status state machine

Twelve statuses. Every transition is either golfer-driven (via the AI) or human-driven (via the dashboard). The status drives both the UI (which buttons render) and the AI (what message to send next when status changes).

```
                     ┌─────────┐
   golfer message ──▶│ pending │
                     └────┬────┘
                          │  agent: "Mark Checking Availability"
                          ▼
              ┌───────────────────────┐
              │ checking_availability │
              └────────┬──────────────┘
        agent: Slot   │     agent: Slot
        Available     │     Unavailable
                ▼     │     ▼
   ┌────────────────┐ │ ┌────────────────────┐
   │ slot_available │ │ │ slot_unavailable   │
   └───────┬────────┘ │ └───────┬────────────┘
           ▼          │         ▼
┌──────────────────────────┐  AI offers alternatives
│ awaiting_golfer_         │  → new pending booking
│ confirmation             │
└────────┬─────────────────┘
   golfer replies "yes"  (AI detects intent)
         ▼
  ┌──────────────────┐
  │ golfer_confirmed │
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ awaiting_payment │  ← AI auto-sends bank details + payment instructions email
  └────────┬─────────┘
   golfer sends image  (AI stores it, transitions status)
         ▼
  ┌──────────────────┐
  │ payment_received │
  └────────┬─────────┘
   agent: "Verify Payment"
         ▼
  ┌──────────────────┐
  │ payment_verified │
  └────────┬─────────┘
   agent: "Mark Club Notified"
         ▼
  ┌──────────────────┐
  │ club_notified    │
  └────────┬─────────┘
   agent: "Mark Completed"
         ▼
  ┌──────────────────┐
  │ completed        │  ← AI sends final confirmation + booking confirmation email
  └──────────────────┘                                       + club notification email

Any state ──▶ cancelled  (Cancel Booking button, always available)
```

Each status has exactly one set of legal next statuses. The mutation `bookings.transitionStatus` validates the transition, writes the new status atomically, appends to `activityLog`, and schedules two side-effect actions:
- `ai.handleStatusTransition` — composes and sends the WhatsApp message for the new status.
- `statusTransitions.sendEmailsForStatus` — picks the right email template for the new status (or no-op).

**Two paths to the same status.** `awaiting_payment` is reached either by the human flipping it manually or by the AI promoting the booking when the golfer says "yes". Both paths fund the same action — there's no second copy of the side-effect logic.

## Webhook contract

Meta WhatsApp Cloud API hits the FastAPI ingress in two shapes:

### Verification (GET)
```
GET /webhook/whatsapp?hub.mode=subscribe
                     &hub.verify_token=…
                     &hub.challenge=…
```
We compare `hub.verify_token` to the configured value and echo `hub.challenge` on match.

### Delivery (POST)
Every inbound message and every outbound status callback. We:

1. Read the raw bytes (NOT the parsed JSON — the signature is over the raw body).
2. Compute `HMAC-SHA256(raw_body, META_APP_SECRET)` and compare to `X-Hub-Signature-256` in constant time.
3. Reject on mismatch with a 401.
4. Parse and normalise to a single shape:

```json
{
  "from": "+60123456789",
  "type": "text" | "image" | "document",
  "text": "Looking for a slot at Saujana on Saturday morning",
  "mediaId": null,
  "providerMessageId": "wamid.HBgL…"
}
```

5. POST it to a Convex HTTP action `/ingress/whatsapp`, which schedules the internal action `ai:handleIncomingMessage`. Convex resolves any image `mediaId` into bytes via the Meta Graph API and stores it in Convex file storage.

The ingress doesn't make business decisions. It validates, normalises, hands off.

## Inbound message flow (golfer → AI)

```
WhatsApp inbound POST
       │
       ▼  validated by FastAPI
Convex http action /ingress/whatsapp
       │
       ▼  schedules
ai.handleIncomingMessage (internal action)
       │
       ├─ upsert golfer by WhatsApp number
       ├─ ensure conversation exists, append user message
       │
       ├─ if image + booking is awaiting_payment:
       │     download media → file storage → transition to payment_received
       │     reply "got it, we'll confirm shortly"
       │     return
       │
       ├─ ONE OpenAI call returns { reply, collected, intent }
       │
       ├─ intent = confirming_yes & booking is awaiting confirmation:
       │     chain transitions: slot_available → awaiting_golfer_confirmation
       │                        → golfer_confirmed → awaiting_payment
       │     (each transition fires its own side effects via the same mutation)
       │     return  ← AI doesn't reply directly; the awaiting_payment transition
       │              auto-sends bank details
       │
       ├─ intent = confirming_no & awaiting confirmation:
       │     transition booking → cancelled, send reply
       │
       ├─ all four fields collected & no active booking:
       │     create booking (status=pending), link to conversation, send ack
       │
       └─ default:
             update conversation stage + collected fields, send reply
```

The clever bit: the AI **never sends a hardcoded "what date would you like" message**. It generates the reply contextually from history + what's been collected so far + what's missing. The state machine only nudges which question is most appropriate to ask next.

## Outbound flow (dashboard human → golfer)

```
Operator clicks "Slot Available"
       │
       ▼  Next.js calls mutation
bookings.transitionStatus({ id, toStatus: "slot_available" })
       │
       ├─ validate legal transition
       ├─ write status + updatedAt
       ├─ append activityLog entry
       │
       ├─ schedule ai.handleStatusTransition
       │     │
       │     ▼ composes: "Good news! {club} has a slot on {date} at {time}…"
       │       sendMessage via WhatsApp Cloud API
       │       append assistant message to conversation
       │       update conversation stage to awaiting_confirmation
       │
       └─ schedule statusTransitions.sendEmailsForStatus
             │
             ▼ picks slotAvailableEmail template
               sendEmail to golfer
```

The dashboard renders the new status the instant the mutation lands — same Convex query that the human just wrote to. No "refresh to see changes". The conversation panel updates as the AI appends its reply.

## Reactive dashboard (no polling, no useEffect)

Every page is a thin shell around `useQuery(api.bookings.list)` or similar. Convex pushes new data over a websocket whenever a mutation touches a row the query depends on.

Concretely: when the AI's `handleIncomingMessage` appends a message to the conversation, the operator's `BookingDetail` page re-renders with the new message — without polling, without manual cache invalidation, without optimistic updates that need rollback logic. The query *is* the subscription.

This is the single biggest reason the codebase is small. There is no React state for "current booking list" that has to be kept in sync with the server.

## File storage

Payment-proof images are downloaded from Meta's media endpoint (24-hour signed URL) and re-stored in Convex file storage immediately on receipt. Two reasons:
1. The Meta URL expires; the booking record needs to point at something durable.
2. We only ever expose the proof to authenticated dashboard users via a Convex storage URL — no static path is leaked.

## Failure surfaces

| Failure | Behaviour |
|---|---|
| Webhook HMAC mismatch | 401, no DB write, no AI call. Logged with a signature-prefix excerpt. |
| OpenAI 5xx / timeout | Action retries via Convex's built-in action retry. User-facing message is delayed, never duplicated (idempotent on conversation `providerMessageId`). |
| WhatsApp send fails | Conversation message is appended *after* successful send; on failure we don't poison the conversation log. Status transition still committed (the operator can re-trigger the send). |
| Golfer goes silent 24h | Cron `crons.ts` sweeps bookings in `awaiting_payment` / `awaiting_golfer_confirmation` with `lastSilencePingAt < now − 24h` and sends one polite follow-up. Marked so we don't ping twice. |
| Convex action transient error | Built-in retry with exponential backoff. The inbound message was persisted before any external call, so retries are safe. |

## Why this shape, not the PRD's shape

The PRD asked for FastAPI + Postgres + Celery + Redis + a Next.js dashboard that calls FastAPI over REST. I shipped Convex + a thin FastAPI ingress + a Convex-native dashboard.

The trade I made:

- **Lost:** vendor portability. The entire backend is on Convex; moving would mean rewriting the schema and actions against another runtime.
- **Gained:** ~half the moving parts, native reactivity (no SSE / polling layer to build), built-in scheduled actions (no Celery worker to host), built-in file storage (no S3 wiring), and a single deployment surface.

For a system this size, run by a small team, the gain is wildly larger than the loss. See [stack-rationale.md](stack-rationale.md) for the full argument.
