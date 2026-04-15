---
title: "The Stripe System Design Question That Separates Senior From Staff Engineers | by The Speedcraft Lab"
source: "https://freedium-mirror.cfd/https://medium.com/codetodeploy/the-stripe-system-design-question-that-separates-senior-from-staff-engineers-b39f1f1a05cf"
author:
published:
created: 2026-04-15
description: "Most candidates answer the wrong question entirely. Here is why the stakes are higher than they..."
tags:
  - "clippings"
---
[< Go to the original](https://medium.com/@speedcraft21/the-stripe-system-design-question-that-separates-senior-from-staff-engineers-b39f1f1a05cf#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/1*ogT0CKa2SevfepZf6j1gwA.png)

## The Stripe System Design Question That Separates Senior From Staff Engineers

## Most candidates answer the wrong question entirely. Here is why the stakes are higher than they appear.

androidstudio · March 23, 2026 (Updated: March 23, 2026) · Free: No

Stripe processes hundreds of millions of payment requests every year. Each one touches a bank, a card network, a merchant account, and a chain of internal services. Now picture this: a network blip causes your client to retry a charge. Did the first attempt go through? You genuinely do not know. The customer might be charged twice. The merchant might fulfil the same order twice. The bank might flag the account for suspicious activity.

How do you guarantee that a payment executes exactly once in a system where nothing is guaranteed?

That is the question. And the way you answer it tells Stripe a great deal about where you are in your career.

### The Deceptively Simple Question

The problem sounds almost trivial when you first hear it. A user clicks "Pay." You call the payments API. You store the result. Done.

But the interviewer is not asking about the happy path. They are asking what happens in the gap between sending a request and confirming its result. That gap is where distributed systems get genuinely interesting.

Most candidates hear "payment system" and immediately jump to database schemas, microservices boundaries, or API rate limits. All of those matter. None of them are the core of this question.

The core is this: in a distributed system, you do not get exactly once execution for free. You get at most once, which risks losing data, or at least once, which risks duplicating side effects. Exactly once is a property you have to build yourself, at the application layer. The question is testing whether you know that.

At any meaningful scale, this breaks because a retry is completely indistinguishable from a new request unless you build a mechanism that makes them distinguishable.

### The Senior Answer

The correct response at the senior level is idempotency keys. The client generates a unique identifier for each intended operation and sends it alongside every request. The server stores that key and its result. If the same key arrives again, the server returns the stored result without re executing the underlying logic.

In plain terms: you are trading storage for safety. Every intended operation gets a fingerprint. The fingerprint survives the retry.

Stripe's public documentation describes exactly this pattern. Clients pass an `Idempotency-Key` header. The server uses it to deduplicate requests. This is the right answer and it earns full credit at the senior level.

It is clean, it is well understood, and it solves the duplicate charge problem in the straightforward case. A senior engineer should be able to explain it clearly, describe how to store and expire idempotency records, and walk through the happy path end to end.

But here is where most people stop. And stopping here is what separates a senior answer from a staff one.

### The Staff Answer

The staff level follow up is: what happens between receiving the idempotency key and writing the result?

Walk through it carefully. Your server receives a request with a fresh idempotency key. It charges the card. It prepares to write "success" to the idempotency store. Then the server crashes. The response never reaches the client. The client retries with the same key. No result exists in the store. The server charges the card again.

You have recreated the original problem inside your solution.

The fix is to treat the idempotency record as a state machine rather than a simple key value pair. Before executing the operation, write a record in an "in progress" state. After completing it, transition to "completed" and store the result. On any retry, if the state is "in progress," return a conflict response and ask the client to wait. If the state is "completed," return the cached result. If no record exists, begin fresh.

This is the approach most teams converge on at this scale, based on publicly available engineering literature on payment system design. It turns a deceptively simple cache into a coordination mechanism.

If two retries arrive simultaneously before either has finished, you need a compare and swap operation or a distributed lock on the idempotency record. Only one execution can hold the "in progress" state at a time. This adds latency, which is a real cost, and it means your idempotency store becomes a critical path dependency whose availability directly affects your payment success rate.

That trade off, latency and availability for correctness, is what the question is really probing.

### The Hidden Dimension

Most engineers feel confident once they reach the state machine answer. Here is the harder question underneath it.

What happens when your payment service calls three downstream services and one of them fails halfway through? You have already notified the bank. You have not updated the merchant's inventory. You have not sent the confirmation email. Your idempotency key covers the top level request. It does not automatically coordinate the state of every service you called on the way down.

This is the boundary between idempotency and transaction management across services. And it is where the saga pattern lives.

A saga breaks a distributed transaction into a sequence of local transactions, each with a corresponding compensating action that can undo it if a later step fails. It does not give you atomicity. It gives you eventual consistency and a defined recovery path.

The trade off is visibility and complexity. Sagas are harder to reason about than a single database transaction. But a single database transaction across five services is not actually an option at this scale because it requires a two phase commit, which is slow, fragile, and does not compose well under failures.

Strong candidates reach idempotency keys and state machines. Exceptional candidates recognise that the harder problem is coordinating state across service boundaries, and they can name at least one pattern that addresses it.

### What Stripe Actually Cares About

Stripe's engineering culture, based on its public writing and talks, is built around correctness, reliability, and the ability to reason clearly about failure. The system design question is a proxy for all three.

What the interviewer is really measuring is not whether you know the word "idempotency." Most engineers with a few years of distributed systems experience do. They are measuring whether you think in terms of failure surfaces. Can you identify where your solution breaks? Can you explain the trade off of your fix? Can you see one level deeper than the obvious answer?

The reusable mental model is this. If your operation has no side effects, idempotency comes for free. If it has side effects, you need idempotency keys plus a state machine. If it has side effects and calls other services, you also need a distributed coordination strategy, whether that is a saga, an outbox pattern, or explicit compensating transactions.

The sentence you can use in any system design discussion: "Exactly once semantics in a distributed system require idempotency at the application layer, not at the transport layer."

Stripe processes payments at a scale where a single missing guarantee, idempotency, turns a network blip into a customer support crisis. The system design question is not really about Stripe. It is about how clearly you can think under conditions where nothing is guaranteed and everything has a cost.

The next hard problem in this space is cross service idempotency at the saga level: how do you make compensating transactions themselves idempotent, so that recovery is safe to retry? That question deserves its own deep treatment, and it is where the most interesting distributed systems work is happening right now.

If this topic interests you, the related piece on the saga pattern goes deeper on how teams handle distributed state management without two phase commit, including the specific failure modes that most saga implementations get quietly wrong. I am covering more system design patterns like this, follow so you do not miss the next one. If you want these breakdowns before they go public, the email list is one click away.

**Follow me for more such content.**

[#system-design-interview](https://medium.com/tag/system-design-interview "System Design Interview") [#distributed-systems](https://medium.com/tag/distributed-systems "Distributed Systems") [#idempotency](https://medium.com/tag/idempotency "Idempotency") [#stripe](https://medium.com/tag/stripe "Stripe") [#staff-engineering](https://medium.com/tag/staff-engineering "Staff Engineering")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)