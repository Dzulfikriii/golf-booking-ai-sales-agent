# Stack rationale

A few notes on the technology choices and the trades behind them.

## Convex

Convex is the database, the server-side logic runtime, the job scheduler, the file store, and the live-query layer to the React client. One service, one language, one repo.

The booking workflow is small and heavily state-driven. The interesting properties are:

- A handful of tables with well-known access patterns.
- A lot of mutations that change state and need to fire side effects (send a message, send an email, schedule a follow-up).
- A dashboard whose entire job is to reflect state and let operators change it.

Convex fits this shape. Mutations are transactional, so a status change that writes to `bookings`, appends to `activityLog`, and schedules an AI action either all commits or all rolls back. Scheduled actions cover the cron use case (24h silence follow-up) without a separate worker. File storage covers payment-proof images without an S3 setup. The React client subscribes to queries over a websocket, so there is no polling layer and no SWR / react-query cache to keep in sync.

The trades:

- Vendor lock-in. The schema and actions are written against Convex's runtime. Moving to Postgres + a job runner is a real piece of work.
- Constrained query language. Convex queries are indexed equality and range. For an operational dashboard this is fine; for ad-hoc analytics it would not be.
- Local development requires a synced Convex dev deployment (`npx convex dev`). Tolerable, not love-it.

For the size and shape of this product, the gain from collapsing five concerns into one runtime is much larger than these trades. At a much larger scale I would probably split the booking tables out behind a domain service, but that is not the problem the system has today.

## FastAPI for the webhook ingress

The Meta WhatsApp Cloud API posts to a single HTTPS endpoint, and that endpoint has to verify a raw-body HMAC-SHA256 signature. Two things make Python the path of least resistance here:

- The `hmac.compare_digest` constant-time comparator is the canonical implementation.
- FastAPI exposes the raw request body in a one-liner before any middleware tries to parse it.

The ingress runs about 200 lines of code. It validates, normalises the payload to a flat shape, and forwards to a Convex HTTP action. It deliberately does not make business decisions; that all lives in Convex.

I considered handling validation inside a Convex HTTP action directly. Two reasons I didn't: reaching the raw body from inside Convex's HTTP action is more fiddly than from FastAPI, and isolating the public-facing endpoint on its own deployable means a bug elsewhere in the system can't reach the validator.

## Next.js + Convex React client

The dashboard's only job is to keep three views in sync with state that both humans and the AI are mutating: the bookings list, the per-booking detail panel, and the conversation thread.

The typical REST-style implementation needs polling, an SSE/WebSocket layer, or optimistic updates with rollback. All three carry a non-trivial amount of code and a bug class each.

`useQuery(api.bookings.list)` registers a subscription. When a mutation modifies a row the query touches, the client receives a fresh snapshot. There is no `useEffect`, no `setInterval`, no `swr`, no `react-query`. The query is the subscription.

## OpenAI with structured outputs

`gpt-4o-mini` with the JSON schema response format. One call per inbound message returns the reply text, the booking fields the model extracted, and an intent classification.

Structured outputs are reliable. The OpenAI API enforces the schema at decode time, so the response either parses or the call fails. There is no "the model wrapped my JSON in prose" failure mode to handle.

A pipeline of separate calls for routing, extraction, and reply generation would mean shipping the conversation history into each step and keeping three prompts in sync. One call sharing one context is simpler and cheaper. Details in [ai-agent-design.md](ai-agent-design.md).

## Mock-first integrations

Every external client (`lib/openai.ts`, `lib/whatsapp.ts`, `lib/email.ts`, `lib/convex_client.py`) exports the same interface in both `mock` and `real` mode. An environment variable picks which one runs at startup. Mock mode logs the payload to stdout and returns a canned response.

This buys three things. The whole system runs end-to-end with no external accounts, which is useful for demos and local development. Tests can run against the mocks deterministically. Switching to production is changing a single env var per service.
