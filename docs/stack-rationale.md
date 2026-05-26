# Stack rationale

The PRD asked for FastAPI + Postgres + Celery + Redis + a Next.js dashboard talking to FastAPI over REST. I shipped a different stack. This doc explains why, what it cost, and what it bought.

## TL;DR

| Original spec | What I shipped | Net |
|---|---|---|
| FastAPI app for business logic | Convex (TypeScript) | -1 deployable, -1 ORM, +reactive queries |
| Postgres | Convex tables | -1 connection pool to tune |
| Celery + Redis | Convex scheduled actions / crons | -2 services to host |
| S3 (or similar) | Convex file storage | -1 SDK, -1 IAM policy to write |
| FastAPI ingress | FastAPI ingress | Kept — Meta posts to one HTTPS URL that *must* validate HMAC |
| Next.js → REST → FastAPI | Next.js → Convex live queries | No polling layer, no SWR/react-query |

## Why Convex (and not Postgres + Celery + Redis)

The original architecture is fine. It's the default for a reason. I picked Convex because the *cost of moving parts* in a small-team product is the dominant cost, and Convex collapses five concerns into one:

1. **Schema + queries + mutations + actions** in one runtime, in one language, in one repo.
2. **Scheduled actions** (`ctx.scheduler.runAfter(...)`) replace Celery. The "send a 24h silence follow-up" cron is ~15 LOC.
3. **File storage** with signed-URL semantics replaces an S3 setup. Payment-proof image is `await ctx.storage.store(blob)`.
4. **Live queries to the React client over a websocket** replace the entire "how do I push UI updates" question. No SSE, no polling, no optimistic-state-with-rollback. `useQuery(api.bookings.list)` is the subscription.
5. **Transactional mutations** — a status transition that writes to `bookings`, appends to `activityLog`, and schedules the AI action either all commits or all rolls back. Zero distributed-transaction gymnastics.

### What I gave up

- **Vendor lock-in.** Moving off Convex would mean re-implementing the schema against Postgres and re-implementing actions against a job runner. Real cost, real consideration.
- **Raw SQL.** Convex's query language is intentionally constrained (indexed equality + range). I would not pick it for a product with heavy ad-hoc analytics. For an operational dashboard with a known query set, the constraint is an asset — every query is fast by construction.
- **Local-only running.** Convex needs `npx convex dev` to maintain a local sync to its cloud. Tolerable; not love-it.

### What I would do instead at 10× scale

If this product ever needed to support hundreds of concurrent agents and millions of bookings, I would peel out the booking + conversation tables to Postgres behind a domain service, keep Convex for the operator-side reactivity layer, and let Convex sync from Postgres. I would not preemptively do that today — premature scaling decisions are a tax on iteration speed.

## Why FastAPI for the webhook ingress (and only there)

Meta WhatsApp Cloud API posts to one HTTPS endpoint. That endpoint *must*:

1. Read the **raw** request body (the HMAC is over bytes, not parsed JSON).
2. Compute `HMAC-SHA256(body, app_secret)` in constant time.
3. Compare to `X-Hub-Signature-256` and reject on mismatch.

I picked FastAPI because:

- Python's `hmac.compare_digest` is the canonical constant-time comparator and the surrounding ecosystem (`fastapi.Request.body()`) makes "read raw bytes before any middleware parses them" a one-liner.
- The ingress has no business logic. It validates, normalises to a flat JSON shape, and forwards to Convex. ~200 LOC, easy to read top-to-bottom, easy to audit.
- It is the only public-internet endpoint that accepts data from an unauthenticated source. Isolating it on its own deployable shrinks the *trusted surface* — a CVE in some unrelated Convex function can't reach the HMAC validator.

I considered putting the HMAC check in a Convex HTTP action directly. Two reasons I didn't:

- Convex HTTP actions receive a parsed body; reaching back for the raw bytes works but is fiddly and version-dependent.
- The hostname matters. Meta's webhook setup wants a single stable URL; keeping it on a dedicated ingress means I can swap Convex deploys without rotating the URL with Meta.

## Why Next.js + Convex React client (and not REST)

The dashboard's whole job is to keep three views in sync with state that's being changed by *both* humans and the AI: the bookings list, the per-booking detail panel, and the conversation thread.

The REST-style implementation requires:
- A polling cadence ("refetch every 5s") or
- An SSE/WebSocket layer ("when a booking changes, push the new row") or
- Optimistic state with rollback logic

All three are a non-trivial amount of code, and each has its own bug class.

Convex's React client gives this for free. `useQuery(api.bookings.list)` registers a subscription. When a mutation modifies a row the query touches, the client gets a new snapshot. There's no `useEffect`, no `setInterval`, no `swr`, no `react-query`. The query *is* the subscription.

This is the single biggest reason the dashboard code is small.

## Why OpenAI (with structured output) for the AI

I chose `gpt-4o-mini` with the structured-output `response_format` API. Three reasons:

1. **One call, multiple jobs.** Reply text + extracted booking fields + intent classification share the same context window. A router-then-extractor-then-replier pipeline is more total tokens *and* introduces sync issues between the steps.
2. **JSON mode is reliable.** Structured outputs guarantee the response parses against a schema. No "the model returned `Sure, here is your JSON: {…}`" failure mode.
3. **Cost & latency for this domain.** Booking intake is closed-vocabulary (clubs, dates, players). The cheap model is enough.

See [ai-agent-design.md](ai-agent-design.md) for the full prompt design and intent-routing logic.

## Why mock-first integrations

Every external client (`lib/openai.ts`, `lib/whatsapp.ts`, `lib/email.ts`) exports the same interface in both modes. An env var (`OPENAI_MODE`, `WHATSAPP_MODE`, `EMAIL_MODE`) picks `mock` or `real` at runtime. In mock mode the function logs the payload to stdout and returns a canned response.

This bought three things:

- **End-to-end demos with zero accounts.** Pull the repo, `npm install`, `npx convex dev`, browse to the dashboard, and the system runs. Critical for showing the system to non-technical stakeholders without burning live WhatsApp message quota.
- **Deterministic tests.** Unit and smoke tests run against the mocks; no flaky external dependencies.
- **One-line cutover to prod.** Set `*_MODE=real`, ship.

## What I'd add if this were going to production for real

These are out of scope for the portfolio build but worth naming:

- **Tracing.** OpenTelemetry from FastAPI → Convex actions, with the WhatsApp `wamid` as the correlation ID. Right now I lean on Convex's per-action logs and structured `console.log` lines; that's fine until it isn't.
- **Idempotency keys** on the inbound webhook. Meta retries on 5xx; we already dedupe by `providerMessageId`, but I'd make this explicit at the HTTP layer.
- **Rate limiting per golfer.** A single golfer firing 20 messages in 5 seconds shouldn't fan out to 20 OpenAI calls. Easy with a Convex mutation that checks a sliding window.
- **Human override hand-off.** Right now the AI is always on. A toggle on the booking detail page that pauses AI replies and lets the operator type directly — useful for the 1% of conversations that need bespoke handling.
- **Audit log export.** `activityLog` is in Convex; a daily dump to cold storage (S3 / GCS) would be table stakes for any deployment with compliance obligations.
