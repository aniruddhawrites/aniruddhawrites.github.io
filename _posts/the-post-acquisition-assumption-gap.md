---
layout: post
title: The Post-Acquisition Assumption Gap
subtitle: "Why a single Product Master change exposed the hidden assumptions behind CDC, SCD, intent capture, and enterprise governance."
cover-img: /assets/img/when-data-becomes-bottleneck-unmasking-real-culprit-behind.png
thumbnail-img: /assets/img/when-data-becomes-bottleneck-unmasking-real-culprit-behind.png
share-img: /assets/img/when-data-becomes-bottleneck-unmasking-real-culprit-behind.png
categories: [Data Engineering, Amazon Redshift, Data Architecture, Performance Engineering]
tags: [DataEngineering, MicrosoftFabric, PowerBI, ServiceLevels, DataPlatform, CloudArchitecture, Latency, PerformanceEngineering, Analytics, DataContention, TechLeadership]
author: Aniruddha Banerjee
reading_time: "12-15 min"
---
## The Post-Acquisition Assumption Gap

Two years after we closed an acquisition, we found out our own pipeline still thought the company hadn't happened.

Not the ERP. Not the org chart. The CDC job watching product category changes — built two years before the deal, quietly assuming there was exactly one PIM instance in the world, because at the time there was. Nobody rearchitected that assumption when the acquisition closed, because "does our pipeline's cardinality assumption survive an acquisition" isn't a line item anyone owns. Integration architecture disbands once systems are technically connected. Data engineering inherits the pipeline, not the reason it was built the way it was. The gap between those two teams is where this incident actually lives, and I want to say clearly: nobody in this story made a mistake. A category manager clicked save on a taxonomy cleanup that had been scheduled for weeks. That's the entire inciting event. Everything expensive that happened afterward was already true before she logged in.

I'm going to move through the CDC-can't-capture-intent problem quickly, because if you've built distributed systems you already know this — row-level change capture answers "what changed," not "why," and no amount of tooling sophistication fixes that; you either capture intent at the write path (an outbox, a domain event) or you don't have it. What I want to spend real time on is what capturing intent actually costs you, because that part gets skipped in almost every version of this argument I've read.

* * *

## The Outbox Pattern — Mechanics and Its Gap

Here's the part nobody puts in the diagram. The outbox pattern doesn't eliminate the intent-capture problem. It relocates it — from "we don't know why this changed" to "we have a `reason_code` column, and now someone has to govern what goes in it, forever, with the same rigor you'd apply to a shared vocabulary in a multi-team API contract." That's a harder job than it sounds like, because unlike a schema, a taxonomy of *meaning* has no compiler to enforce it. Marketing adds `'seasonal-refresh'` because it unblocks their sprint. Nobody tells compliance, who six months later need to isolate `'regulatory-correction'` events for an audit and discover the taxonomy has quietly drifted into unreliability.

The uncomfortable version of this claim: **most organizations that adopt the outbox pattern to solve the CDC-intent problem end up trading a technical debt they understood for a governance debt they don't have a process for at all.** Technical debt shows up in a code review. Taxonomy drift shows up eighteen months later, in an audit, as a surprise. I don't think this makes the pattern wrong. I think it means "we added an outbox" is not the finish line people treat it as, and if your architecture review signs off on intent-capture without also assigning a human owner to the reason-code vocabulary, you've solved the half of the problem that was easier to solve.

* * *

## Reason-Code Governance Debt Over Time

I think most of the SCD debate — Type 1 versus Type 2, table-level versus attribute-level, how much history is enough — is arguing about the wrong variable. The question isn't "what SCD type does this attribute deserve." It's "who is asking, and what does *they* need to be true about the past."

Finance, reconstructing last quarter's numbers, needs the category that was true on the transaction date — Type 2, strict effective dating. Merchandising, planning next season's assortment, wants the current hierarchy applied backward so trend lines are comparable to today's structure — which is functionally Type 1, applied retroactively, and is *correct* for their question even though it's the same "wrong" behavior that blew up finance's quarter. Compliance, defending an audit, needs to know not just what the value was, but who decided it and why — which neither Type 1 nor Type 2 gives you without an outbox event attached.

Three consumers. Three different correct answers. One physical attribute. Most warehouses pick one SCD treatment per attribute and call it done, because building consumer-specific temporal views is expensive — and then act surprised when one consumer's "correct" implementation is another consumer's incident. I don't think there's a clean fix for this that doesn't cost something. Type 6 hybrids (carry both the historical and current value on every row) get you partway there for two consumers at once, but they don't solve for compliance's need for decision-provenance, which requires the event-level metadata a bare dimension table was never built to hold. I'm not sure the honest answer is more sophisticated than "you need to know which consumer you're building for before you pick a strategy, and most teams pick the strategy before anyone's asked the question."

* * *

## Consumer-Relative SCD — The Core Reframe

> I used to tell every team the same rule: never hard-delete a crosswalk row, always end-date it. I don't hold that as a blanket rule anymore.
> 
> End-dating solves a real problem — it lets you tell "this relationship expired" apart from "this relationship never existed," which a hard delete destroys completely. That part I still believe.
> 
> What I left out for years: if that relationship touches anything adjacent to personal data, even indirectly, keeping it around forever in end-dated form is still retention — and some data minimization regimes have an affirmative deletion requirement that end-dating doesn't satisfy no matter how well-intentioned it is. "Never delete" and "comply with deletion law" are not always compatible instructions, and most SCD advice, mine included, has quietly assumed more history is an unqualified good. It isn't. It's a trade-off with a legal dimension I used to treat as someone else's problem.

* * *

## Hard Delete vs. End-Date — The Privacy Tension

Instead of presenting the typo/decision rows as "here's proof of the problem," frame it as a direct provocation, which lands harder and is more shareable as a standalone image:

> Look at your own `dim_product` right now. Pick any Type 2 attribute. Can you tell, from the table alone — no tribal knowledge, no Slack archaeology — which of its historical rows represent a business decision and which represent someone fixing a typo?
> 
> If the answer is no, you don't have an audit trail. You have a very expensive list of things that happened, with the one piece of information that would make it useful — *why* — living somewhere else, if it's living anywhere at all.

(Table follows as before, now serving as evidence for a claim already made, not the claim itself.)

* * *

## The Bridge-Table Wager

Call it **the bridge-table wager.** Every team building a crosswalk between two systems is making an implicit bet about how many independent dimensions of change that relationship will need over its lifetime. Bet low — one dimension, effective dating alone — and a Kimball bridge table is simpler, cheaper, and analyst-friendly. Bet high — effective dating plus provenance plus, eventually, some kind of confidence scoring you haven't thought of yet — and a Data Vault Link with its own Satellite ages better, because it doesn't force every new dimension of change into the same row-versioning event.

Nobody knows which bet is right at design time. That's the actual disagreement, and I don't think it resolves with more expertise — it resolves with hindsight, which is exactly the thing architecture decisions never get to have. I've watched two architects I equally respect land on opposite sides of this exact wager for reasons that were both completely defensible given what they knew when they made the call.

## Provenance You Can't Verify

One thing worth keeping at full strength rather than trimming: when you retrofit a `source_system` column onto a crosswalk table that's been running for five years across three different loaders, the rows written before that column existed don't get real provenance. They get your best guess, formatted to look like data. You can infer which loader probably wrote a row from column population patterns or ID formatting, but you can't verify it, and a compliance query built on that inference is answering a question with a guess wearing a structured column's clothing. I don't think this is a solvable problem after the fact. I think it's a reason to instrument provenance on day one even when it feels like premature process, because there is no retroactive fix that isn't fiction