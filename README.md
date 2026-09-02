# Construction Management Platform — case study

A CRM, quoting and invoicing system I designed, built and maintain for a small construction
company (my family's business). It runs in production and is used day to day to manage
clients, quotes, invoices and jobs.

> **This repository is documentation only.** The source code and the database stay private —
> the system holds real client and financial data. This page describes what I built, the
> constraints I built it under, and why I made the decisions I made.

`Node.js` · `Express` · `SQLite (better-sqlite3)` · `Vanilla JS (ES modules)` · `JWT + bcrypt` · `Zod` · `Helmet` · `Tailscale`

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

- No budget for SaaS subscriptions or cloud hosting.
- The people using it are not technical.
- It has to work from the office PC, a laptop and a phone.
- It handles client and financial data, so it must not sit exposed on the public internet.

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
Browser  (office PC · laptop · phone — all on a private network)
   │
   │  httpOnly JWT cookie
   ▼
Express API
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
to compile, nothing to redeploy, no toolchain to keep alive. For a system that has to keep
working for years with occasional maintenance from one person, a dependency-light stack is a
feature, not a limitation.

**SQLite as a single file.** The whole database is one file on disk. That makes backups a file
copy, which is something a non-technical user can actually be taught to do — and it removes an
entire class of database administration from a business that has nobody to do it.

**Security sized for a small business, but not skipped.** Passwords are hashed with bcrypt,
sessions are JWTs in httpOnly cookies, every request body is validated with Zod, and Helmet and
rate limiting are in front. Access control is role-based and enforced **per endpoint**, not in
the UI: three roles (full administration, day-to-day operations without deleting or configuring,
and read-only) map to an explicit permission matrix.

**A private network instead of a public server.** Rather than exposing the machine to the
internet or paying for hosting, the system is reachable only over a private mesh VPN
(Tailscale). Every device that needs it joins the network; nothing else can reach the server.
No open ports, no public attack surface, no monthly cost.

**Domain rules encoded, not improvised.** Invoice numbering is correlative and resets per
year, VAT is applied and rounded consistently across quotes and invoices, and a quote converts
into an invoice without re-entering anything. These are not arbitrary product decisions —
they are how invoicing legally has to work, and getting them wrong is not an option.

**Bilingual documents.** Quotes and invoices print in Spanish or Catalan, because that is what
the clients actually need to receive.

## Limitations and what I would do next

- **User management is still done directly in the database.** It works at this scale, but an
  admin screen for creating and deactivating users is the first thing I would add.
- **Backups are manual.** A file copy is simple, but it depends on somebody remembering; an
  automated scheduled copy is the obvious improvement.
- **No automated test suite.** The system was validated in use rather than in CI. For anything
  larger, or with more than one maintainer, that would not be acceptable.
- **Plans, drawings and project documents still live outside the system.** A quote
  describes work that is defined in files held elsewhere; bringing those into the
  platform is the natural next step.
- **Single-node by design.** SQLite and one machine are the right call for this business, and
  the wrong call the moment it needs concurrent write-heavy access from many sites.

## Why this project matters to me

It is the clearest example I have of the kind of work I want to do: sitting with the people who
have the problem, understanding a domain I did not know, choosing boring and robust technology
on purpose, and shipping something that is still in use — rather than something that only ever
had to work in a demo.

I am happy to walk through the codebase, the data model or any of these decisions in a
conversation.
