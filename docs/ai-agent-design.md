# AI agent design

The agent runs as one structured-output OpenAI call per inbound message. The same call covers slot extraction, intent classification, and reply generation.

## The structured response

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

OpenAI's structured outputs API enforces the schema at decode time, so the response always parses cleanly. The action that calls the model uses each field directly without any post-processing.

A pipeline of separate calls (intent classifier, slot extractor, reply writer) would be more total tokens because the conversation history would have to ride along each step. It would also create three prompts to keep in sync. One call sharing one context window is simpler and produces better replies, because the reply benefits from the same conversation state the extractor needs.

## The prompt

The system prompt does a few things:

- Sets persona: friendly, concise, professional. Replies in whatever language the golfer used (English or Bahasa Melayu).
- Lists the active golf clubs from the database, so the model doesn't hallucinate a club name.
- Anchors the date with today's value, so the model resolves relative references like "Saturday" without web search.

The user message is the inbound text. The recent conversation goes in as prior assistant / user turns; history is bounded to the last twelve messages.

## Intent routing in the action

After the model returns, the action picks a branch based on the `intent` field and the current booking state.

```ts
// Golfer says yes to a slot-available confirmation:
// walk the booking through slot_available → awaiting_golfer_confirmation →
// golfer_confirmed → awaiting_payment. The awaiting_payment transition
// auto-sends bank details, so we don't send the structured reply here.
if (awaitingConfirmation && intent === "confirming_yes" && activeBooking) {
  ...chain transitions...
  return;
}

// Golfer says no:
if (awaitingConfirmation && intent === "confirming_no" && activeBooking) {
  transition(activeBooking, "cancelled");
  send(reply);
  return;
}

// All four fields collected and no active booking yet:
if (allCollected && canStartFreshBooking) {
  createPending({ club, date, time, players });
  send(reply);
  return;
}

// Default: persist what we learned, send the reply.
updateStage(pickStage(currentStage, collected));
send(reply);
```

The model classifies; the action branches. There is no separate router model.

## Status-triggered messaging

The opposite direction (operator flips a status, system writes the golfer a message) is not an LLM call. The outbound messages here carry transactional data where exactness matters more than tone variety. Example:

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

A misread bank account number is a much worse failure than slightly stiff phrasing. The rule of thumb is conversational replies are AI-generated; transactional messages are templated.

## Payment-proof images

When the inbound message is an image and the active booking is in `awaiting_payment`, the LLM call is skipped entirely.

```ts
if (type === "image" && mediaId && activeBooking?.status === "awaiting_payment") {
  const bytes     = await downloadMedia(mediaId);
  const blob      = new Blob([bytes], { type: "image/png" });
  const storageId = await ctx.storage.store(blob);
  await setPaymentProof(activeBooking._id, storageId);
  await transitionStatus(activeBooking._id, "payment_received");
  await reply("Got it. I've passed your payment proof to our team. We'll confirm shortly.");
  return;
}
```

The fact that "this image is payment proof for this booking" is implied by the booking's status. No classification needed.

If an image arrives outside `awaiting_payment`, the system falls back to a "thanks, which booking is this for?" reply without storing the file or making an LLM call.

## Edge cases the design handles cleanly

- Multiple inbound messages arriving in quick succession are persisted in order; the LLM call for message N sees messages 1..N-1 in history and coalesces them naturally.
- A golfer changing their mind mid-flow ("actually, make it 4 players") overwrites the collected field without creating a new booking; the operator can edit the existing pending row.
- A "yes" arriving when no booking is awaiting confirmation falls through to the default branch, where the model writes a contextual "which booking are you confirming?" reply.
- Silence for 24h on a booking awaiting payment or confirmation triggers a single follow-up via a cron, with `lastSilencePingAt` set so the system never double-pings.
- Off-topic messages ("what's the weather?") are gently redirected by the model toward the booking flow, with no state mutation.
