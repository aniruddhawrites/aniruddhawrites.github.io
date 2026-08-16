---
layout: post
title: "The Post-Acquisition Assumption Gap"
seo_title: "Post-Acquisition Data Engineering: CDC, SCD and Hidden Architecture Assumptions"
subtitle: "Why a single Product Master change exposed the hidden assumptions behind CDC, SCD, intent capture, and enterprise governance."
cover-img: /assets/img/consumer-relative-scd-the-core-reframe.png
thumbnail-img: /assets/img/consumer-relative-scd-the-core-reframe.png
share-img: /assets/img/consumer-relative-scd-the-core-reframe.png
tags: [ DataEngineering, DataArchitecture, CDC, SCD, DataGovernance, DataProvenance, EnterpriseArchitecture, DataPlatforms, TechnicalDebt]
---
## The Post-Acquisition Assumption Gap

Two years after we closed an acquisition, we found out our own pipeline still thought the company hadn't happened.

Not the ERP. Not the org chart. The CDC job watching product category changes — built two years before the deal, quietly assuming there was exactly one PIM instance in the world, because at the time there was.

Nobody rearchitected that assumption when the acquisition closed, because "does our pipeline's cardinality assumption survive an acquisition" isn't a line item anyone owns.

Integration architecture disbands once systems are technically connected.

Data engineering inherits the pipeline, not the reason it was built the way it was. The gap between those two teams is where this incident actually lives.

And I want to say this clearly: nobody in this story made a mistake.

A category manager clicked save on a taxonomy cleanup that had been scheduled for weeks. That's the entire inciting event.

Everything expensive that happened afterward was already true before she logged in.

I'm going to move through the CDC-can't-capture-intent problem quickly, because if you've built distributed systems you already know this — row-level change capture answers "what changed," not "why."

No amount of tooling sophistication fixes that.

You either capture intent at the write path — through an outbox, a domain event, or something equivalent — or you don't have it.

What I want to spend real time on is what capturing intent actually costs you, because that part gets skipped in almost every version of this argument I've read.

---

## The Outbox Pattern Doesn't Solve Governance

Here's the part nobody puts in the diagram.

The outbox pattern doesn't eliminate the intent-capture problem.

It relocates it.

You move from:

> "We don't know why this changed."

to:

> "We have a `reason_code` column, and now someone has to govern what goes in it, forever."

That governance needs the same rigor you'd apply to a shared vocabulary in a multi-team API contract.

That's a harder job than it sounds like, because unlike a schema, a taxonomy of meaning has no compiler to enforce it.

Marketing adds `seasonal-refresh` because it unblocks their sprint.

Nobody tells compliance.

Six months later, compliance needs to isolate `regulatory-correction` events for an audit and discovers that the taxonomy has quietly drifted into unreliability.

The uncomfortable version of this claim is that most organizations adopting the outbox pattern to solve the CDC-intent problem are trading a technical debt they understand for a governance debt they don't have a process for.

Technical debt shows up in a code review.

Taxonomy drift shows up eighteen months later, in an audit, as a surprise.

I don't think this makes the outbox pattern wrong.

I think it means **"we added an outbox" is not the finish line people treat it as.**

If your architecture review signs off on intent capture without also assigning a human owner to the reason-code vocabulary, you've solved the half of the problem that was easier to solve.

---

## The Governance Debt Hidden Inside `reason_code`

The deeper problem is that intent is not static metadata.

Meaning evolves.

A reason code that made perfect sense when two organizations were separate can become ambiguous after an acquisition. A business process that was once local can become shared. A code that was descriptive for one team can become dangerously vague when another team starts consuming the same events.

That means the event contract has two dimensions:

1. **Technical contract** — what fields exist, what types they have, and how events are transported.
2. **Semantic contract** — what those fields actually mean, who owns that meaning, and how it changes.

Most architectures are very good at the first.

The second is where the assumption gap grows.

---

## The SCD Debate Is Often Asking the Wrong Question

I think most of the SCD debate — Type 1 versus Type 2, table-level versus attribute-level, how much history is enough — is arguing about the wrong variable.

The question isn't:

> "What SCD type does this attribute deserve?"

It's:

