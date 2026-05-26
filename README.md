# Golf Booking AI Sales Agent

A hybrid human + AI system for handling golf-club tee-time bookings over WhatsApp. The AI handles the golfer-facing conversation. A human operator handles club-side coordination from a dashboard. Both work against the same source of truth and stay in sync without anyone refreshing a page.

**Stack:** Convex · Next.js 14 · FastAPI · OpenAI · WhatsApp Business Cloud API · Nodemailer

---

## Demo

[Watch the system walkthrough](https://drive.google.com/file/d/1INl1hRP4sky_E-mgJGxSnVq5Cf31jDne/view?usp=sharing)

The video runs through a full booking end-to-end: golfer messages the WhatsApp number, the AI collects intent, the operator calls the club and marks the slot available, the AI confirms with the golfer, the golfer sends a payment screenshot, the operator verifies and marks the booking complete. Each status change triggers the right outbound WhatsApp message and email automatically.

**Inbound conversation — club discovery and prompt-injection deflection**

The AI lists active clubs from the database when asked, and refuses an obvious prompt-injection attempt without breaking character.

<img src="assets/screenshots/1.jpeg" alt="WhatsApp conversation showing the AI listing supported golf clubs and deflecting a prompt-injection attempt" width="380" />

**Booking intake — natural-language slot extraction**

The golfer states club, date, and time in one sentence. The AI extracts the fields, asks only for what's missing (number of players), and recaps before handing the booking off to the operator.

<img src="assets/screenshots/2.jpeg" alt="WhatsApp conversation showing the AI collecting booking details for Cyberjaya Lake Club" width="380" />

---

## What it does

Small sales teams handling golf bookings spend most of their day typing the same messages: greeting the golfer, asking for date and time, relaying availability, sending bank details, asking for payment proof, then writing a confirmation. The actual judgement calls (phoning the club, verifying a payment screenshot) take a minute or two each. The typing takes hours.

This system automates the typing.

- Inbound WhatsApp messages are parsed by the AI, which extracts club, date, time, and number of players from natural language (English or Bahasa Melayu) and creates a booking in the database.
- The operator sees the booking on a dashboard, calls the club, and flips a status with a single button click.
- The status change fires an AI-composed reply back to the golfer. Payment instructions, slot confirmations, and final booking details are all generated and sent without anyone typing.
- Payment screenshots sent over WhatsApp are downloaded, stored, and attached to the booking automatically. The operator just needs to look at the image and click "Verify Payment".

The operator never opens WhatsApp.

## Architecture

```
WhatsApp (Meta Cloud API)
        │   GET  /webhook/whatsapp   verification handshake
        │   POST /webhook/whatsapp   HMAC-signed messages + status callbacks
        ▼
┌──────────────────────────┐
│  FastAPI ingress         │  validates X-Hub-Signature-256, normalises
│                          │  the payload, forwards to Convex
└────────────┬─────────────┘
             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Convex (TypeScript)                                             │
│   schema, mutations, internal actions, scheduled jobs,           │
│   file storage for payment-proof images                          │
└────────────┬─────────────────────────────────────┬───────────────┘
             ▼                                     ▼
┌──────────────────────────┐         ┌──────────────────────────┐
│  Next.js 14 dashboard    │         │  Outbound: OpenAI,       │
│  (Convex live queries)   │         │  WhatsApp, SMTP          │
└──────────────────────────┘         └──────────────────────────┘
```

Three deployables, one database. The dashboard subscribes to Convex queries over a websocket; any state the AI changes is on the operator's screen within a tick, and vice versa. There is no polling layer.

Full write-up: [docs/architecture.md](docs/architecture.md)

## Stack notes

| Layer | Choice |
|---|---|
| Database, server logic, scheduled jobs, file storage | Convex |
| Webhook ingress | FastAPI (the only Python service; isolates HMAC validation) |
| AI | OpenAI `gpt-4o-mini` with structured JSON outputs |
| WhatsApp transport | Meta WhatsApp Business Cloud API |
| Operator dashboard | Next.js 14 (App Router) + Convex React client |
| Email | Nodemailer over SMTP |

Most of the system runs on Convex because the booking workflow is heavily state-driven: a small number of tables, lots of transitions, and a UI that needs to react instantly when state changes. Convex collapses the "database + jobs + websocket sync to the client" stack into one runtime. The Python service exists only because Meta posts to a single HTTPS endpoint that has to validate a raw-body HMAC signature, which is cleaner to do in isolation.

More on the trades: [docs/stack-rationale.md](docs/stack-rationale.md)

## AI design

The AI agent runs as a single OpenAI call per inbound message. The response is a structured object containing the reply text, the booking fields the model extracted (`club`, `date`, `time`, `players`), and an intent classification (`providing_info`, `confirming_yes`, `confirming_no`, `other`).

There is no router agent, no separate extractor, no chained calls. The action that handles the message uses the `intent` field to branch in TypeScript: a `confirming_yes` on a slot-available booking walks the booking through to `awaiting_payment`, which fires the bank-details message via a templated path. Anything else falls through to a default "persist what we learned, send the reply" branch.

Conversation history is bounded to the last twelve messages. The system prompt is anchored on the current date and the list of active clubs from the database, so the model doesn't hallucinate club names or misread "Saturday".

Details: [docs/ai-agent-design.md](docs/ai-agent-design.md)

## Business impact

The system is built around a specific observation: in the typical booking, the operator's time goes mostly to message-typing rather than to the parts of the job that need human judgement. Removing the typing leaves the operator free to handle three to four times the booking volume without adding headcount, and lets the AI keep covering inbound messages outside office hours.

Cost to run is small enough not to matter at the operating scale. The dominant operational cost is people-time, which is the cost the system actually attacks.

More: [docs/business-impact.md](docs/business-impact.md)

## Repo contents

```
.
├── README.md
├── LICENSE
├── docs/
│   ├── architecture.md
│   ├── stack-rationale.md
│   ├── ai-agent-design.md
│   └── business-impact.md
└── assets/
    └── screenshots/
```

Source code is in a separate private repository.

## License

MIT.
