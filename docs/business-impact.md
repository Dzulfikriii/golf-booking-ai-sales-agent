# Business impact

A few notes on what the system changes about the operator's day, and why the math works the way it does.

## What the operator used to spend time on

Walk through a single booking pre-system and the agent's time is mostly in messaging work: reading the inbound, parsing out club / date / time / players, asking follow-up questions, waiting for replies, relaying availability back, sending bank details, eyeballing a payment screenshot, then writing a final confirmation. The actually skilled parts (phoning the club, verifying the payment lands) are short. The typing is long.

Each booking has on the order of twenty to thirty individual messages exchanged. Most of them are mechanical: clarifying details, sending the same bank account number, formatting the same confirmation. Doing this across five concurrent conversations means a lot of context-switching, and inevitably some messages slip through unanswered.

## What changes

The system absorbs the mechanical messaging. The operator's job becomes:

- Take a phone call to the golf club.
- Click one button on the dashboard ("Slot Available" or "Slot Unavailable").
- Eyeball a payment screenshot when one arrives, click "Verify Payment".
- Phone the club again to confirm.
- Click "Mark Club Notified" and then "Mark Completed".

That's it. The AI handles the rest: parsing intent, asking for missing fields, sending availability updates, sending bank details, ingesting the payment screenshot, sending the final booking confirmation.

In rough terms, the typing time per booking drops from around twenty to twenty-five minutes to under ten. The parts that stay are the parts that need a human: actual conversations with clubs, the judgement call on a payment screenshot.

## Why the gain compounds beyond per-booking time

A per-booking time saving isn't the only thing that changes.

Context-switching collapses. The dashboard puts every active booking on one screen with status-driven action buttons; the operator stops scrolling WhatsApp threads looking for the message they were halfway through.

Off-hours coverage becomes real. A Saturday-evening inquiry doesn't sit in an unread inbox until Monday. The AI collects all the details overnight and presents the operator with a ready-to-act booking at the start of business.

Consistency improves. Bank account numbers, cancellation phrasing, time-of-day formatting are identical across bookings. The audit log captures every status change with timestamp and actor, which matters for any business handling payments.

The throughput-per-operator number ends up somewhere around two and a half to three times higher than pre-system. The exact multiplier depends on how much of the typical operator's day was previously lost to context-switching, which varies.

## What the system intentionally doesn't automate

Equally important to what's automated:

- Calling the club. Clubs only take phone calls, and the relationship between the booking agent and the club matters. Voice automation would be wrong here even if it were technically possible.
- Verifying payment screenshots. A 60-second eyeball check is fast, and the failure mode of misclassification (sending the golfer a "confirmed" message when the payment didn't land) is bad enough that automating it isn't worth the upside.
- Picking which alternative slots to offer when the first attempt fails. This needs domain judgement about which clubs a particular golfer would actually prefer.

The system isn't trying to be the entire job. It's trying to be the part of the job that's data-shuffling.

## Operating cost

The operating cost per booking is small. OpenAI usage is on the order of a few cents per booking with `gpt-4o-mini`. WhatsApp outbound is a few cents in the relevant region. Convex usage at this volume is rounding noise. The dominant operational cost in the business is people-time, which is the cost the system actually attacks.

## Where the bottleneck moves

After this system is in place, the limiting factor on booking throughput is no longer typing speed. It's the time it takes to make phone calls to clubs. That's a healthier bottleneck for the business: it shifts the scaling question from "do we have enough software" to "do we have enough phone-call hands". The right response to the second one is hiring, not engineering.