> **"Who is asking, and what do they need to be true about the past?"**

Finance, reconstructing last quarter's numbers, needs the category that was true on the transaction date.

That's Type 2: strict effective dating.

Merchandising, planning next season's assortment, may want the current hierarchy applied backward so that trend lines remain comparable to today's structure.

That's functionally Type 1, applied retroactively.

And it can be correct for that question even though the same behavior would destroy Finance's historical truth.

Compliance has a third requirement.

They may need to know not just what the value was, but who decided it and why.

Neither Type 1 nor Type 2 gives you that without an event-level record attached.

Three consumers.

Three different correct answers.

One physical attribute.

Most warehouses pick one SCD treatment per attribute and call it done, because building consumer-specific temporal views is expensive.

Then everyone is surprised when one consumer's "correct" implementation becomes another consumer's incident.

I don't think there's a clean fix that doesn't cost something.

Type 6 hybrids — carrying both historical and current values on every row — get you partway there for multiple consumers.

But they still don't solve compliance's need for decision provenance.

That requires event-level metadata that a dimension table was never designed to hold.

The honest answer may be less sophisticated than the architecture diagrams suggest:

> **You need to know which consumer you're building for before you pick the historical strategy.**

And most teams pick the strategy before anyone has asked the question.

---

## Consumer-Relative SCD — The Core Reframe

> I used to tell every team the same rule: never hard-delete a crosswalk row, always end-date it. I don't hold that as a blanket rule anymore.

End-dating solves a real problem.

It lets you distinguish:

> "This relationship expired."

from:

> "This relationship never existed."

A hard delete destroys that distinction completely.

That part I still believe.

What I left out for years is that if the relationship touches anything adjacent to personal data, even indirectly, keeping it around forever in end-dated form is still retention.

And some data-minimization regimes have affirmative deletion requirements that end-dating doesn't satisfy, no matter how well-intentioned it is.

"Never delete" and "comply with deletion law" are not always compatible instructions.

Most SCD advice, mine included, has quietly assumed that more history is an unqualified good.

It isn't.

It's a trade-off with a legal dimension I used to treat as someone else's problem.

---

## Hard Delete vs. End-Date — The Privacy Tension

Look at your own `dim_product` right now.

Pick any Type 2 attribute.

Can you tell, from the table alone — no tribal knowledge, no Slack archaeology — which historical rows represent a business decision and which represent someone fixing a typo?

If the answer is no, you don't have an audit trail.

You have a very expensive list of things that happened, with the one piece of information that would make it useful — **why** — living somewhere else, if it's living anywhere at all.

That's the distinction between historical state and decision provenance.

A dimension can tell you that something changed.

It cannot necessarily tell you whether that change was:

- a deliberate business decision,
- a regulatory correction,
- a data-quality fix,
- an integration artifact,
- or an accidental edit.

If those distinctions matter to a downstream consumer, they need to exist somewhere deliberately.

They shouldn't have to be reconstructed from the shape of the data years later.

---

## The Bridge-Table Wager

Call it the bridge-table wager.

Every team building a crosswalk between two systems is making an implicit bet about how many independent dimensions of change that relationship will need over its lifetime.

Bet low — one dimension, effective dating alone — and a Kimball bridge table is simpler, cheaper, and analyst-friendly.

Bet high — effective dating plus provenance plus, eventually, some kind of confidence scoring you haven't thought of yet — and a Data Vault Link with its own Satellite ages better, because it doesn't force every new dimension of change into the same row-versioning event.

Nobody knows which bet is right at design time.

That's the actual disagreement.

And I don't think it resolves with more expertise.

It resolves with hindsight, which is exactly the thing architecture decisions never get to have.

I've watched two architects I equally respect land on opposite sides of this exact wager for reasons that were both completely defensible given what they knew when they made the call.

The lesson isn't that one modelling approach is universally superior.

The lesson is that **the modelling choice is itself an assumption about the future.**

That assumption deserves to be visible.

---

## Provenance You Can't Verify

One thing worth keeping at full strength is what happens when provenance is added after the fact.

When you retrofit a `source_system` column onto a crosswalk table that's been running for five years across three different loaders, the rows written before that column existed don't get real provenance.

