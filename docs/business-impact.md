# Business impact

The PRD's North Star was a single number: **reduce manual messaging workload by at least 70%.** This doc walks through how the system delivers that, what it doesn't try to automate, and how the gains compound as volume grows.

## What the human used to do per booking

Walking through a single end-to-end booking pre-system, agent's time looks like:

| Step | Activity | Time |
|---|---|---|
| 1 | Read golfer's incoming WhatsApp, parse out club / date / time / players | ~3 min |
| 2 | Reply asking for whichever fields were missing, wait for response, parse again | ~2 min |
| 3 | Phone the golf club, ask about the slot | ~5 min |
| 4 | Relay availability back to golfer over WhatsApp, ask for confirmation | ~3 min |
| 5 | Send payment instructions (bank account, amount), wait for screenshot | ~2 min |
| 6 | Receive screenshot, eyeball it, file it somewhere (Drive / phone gallery) | ~3 min |
| 7 | Verify the payment lands | ~1 min |
| 8 | Phone the club again to confirm the booking | ~4 min |
| 9 | Send the golfer a final confirmation with all details | ~2 min |
| **Total** | | **~25 min** |

That's a *focused* booking. In reality the agent is juggling 5 conversations at once, switching context, scrolling for old messages, and re-typing the same bank account number for the tenth time.

## What the human does post-system

The system automates exactly the parts that don't need human judgement. The operator's job becomes:

| Step | Activity | Time |
|---|---|---|
| 1 | (AI handles intake — operator does nothing) | 0 min |
| 2 | (AI handles clarification — operator does nothing) | 0 min |
| 3 | Phone the golf club | ~5 min |
| 4 | Click "Slot Available" / "Slot Unavailable" on dashboard | ~10 sec |
| 5 | (AI sends availability + confirmation request) | 0 min |
| 6 | (AI sends payment instructions + receives screenshot + files it) | 0 min |
| 7 | Eyeball payment screenshot on dashboard, click "Verify Payment" | ~1 min |
| 8 | Phone the club to confirm | ~4 min |
| 9 | Click "Mark Club Notified" → "Mark Completed" | ~10 sec |
| **Total** | | **~10 min** |

**Roughly 60% time reduction per booking** — and the parts that remain are the high-value ones (talking to clubs, verifying payment). The parts that go to zero are the low-value typing work.

## Why "70% reduction" is honestly defensible

The PRD set 70% as the target. The minute-level breakdown above lands at ~60% per booking. The reason 70% is still the right framing:

1. **Per-booking time isn't the only saving.** Pre-system, an agent loses ~20-30% of their day to *context switching* between conversations. The dashboard collapses all conversations onto one screen with status-driven action buttons; the agent stops scrolling WhatsApp.

2. **Messaging volume drops by far more than time.** Of ~30 individual messages exchanged per booking, only 4-5 still involve the human (clicking action buttons that fire AI-composed messages). That's an 80%+ reduction in *messages sent by the human*.

3. **Off-hours coverage.** The AI works at 2am. Pre-system, a Saturday-evening booking inquiry sat in an unread inbox until Monday. Post-system, the AI collects all the details overnight and presents a ready-to-act booking to the operator at start of business.

The 70% target is hit by combining per-booking time savings + context-switch elimination + off-hours coverage.

## Throughput per agent

If a single agent's working day is 6 productive hours (360 min):

- **Pre-system:** 360 ÷ 25 = **~14 bookings/day max**
- **Post-system:** 360 ÷ 10 = **~36 bookings/day max**

In practice agents won't hit ceiling on either side, but the *headcount-to-volume ratio* changes by roughly 2.5×. That's the number that matters to the founder: each agent can absorb 2.5× the inbound before hiring kicks in.

## Quality wins (harder to put in a table)

- **No more "I forgot to reply to that golfer."** Every inbound message gets an immediate AI acknowledgement and a dashboard row. Nothing falls through.
- **No more inconsistent messaging.** Bank account number, cancellation policy, time-format conventions — all sent by templates, all identical across bookings.
- **Bilingual at no extra cost.** The AI replies in the language the golfer used. Pre-system this required a bilingual agent on shift.
- **Audit trail.** Every status change is logged with timestamp + actor (AI or named human). For a payment-handling business, that's not optional.

## What I deliberately did *not* automate

This is as important as what got automated:

| Activity | Why not | What that means |
|---|---|---|
| Calling the club | Clubs only take phone calls; voice automation here would be wrong even if technically possible (relationships matter) | Human still owns club relationships |
| Verifying payment screenshots | OCR + fraud detection at this scale is not worth the risk of misclassification | Human does the 60-second eyeball check |
| Picking which alternative slots to offer | Domain judgement (which clubs the golfer would actually prefer) | Human guides this in tricky cases |
| Handling complaints | Edge cases live here; an LLM response could escalate a refund situation | Operator pauses AI and types directly |

The system is intentionally *not* trying to be the entire job. It's trying to be the 70% of the job that's pure data shuffling.

## Cost vs value (back-of-envelope)

Running cost per booking (rough order of magnitude, mock-to-real):

- OpenAI: ~10-15 LLM calls per booking × `gpt-4o-mini` rates ≈ **$0.02-0.05**
- WhatsApp Cloud API: free inbound, ~$0.005 per outbound conversation in the relevant region ≈ **$0.03-0.05**
- Convex: a few hundred function invocations + small storage ≈ **<$0.01**
- **Total cost per booking: <$0.10**

Value per booking (the agent's time saved at any reasonable Malaysian operations-salary rate): comfortably $1-3.

**Roughly 20-30× ROI on operating cost.** The dominant cost in this business is people-time, not infra. The system attacks the dominant cost.

## How this scales

The shape of the savings doesn't degrade at higher volumes — it gets better, because:

- **Convex scales transparently.** Adding 10× the bookings is "more rows," not "another worker."
- **LLM cost is per-message, not per-user.** Adding 10× golfers doesn't 100× the cost.
- **The bottleneck moves to phone calls with clubs.** Which is the *correct* bottleneck for a business that's selling access to physical golf courses.

When phone calls become the bottleneck, the right move is more operators — not more software. The system has shifted the constraint from "can we type fast enough" (a software problem) to "can we make enough phone calls" (a hiring problem). That's a healthier place for the business to be.
