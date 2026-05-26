# AI agent design

The AI does three things: extracts booking details from natural language, classifies the golfer's intent (e.g. "are they saying yes to the confirmation?"), and writes the next reply. All three happen in **one** OpenAI call per inbound message.

## The core decision: one call, not a chain

The naive design is a pipeline:
1. Classify intent (router agent)
2. Extract slots (NER agent)
3. Generate reply (writer agent)

That's three calls, three sets of failure modes, three things to keep in sync if you change the prompt. It's also more total tokens because you have to ship the conversation history into each call.

I picked a single structured-output call instead. The schema:

```ts
type StructuredReply = {
  reply: string;                  // the WhatsApp message to send back
  collected: {
    club:    string | null;
    date:    string | null;       // ISO YYYY-MM-DD
    time:    string | null;       // HH:MM 24h
    players: number | null;
  };
  intent:
    | "providing_info"
    | "confirming_yes"
    | "confirming_no"
    | "other";
};
```

One model call, one context window, one source of truth. The model sees the conversation history + the current "collected so far" + a hint about what stage the conversation is in, and returns all three outputs together.

### Why this works

- **Reply quality benefits from context the extractor also needs.** A reply like "Got it — Saturday at 8am for two players. Which club?" requires knowing both what's been collected and what's missing. Splitting that across two calls means sharing context twice.
- **Intent is cheap to attach.** Once the model is already reading the conversation, classifying "is this a yes?" is essentially free.
- **Structured outputs are deterministic.** The OpenAI structured-output API enforces the schema at the decode layer. The reply either parses or the call fails — there's no "the model wrapped my JSON in prose" failure mode to handle.

## The prompt

The system prompt does three things:

1. **Define the persona.** Friendly, concise, professional. Replies in whatever language the golfer used (Bahasa Melayu or English).
2. **Bound the universe.** Lists the registered golf clubs (fetched from Convex on every call) so the model doesn't hallucinate a club name.
3. **Give a date anchor.** "Today is 2026-05-27" so the model resolves "Saturday" correctly without web search.

The user prompt is the inbound text. The assistant turns in the few-shot history come from the conversation itself — the model has the same context the golfer has.

The function-tool argument carries the structured schema described above; we use OpenAI's `response_format: { type: "json_schema", … }` with `strict: true`.

## Intent routing in the action

Once we have `{ reply, collected, intent }`, the action picks a branch:

```ts
// Confirming a slot_available booking
if (awaitingConfirmation && intent === "confirming_yes" && activeBooking) {
  // Walk the booking through the chain:
  //   slot_available → awaiting_golfer_confirmation
  //                  → golfer_confirmed
  //                  → awaiting_payment
  // Each transition fires its own side-effect action. The final transition
  // (awaiting_payment) auto-sends bank details to the golfer — so we don't
  // send the structured reply here. The status-triggered message replaces it.
  ...
  return;
}

// Declining
if (awaitingConfirmation && intent === "confirming_no" && activeBooking) {
  transition(activeBooking, "cancelled");
  send(reply);
  return;
}

// All four fields collected + no active booking + conversation in a state
// where a new booking is legal:
if (allCollected && canStartFreshBooking) {
  createPending({ club, date, time, players });
  send(reply);
  return;
}

// Default: persist what we learned, send the reply.
updateStage(pickStage(currentStage, collected));
send(reply);
```

There is **no separate router model**. Intent routing is `switch (intent)` in TypeScript. The model classified; the code branches.

## Status-triggered messaging

The second AI flow is the opposite direction: a human flips a status in the dashboard, and the system needs to write the golfer a message *about that change*.

This is **not** a fresh LLM call. The messages here are deterministic templates because the surface area is small (6 status transitions trigger a message) and templates are more reliable than LLMs for "say this exact bank account number to the golfer." Example:

