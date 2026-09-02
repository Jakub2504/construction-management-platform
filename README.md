# Construction Management Platform — case study

A CRM, quoting and invoicing system I designed, built and maintain for a small construction
company (my family's business). It runs in production and is used day to day to manage clients,
quotes, invoices and jobs — including an assistant that drafts quote line items from a plain
description of the work, and from the project's floor plan.

> **This repository is documentation only.** The source code and the database stay private —
> the system holds real client and financial data. This page describes what I built, the
> constraints I built it under, and why I made the decisions I made.

`Node.js` · `Express` · `libSQL / Turso` · `Vanilla JS (ES modules)` · `Gemini` · `Computer vision (custom, in JS)` · `JWT + bcrypt` · `Zod` · `Helmet` · `Render`

---

## The problem

Before this system, all of it was done by hand. My father would sketch out a quote on paper,
I would type it up in Word, and the invoices were built separately in Excel — every line, every
total, every number entered manually, twice.

It worked, but it was fragile. Totals were easy to get wrong, nothing carried over from a quote
to its invoice without being retyped, prices had to be looked up from scratch every time, and
there was no single place to see which quotes were still open or which invoices had actually
been paid.

The constraints were as real as the problem:

- The people using it are not technical.
- It has to be reachable from anywhere — the office, a laptop, a phone on site.
- It holds client and financial data and it is reachable from the internet, so access control
  has to be real rather than cosmetic.
- It has to stay maintainable by one person, working on it occasionally.

## What it does

- **Clients (CRM)** — client records and the work associated with each one.
- **Quotes** — line-item editor with automatic subtotals, VAT and totals, and correlative numbering.
- **Invoices** — generated from an existing quote, numbered correlatively per year, with payment status.
- **Jobs** — construction work linked to clients.
- **Quote drafting from a description** — the user describes the job in plain language; the
  system returns draft line items with quantities, units and prices to review.
- **Quote drafting from a floor plan** — upload a plan (image or PDF), the system detects walls,
  identifies rooms and areas, and feeds that into the same drafting flow.
- **Pipeline & dashboard** — the current state of the business at a glance.
- **Printable documents** — bilingual (Spanish / Catalan) quotes and invoices, produced from the browser.

## Architecture

```text
Browser  (office PC · laptop · phone — from anywhere)
   │   httpOnly JWT cookie over HTTPS
   ▼
Express API  —  deployed on Render
   │   Helmet + CSP · per-route rate limits · Zod validation · RBAC on every endpoint
   │
   ├── auth · clients · quotes · invoices · jobs · dashboard · company
   │
   └── /api/ia ── quote drafting  ──┬── company's own quote history
       (Gemini)   plan reading      ├── market price reference
                                    ├── verification layer
                                    └── margin applied in code
   ▼
Turso  (hosted libSQL — SQLite-compatible)
```

Wall detection runs **in the browser** as pure computer vision; the plan image is also sent to
the vision model for room labels and areas. Documents are rendered with a print stylesheet, so
the browser produces the PDF and the system needs no PDF library.

## Where the AI is, and where it deliberately isn't

This is the part of the project I would most want to talk through.

**The model never decides a price on its own.** Prices resolve through a fixed precedence: the
company's own historical line items first (their real prices, used as-is), then a market price
reference, and only then a model estimate, flagged as low confidence. Every generated line
reports which of the three it came from, so the user knows what to check.

**A verification layer sits between the model and the user.** Generated lines are checked back
against the company's own history: when the model estimates a price for work that has actually
been quoted before, the real price replaces the estimate. That single substitution measurably
raised the share of each quote backed by real data rather than inference.

**Margin is applied in code, not by the model.** Published reference prices are execution cost
without contractor margin, so a margin has to be added — by a configurable factor in the
service, while the prompt explicitly forbids the model from adding one. If both applied it,
every quote would come out inflated twice over.

**Rules exist because the model got things wrong in specific, repeatable ways.** When the
description says the client supplies the material, only labour is priced — without that rule the
model priced material nobody was buying and roughly doubled the figure. Others came from the same
route: an early pricing heuristic was measurably overvaluing jobs that bundle several tasks into
one visit, and had to be replaced once the error was quantified rather than guessed at.

**The accuracy is stated honestly, not flattered.** Measured against real quotes that were not
used for calibration, the drafts came out +35%, +22% and −40%. Realistic precision is ±20–35% —
and that figure is the *floor*, measured on jobs with no comparable history to draw on. The error
leans high on ordinary work, which is the safer direction to be wrong in, and the −40% case was a
quote deliberately priced aggressively; at the aggressive setting it lands within 4%. The output
is a draft for a human to adjust, not an autonomous pricer, and it is designed that way.

**I chose not to chase a better number.** Tuning the prompt brought one quote to within 0.6% and
broke the other two, to +59% and +97%. With three examples that is overfitting, so I stopped:
tightening it properly needs 10 to 20 real quotes, not prompt edits. The mechanism that actually
improves it is the history — every saved quote adds real prices to the top precedence level, so
the external reference is a starting crutch that becomes irrelevant on its own.

**Two negative results worth recording.** The original design had the model search the web for
market prices when there was no history; it returns HTTP 429 on the free tier, because search
grounding is a billed feature — so the lookup was done once, offline, and seeded. And semantic
retrieval over the quote history, using embeddings to pull the most similar past line items, is
implemented but switched off: in this domain the similarity ranking behaved badly and could leave
the correct line item out entirely. Retrieval stays deterministic until something can be shown to
beat it.

**For floor plans: computer vision for geometry, the model for meaning.** Walls are found with
classical CV I wrote from scratch in JavaScript — threshold, a morphological opening to strip
text and dimension lines, a closing to bridge dashed partitions, then connected-component
denoising — verified pixel-identical against a Python reference implementation. Room labels and
areas come from the vision model. The wall elements the model returns are discarded on purpose:
a wall is a thin, long line, and a vision model gives an approximate box that lands in the
middle of the room.

**And where the model could not be trusted at all, I took it out of the path.** Reading the total
floor area off a scanned plan is not reliable — not with a vision model, not with OCR — and
having to verify every number defeats the point of automating it. So the system does not read the
area: the user drags a line over one known dimension to calibrate the scale, and the floor area
is computed geometrically from the detected footprint.

## Engineering decisions

**No framework and no build step.** The frontend is native ES modules and plain CSS. Nothing to
compile, no toolchain to maintain. For a system that has to keep working for years with
occasional attention from one person, a dependency-light stack is a feature.

**SQLite semantics, managed persistence.** The data model is SQLite-compatible, which keeps
queries and migrations easy to reason about, but the database is hosted (Turso) rather than a
file next to the app — so the data is not tied to the container's lifetime and there is no
database server to operate. Migrating there without rewriting every query meant writing a small
async wrapper that preserves the previous driver's call signature.

**Security that has to actually hold.** Every API route sits behind authentication. Passwords are
hashed with bcrypt, sessions are JWTs in httpOnly cookies, request bodies are validated with Zod,
and Helmet applies an explicit CSP. Authorisation is role-based and enforced per endpoint, never
in the UI alone: three roles map to an explicit permission matrix. Uploaded files are not trusted
on the client-declared MIME type.

**Rate limiting shaped by what each route costs.** Login, general API traffic and the AI endpoints
have separate limits, and the AI endpoints additionally carry a per-user daily cap — because an
unbounded model call is a bill, not just a request. Those limiters run after authentication
precisely so they can key on the user.

**Domain rules encoded, not improvised.** Invoice numbering is correlative and resets per year,
VAT is applied and rounded consistently across quotes and invoices, and a quote converts into an
invoice without re-entering anything. This is how invoicing legally has to work, and getting it
wrong is not an option.

## Evaluation and operations

The AI features are covered by an evaluation harness rather than by impressions. On a single call
it captures **both the model's raw output and the verified output**, which makes it possible to
measure two different things: how much the model invents, and how much of that the verification
layer actually catches. Cases can be replayed against a different model, or repeated to measure
variance between runs, and results are written per model version — so upgrading the model is a
comparison rather than an assumption, and a regression in a generated quote gets caught before a
client sees it.

On the operational side there is a backup routine and a pre-deploy health check that reads the
production database without writing to it. That check exists because the test suite runs against a
local database file while production talks to a hosted one over HTTP — and those are not the same
thing, which is exactly the sort of difference that only shows up in production.

## Limitations and what I would do next

- **Invoice numbering derives the next number from the current maximum for the year.** It holds
  in normal use, but deleting the last invoice of a year would reuse a number. Moving the counter
  into a dedicated series table is the pending fix, and it is a correctness issue rather than a
  cosmetic one.
- **Drafted prices are a starting point, not an answer** — and they improve as the company's own
  history accumulates, which is the mechanism the design leans on.
- **Windows and doors are not reliably separated from thin partitions** in wall detection, since a
  window frame is about the width of a light partition, so they are corrected by hand.
- **The calculated floor area is approximate.** It depends on how closely the detected footprint
  matches the real outline, and on the user's calibration.
- **Single service by design.** One application service plus a hosted database is the right shape
  for this business, and the wrong shape the moment it has to serve many teams at once.

## Why this project matters to me

It is the clearest example I have of the kind of work I want to do: sitting with the people who
have the problem, learning a domain I did not know, and being deliberate about where a model
genuinely helps and where it has to be kept out of the path. Most of the engineering here is not
the model call — it is the grounding, the guardrails, the evaluation, and the honesty about how
accurate the thing really is.

And it is in use, which is the part I care about most.

I am happy to walk through the codebase, the data model or any of these decisions in a conversation.
