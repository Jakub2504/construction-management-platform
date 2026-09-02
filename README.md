# Construction Management Platform — case study

A CRM, quoting and invoicing system I designed, built and maintain for a small construction
company (my family's business). It runs in production and is used day to day to manage
clients, quotes, invoices and jobs.

> **This repository is documentation only.** The source code and the database stay private —
> the system holds real client and financial data. This page describes what I built, the
> constraints I built it under, and why I made the decisions I made.

`Node.js` · `Express` · `SQLite (better-sqlite3)` · `Vanilla JS (ES modules)` · `JWT + bcrypt` · `Zod` · `Helmet` · `Render`

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
- It has to be reachable from anywhere — the office, a laptop, a phone on site — not tied to
  one computer being switched on.
- It holds client and financial data and it is reachable from the internet, so access control
  has to be real rather than cosmetic.
- It has to stay maintainable by one person, working on it occasionally.

## What it does

- **Clients (CRM)** — client records and the work associated with each one.
- **Quotes** — line-item editor with automatic subtotals, VAT and totals, and correlative numbering.
- **Invoices** — generated from an existing quote, numbered correlatively **per year**, with payment status tracking.
- **Jobs** — construction work linked to clients.
- **Pipeline & dashboard** — the current state of the business at a glance.
- **Printable documents** — bilingual (Spanish / Catalan) quotes and invoices, produced straight from the browser.
- **Company settings** — the fiscal and contact data that appears on every document.

## Architecture

```text
Browser  (office PC · laptop · phone — from anywhere)
   │
   │  httpOnly JWT cookie over HTTPS
   ▼
Express API  —  deployed on Render
   │   Helmet  ·  rate limiting  ·  Zod validation  ·  RBAC on every endpoint
   │
   ├── auth · clients · quotes · invoices · jobs · dashboard · company
   │
   ▼
SQLite  (single file, better-sqlite3)
```

The frontend is a small single-page app written in plain ES modules: a router, an API client,
and one view per screen. Documents are rendered with a dedicated print stylesheet, so the
browser produces the PDF and the system needs no PDF library.

## Engineering decisions

The interesting part of this project was never the CRUD. It was the constraints.

**No framework and no build step.** The frontend is native ES modules and plain CSS. Nothing
to compile, nothing to keep in sync, no toolchain to maintain. For a system that has to keep
working for years with occasional attention from one person, a dependency-light stack is a
feature, not a limitation.

**SQLite as a single file.** The entire database is one file, which keeps the operational
surface small: no separate database service to provision, secure or pay for. For one
maintainer and a handful of users that is the right trade — and it is easy to recognise the
point at which it would stop being.

**Deployed as a hosted service.** The platform runs on Render rather than on a machine in the
office, so it is reachable from anywhere without anyone maintaining a server and without
depending on one computer being switched on. The trade-off is that it is exposed to the
internet — which is exactly why the security below is load-bearing rather than decorative.

**Security that has to actually hold.** Passwords are hashed with bcrypt, sessions are JWTs in
httpOnly cookies, every request body is validated with Zod, and Helmet and rate limiting sit in
front. Authorisation is role-based and enforced **per endpoint**, never in the UI alone: three
roles — full administration, day-to-day operations without deleting or configuring, and
read-only — map to an explicit permission matrix.

**Domain rules encoded, not improvised.** Invoice numbering is correlative and resets per
year, VAT is applied and rounded consistently across quotes and invoices, and a quote converts
into an invoice without re-entering anything. These are not arbitrary product decisions —
they are how invoicing legally has to work, and getting them wrong is not an option.

**Bilingual documents.** Quotes and invoices print in Spanish or Catalan, because that is what
the clients actually need to receive.

## Limitations and what I would do next

- **User management is still done directly in the database.** It works at this scale, but an
  admin screen for creating and deactivating users is the first thing I would add.
- **Backups need a deliberate routine.** A single-file database makes backups simple in
  principle, but simple is not the same as automatic.
- **No automated test suite.** The system was validated in use rather than in CI. For anything
  larger, or with more than one maintainer, that would not be acceptable.
- **Plans, drawings and project documents still live outside the system.** A quote describes
  work that is defined in files held elsewhere; bringing those in is the natural next step.
- **Single-node by design.** SQLite and a single service are the right call for this business,
  and the wrong call the moment it needs concurrent write-heavy access from many places.

## Why this project matters to me

It is the clearest example I have of the kind of work I want to do: sitting with the people who
have the problem, understanding a domain I did not know, choosing boring and robust technology
on purpose, and shipping something that is still in use — rather than something that only ever
had to work in a demo.

I am happy to walk through the codebase, the data model or any of these decisions in a
conversation.