```ts
case "awaiting_payment": {
  const bank   = process.env.PAYMENT_BANK_NAME;
  const acct   = process.env.PAYMENT_BANK_ACCOUNT;
  const holder = process.env.PAYMENT_BANK_HOLDER;
  body = `Please make the booking payment to:\n${bank}\nA/C ${acct}\n${holder}\n\nSend back the transfer screenshot once done.`;
  break;
}
case "completed": {
  body = `Your booking is confirmed!\n\nClub: ${club}\nDate: ${date}\nTime: ${time}\nPlayers: ${players}\n\nEnjoy your round!`;
  break;
}
```

I could have made these LLM-generated for tone variety. I didn't, because:
- The data inside the message (bank account, dates, player count) must be *exactly* correct. Templates guarantee that.
- The operational cost of debugging "the AI dropped a digit from the bank account" is much higher than the upside of slightly varied phrasing.

The line: **AI generates conversational replies; templates send transactional facts.** Same principle as "your bank says transaction details deterministically and small-talks dynamically."

## Image handling (payment proofs)

A specifically interesting branch: when the inbound is `type: "image"` and the active booking is in `awaiting_payment`, we skip the LLM call entirely.

```ts
if (type === "image" && mediaId && activeBooking?.status === "awaiting_payment") {
  const bytes     = await downloadMedia(mediaId);
  const blob      = new Blob([bytes], { type: "image/png" });
  const storageId = await ctx.storage.store(blob);
  await setPaymentProof(activeBooking._id, storageId);
  await transitionStatus(activeBooking._id, "payment_received");
  await reply("Got it — I've passed your payment proof to our team. We'll confirm shortly.");
  return;
}
```

No model call. The fact "this image is payment proof for this booking" is implied by the booking's status. We just store it and move state forward.

If an image arrives outside `awaiting_payment` we fall back to a polite "thanks — what booking is this for?" reply. We don't try to OCR or auto-classify; the human verifies the image on the dashboard.

## Edge cases the design handles cleanly

| Scenario | What happens |
|---|---|
| Golfer sends 4 messages in 5 seconds before AI responds | Each is persisted in order; the LLM call for message N sees messages 1..N-1 in history. The model coalesces context naturally. |
| Golfer changes their mind mid-flow ("actually, make it 4 players") | `collected.players` overwrites; conversation stage stays the same. New booking record is *not* created — the existing pending booking can be edited by the operator. |
| Golfer says "yes" but no booking is in `slot_available` | `intent === "confirming_yes"` but `awaitingConfirmation === false`. We fall through to default branch and reply contextually — "Sorry, which booking are you confirming?" |
| Golfer goes silent for 24h | Convex cron sweeps for stale bookings and sends a single follow-up. `lastSilencePingAt` is set so we never double-ping. |
| Image arrives during a booking that isn't `awaiting_payment` | "Thanks — could you let me know which booking this is for?" — no storage, no LLM call. |
| Golfer asks something off-topic ("what's the weather?") | Model is instructed to gently redirect: "I can help you book a tee time — which club were you thinking?" No state mutation. |

## What I deliberately didn't build

- **No tool-calling / function-calling.** The model doesn't call DB functions. It returns structured data, and the *action* decides what to do. This keeps the model side stateless and easy to swap.
- **No multi-turn planning loop.** No "agent thinks, takes action, observes, thinks again." This isn't a research agent; it's a domain chatbot with a tight state machine. The LLM is one stateless function call inside a deterministic flow.
- **No vector memory.** Conversation history is bounded to the last 12 messages (enough for context, well under any context window). Long-term memory across bookings isn't a feature golfers want; it'd be a privacy surface for no upside.

## What I'd tune in production

- **Prompt evals.** A regression set of "the golfer said X — what should the model collect / classify?" and run it against every prompt change. Without this, you find out about prompt regressions when a paying user gets a bad reply.
- **Per-club guardrails.** Each club has its own pricing / cancellation policy. Today the model doesn't know that. Easy add: include club policy in the system prompt when a club is detected.
- **Multi-language QA.** I tested English and Bahasa Melayu. Production should have a small eval set per language.
