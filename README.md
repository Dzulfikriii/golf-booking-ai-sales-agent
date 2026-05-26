# Golf Booking AI Sales Agent

> A hybrid human-AI system that automates golfer-facing WhatsApp communication while keeping a human in the loop for club-side coordination. Built end-to-end: AI conversation engine, reactive operations dashboard, and a hardened webhook ingress.

**Stack:** Convex · Next.js 14 · FastAPI · OpenAI · WhatsApp Cloud API · Nodemailer
**Role:** Solo build — architecture, backend, frontend, AI prompt design, ops.

---

## Demo

[**Watch the 5-minute system walkthrough →**](https://drive.google.com/file/d/1INl1hRP4sky_E-mgJGxSnVq5Cf31jDne/view?usp=sharing)

The video walks an end-to-end booking: golfer DMs the WhatsApp number → AI collects intent → human agent reviews the dashboard, calls the club, marks the slot available → AI confirms with the golfer, requests payment, ingests the screenshot → human verifies → AI sends the final confirmation. Every state transition fires the right WhatsApp message and email automatically.

![Bookings dashboard](assets/screenshots/bookings-list.png)
![Conversation view with payment proof](assets/screenshots/conversation-view.png)

---

## The problem

A small team of human sales agents was hand-running every step of every golf booking: reading WhatsApp messages, phoning clubs to check availability, relaying answers back, chasing payment screenshots, then phoning the club again to confirm. Linear in headcount, error-prone, and the founder couldn't add bookings without adding people.

**The constraint that shaped the system:** the *club side* of the workflow can't be automated. Clubs only accept phone calls. So this is not a "replace humans with AI" project — it's "automate the half of the job that's safe to automate, and give the human a cockpit for the rest."

## The solution

A hybrid system with a clear seam between **what the AI owns** and **what the human owns**:

| AI owns | Human owns |
|---|---|
| WhatsApp intake — extracting club, date, time, players in any order, in Bahasa Melayu or English | Phoning the club to check availability |
| Status-triggered outbound messages (slot available, payment instructions, final confirmation) | Verifying payment screenshots |
| Payment-proof image ingestion → file storage → status transition | Resolving anything the AI is uncertain about |
| 24-hour silence follow-ups | Marking the club as notified / booking complete |

The dashboard is the seam. When the human flips a status, the AI picks up and writes the next message. The human never opens WhatsApp.

## Architecture (one screen)

```
WhatsApp (Meta Cloud API)
        │   GET  /webhook/whatsapp  — verify-token handshake
        │   POST /webhook/whatsapp  — HMAC-signed inbound + status callbacks
        ▼
┌─────────────────────────┐
│  FastAPI ingress        │  Validates X-Hub-Signature-256, normalises payload,
│  (the only Python svc)  │  forwards to a Convex HTTP action.
└────────────┬────────────┘
             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Convex (TypeScript)                                            │
│                                                                 │
│  • schema: bookings, golfers, clubs, conversations,             │
│    activityLog, agents, settings                                │
│  • mutations: createPending, transitionStatus, …                │
│  • internal actions: ai.handleIncomingMessage,                  │
│    ai.handleStatusTransition, statusTransitions.sendEmailsFor…  │
│  • cron: 24h silence follow-up                                  │
│  • file storage: payment-proof images                           │
└────────────┬───────────────────────────────────┬────────────────┘
             ▼                                   ▼
┌─────────────────────────┐         ┌──────────────────────────┐
│  Next.js 14 dashboard   │         │  Outbound: OpenAI,       │
│  (Convex live queries)  │         │  WhatsApp Cloud, SMTP    │
└─────────────────────────┘         └──────────────────────────┘
```

Three deployables, one source of truth (Convex), zero polling — the dashboard reacts to mutations the AI makes, the AI reacts to mutations the dashboard makes.

→ **[Read the full architecture case study](docs/architecture.md)**

## What was interesting to build

Four design decisions worth defending in an interview:

1. **One LLM call per inbound message, not a chain.** A single OpenAI call returns `{ reply, collected: {club, date, time, players}, intent }` as structured JSON. Reply generation, slot extraction, and intent classification share the same context — no router agent, no second hop, no JSON-mode-on-top-of-prose. → [AI agent design](docs/ai-agent-design.md)

2. **Convex instead of FastAPI + Postgres + Celery + Redis.** The PRD asked for the latter. I shipped Convex. The dashboard gets reactivity for free, scheduled actions replace Celery, file storage replaces S3 wiring. The Python service shrank to a 200-line webhook ingress that only exists because Meta posts to one HTTPS endpoint that needs HMAC validation. → [Stack rationale](docs/stack-rationale.md)

3. **A status machine the UI is generated from.** Twelve statuses, each with a set of legal next statuses, each with a set of action buttons. The dashboard renders buttons by reading the current status — no per-page state logic. → [Architecture](docs/architecture.md#booking-status-state-machine)

4. **Mock-first integrations.** Every external service (OpenAI, WhatsApp, SMTP, Convex-from-Python) has a `mock` and a `real` mode switched by env var. The entire system runs end-to-end with zero accounts — critical for fast iteration and for shipping a demo without burning live message quota.

## Stack — and why

| Layer | Choice | Why this one |
|---|---|---|
| **DB + backend logic** | Convex | One platform: schema, queries, mutations, actions, scheduled jobs, file storage, live websockets to the client. Removed Postgres, Celery, Redis, and a separate FastAPI app from the original spec. |
| **AI** | OpenAI (`gpt-4o-mini`) via structured-output JSON | One call returns reply + extracted fields + intent. Cheap, fast, deterministic enough for this domain. |
| **WhatsApp** | Meta WhatsApp Cloud API | First-party, free inbound, predictable webhook contract. (The non-obvious gotcha: WABA subscription is a *list* the app must be in, not just an app-level toggle — silent inbound failure if you miss it.) |
| **Ingress** | FastAPI | HMAC-SHA256 validation of the raw body needs Python's `hmac` module and tight control over middleware ordering. ~200 LOC, single endpoint, easy to harden. |
| **Dashboard** | Next.js 14 (App Router) + Convex React client | Live queries, no `useEffect` polling, no `swr`/`react-query`. Status changes in the AI's path are visible in the operator's UI within a tick. |
| **Email** | Nodemailer over SMTP | App-password Gmail in dev, swap to transactional SMTP (Postmark/SES) in prod with no code change. |

→ **[Full stack rationale and the trades I made](docs/stack-rationale.md)**

## Business impact

The PRD's North Star: **reduce manual messaging workload by ≥70%.**

The split, conservatively:

| Activity | Pre-system (human) | Post-system | Time saved per booking |
|---|---|---|---|
| Read intake msg & extract club/date/time/players | ~3 min | 0 — AI does it | 3 min |
| Acknowledge & clarify with golfer | ~2 min | 0 — AI does it | 2 min |
| Phone the club | ~5 min | ~5 min | 0 (intentionally) |
| Relay availability to golfer + request confirmation | ~3 min | 0 — AI does it | 3 min |
| Send payment instructions, receive screenshot, file it | ~5 min | 0 — AI does it, stores it | 5 min |
| Verify payment | ~1 min | ~1 min | 0 (intentionally) |
| Confirm with club + send final confirmation | ~4 min | ~2 min (club call only) | 2 min |
| **Total** | **~23 min** | **~8 min** | **~15 min (≈65%)** |

In a 50-booking week, that's ~12.5 hours of agent time back. Operationally more important: the human stops being the typing bottleneck, so each agent can carry ~3× the booking volume before adding headcount.

→ **[Full business impact breakdown](docs/business-impact.md)**

## Repo layout (in this portfolio)

```
.
├── README.md                  ← you are here
├── docs/
│   ├── architecture.md        ← deep dive: state machine, webhook contract, reactive flow
│   ├── stack-rationale.md     ← why Convex over Celery/Redis, why FastAPI for ingress
│   ├── ai-agent-design.md     ← single-LLM-call structured output pattern
│   └── business-impact.md     ← labour reduction math + scaling story
└── assets/
    └── screenshots/           ← UI captures (bookings list, conversation view, …)
```

## License

MIT — see [LICENSE](LICENSE).

---

_Source code lives in a separate private repo. This portfolio is the public-facing case study; reach out if you'd like a code walk-through._