They get your best guess, formatted to look like data.

You can infer which loader probably wrote a row from column-population patterns or ID formatting.

But you can't verify it.

A compliance query built on that inference is answering a question with a guess wearing a structured column's clothing.

I don't think this is a solvable problem after the fact.

I think it's a reason to instrument provenance on day one, even when it feels like premature process.

Because there is no retroactive fix that isn't fiction.

---

## What the Incident Actually Taught Us

The incident looked like a CDC failure.

It wasn't.

CDC was doing exactly what it had been designed to do.

The deeper failure was that the architecture carried assumptions nobody had made explicit:

- there was one PIM instance;
- product identity was stable across systems;
- a change was enough information without its intent;
- one SCD strategy could satisfy every consumer;
- historical retention was always preferable to deletion;
- provenance could be reconstructed later if anyone needed it.

None of those assumptions were unreasonable when they were made.

That's the dangerous part.

Architecture debt rarely begins with a bad decision.

It begins with a **reasonable decision whose original context disappears**.

An acquisition accelerates that process because it introduces new systems, new consumers, new governance requirements and new interpretations of what the data means — while the old pipelines continue operating as if none of that happened.

The systems integrate.

The assumptions don't.

---

## The Architecture Questions I Would Ask Now

If I were reviewing the architecture today, I wouldn't start by asking which CDC tool we're using or whether the warehouse should use Type 1, Type 2, Type 6 or Data Vault.

I'd ask:

1. **What business intent do we need to preserve when this record changes?**
2. **Who owns the vocabulary used to describe that intent?**
3. **Which consumers need historical truth, and which need current truth?**
4. **Which consumers need decision provenance rather than simply state history?**
5. **What data must eventually be deleted rather than merely end-dated?**
6. **Can we verify the provenance of every historical relationship, or are some rows already inference?**
7. **What assumptions about identity, cardinality and ownership would break if another company or system joined this architecture tomorrow?**

Those questions cost less to answer during design than they do during an incident.

Not because they prevent every failure.

They don't.

They make the assumptions visible before someone discovers them at production scale.

---

## The Larger Lesson

The hardest part of post-acquisition data integration isn't connecting the systems.

That's the part organizations are usually good at measuring:

- interfaces are live;
- pipelines are green;
- records are flowing;
- dashboards are populated.

The harder question is whether the combined architecture still means what its individual systems originally meant.

That's where the assumption gap appears.

A pipeline can be technically correct and architecturally wrong.

A dimension can preserve history and still fail to preserve provenance.

An outbox can capture intent and still create governance debt.

A deletion strategy can preserve auditability and still violate a retention requirement.

And a design that was completely reasonable two years ago can become the wrong design without anyone making a new mistake.

**That, to me, is the real post-acquisition problem.**

Not integration.

**Context loss.**

---

## Related Reading

If you're interested in how hidden assumptions surface elsewhere in enterprise data platforms, I explored a related problem in:

- **[When Data Becomes the Bottleneck](/2026-05-13-when-data-becomes-bottleneck-unmasking-real-culprit-behind/)** — why apparent data-platform and SLA failures can actually emerge from contention, latency and interactions across the architecture.

The specific failure mode is different.

The architectural pattern is not:

> **The system rarely fails because of the assumption everyone can see. It fails because of the assumption nobody realised they were still making.**

---

## Author's Note

This article is a practitioner-led reflection on enterprise data architecture, based on patterns and failure modes I have encountered or studied across complex data environments.

The scenarios are intentionally presented as architectural narratives rather than as descriptions of a specific client, company, or production incident. The purpose is to examine the reasoning behind design decisions — particularly around CDC, historical modelling, provenance, governance, and post-acquisition integration.

The opinions and architectural trade-offs presented here are my own and should not be interpreted as prescriptive guidance for every environment.

---

## About the Author

**Aniruddha Banerjee** is a data and technology architect working across enterprise data platforms, analytics, cloud architecture, performance engineering, and AI-enabled systems.

Through **Aniruddha Writes**, he explores the engineering decisions behind complex data platforms — particularly the assumptions, trade-offs, and failure patterns that are often invisible until systems reach enterprise scale.