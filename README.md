# HydraFlow

**A full-stack automation platform that unifies inventory, pricing, and order management across a multi-channel e-commerce operation — built and owned end-to-end, from database schema to live dashboard.**

🔗 **Live product:** [https://dev.buyboxgr.com/login.html](https://dev.buyboxgr.com/login.html)
*(A private, in-production business system built for a real company. Not open for public sign-up — included here as a reference for engineering review.)*

---

## What This Project Is

HydraFlow is the backend and internal dashboard I designed and built for a retail operation that sells the same product catalog across a supplier and several marketplace storefronts simultaneously. Before this system existed, keeping stock levels and prices consistent across channels was manual, error-prone spreadsheet work, done by a small operations team every day.

HydraFlow replaces that entirely: it ingests supplier data automatically, recalculates sell prices per channel according to configurable business rules, keeps stock synchronized across every connected marketplace in real time, and gives the operations team a single dashboard instead of four separate platform logins.

This is a **live system in daily production use**, not a portfolio demo — every feature described below is running against a real product catalog and real order flow today.

I own this project end-to-end: system architecture, backend, frontend, database design, third-party API integration, and production debugging.

---

## System Capabilities

### Supplier & Catalog Automation
- Resilient, scheduled synchronization against a supplier's REST API — handling pagination, transient failures, timeouts, and partial-failure recovery without manual intervention.
- Bulk catalog ingestion from spreadsheet exports, with automatic creation of new catalog entries when source data is complete, rather than requiring every product to be pre-registered manually.
- Row-level fault isolation on bulk imports, so a single malformed record in a multi-thousand-row batch cannot silently corrupt the rest of the operation.

### Dynamic, Rule-Based Pricing Engine
- Per-channel markup configuration (percentage-based or fixed-amount), editable live from the dashboard with no deployment required.
- Tiered price-floor protection, where the minimum permissible sell price scales with the product's cost band — preventing accidental below-cost listings on low- and high-value items alike.
- A pricing model designed to be extended to additional channels or rule types without touching core calculation logic.

### Multi-Marketplace Order Synchronization
- Real-time order ingestion via a marketplace webhook integration, feeding a single unified order model shared across every sales channel — chosen over polling for lower latency and to match the actual constraints of the third-party API (which, on investigation, did not expose a polling endpoint at all).
- A second, independently integrated marketplace using a REST polling model — same unified downstream pipeline, different transport layer, demonstrating the system's adapter-based extensibility.
- Automatic stock decrement on order receipt, propagated outward to every other connected channel so inventory never oversells across platforms.
- Full order lifecycle visibility (incoming → shipped → returned → cancelled) per channel, in one dashboard.

### Operations Dashboard
- Live, computed (not cached) metrics: current stock levels, pending order counts, low-stock alerts.
- A delivery-reconciliation workflow comparing what was ordered from the supplier against what physically arrived, with discrepancy logging.
- Manual override controls, including per-SKU exclusion flags so specific products can be pinned out of automated pricing or stock updates when the business needs a manual hold.
- An independent background job scheduler running catalog sync, repricing, and feed generation on separate cycles, with human-readable structured logs surfaced directly in the UI so non-technical staff can self-diagnose issues without engineering involvement.

---

## Architecture

The system follows **Clean Architecture**, keeping core business rules independent of any specific framework, database, or external API:

```
domain/           Core business models and repository interfaces — zero framework dependencies
application/      Use cases and orchestration logic (pricing, inventory, order sync)
infrastructure/   Persistence, third-party API gateways, scheduled jobs
presentation/     REST API layer
```

This boundary discipline paid off directly during hardening: several production issues (below) were fully diagnosed and fixed within a single layer, without any risk of side effects elsewhere in the system — and new marketplace integrations can be added by implementing a small, well-defined interface rather than modifying shared logic.

**Technology stack:**

| Layer | Technology |
|---|---|
| Backend | Java, Spring Boot, Spring Data JPA |
| Database | PostgreSQL |
| Frontend | HTML / CSS / JavaScript dashboard |
| Integration | REST APIs, webhook consumption, scheduled background jobs |

---

## Engineering Highlights

A selection of real production issues I diagnosed and resolved, included here because they reflect the kind of problem-solving this project actually demanded — not just feature assembly:

**Transaction poisoning under bulk load.**
PostgreSQL aborts an entire database transaction after a single failed statement inside it. An early version of the bulk-import path wrapped an 8,000+ row loop in one transaction — meaning a single malformed row silently failed *every row processed after it*, with no clear signal as to why. I traced this to the transaction boundary itself and resolved it by isolating each row into its own independent transaction (via a self-injected Spring proxy to correctly trigger transactional advice), with per-row error detail surfaced back to the operator instead of a single opaque failure.

**Ambiguous SQL in a native upsert query.**
A hand-written `INSERT ... ON CONFLICT` query intermittently failed with an "ambiguous column reference" error under PostgreSQL, caused by an unqualified column name colliding with the implicit `EXCLUDED` row scope. Diagnosed and resolved via explicit table aliasing — a subtle SQL semantics issue that only surfaces under specific conflict conditions.

**Reverse-engineering a third-party integration from documentation alone.**
Mid-implementation, I discovered that a marketplace's orders API did not expose any list/polling endpoint at all — only a webhook push model and a "fetch by known ID" endpoint. Rather than retrofit a broken polling design, I read through the platform's full API and webhook documentation, re-derived the correct integration model, and rebuilt the order-ingestion path around a proper webhook receiver with payload validation and idempotent order upserts.

**Eliminating a silent integration failure.**
A third-party stock-sync gateway was failing on every call with no visible error anywhere in the system, because the failure was being swallowed several layers up. I made the failure explicit and isolated its blast radius so one broken downstream integration can no longer block or roll back unrelated operations.

---

## About the Developer

I'm a full-stack developer, and HydraFlow is the project I'd point to as the clearest example of how I work end-to-end: designing a system architecture from a messy real-world requirement, integrating with third-party APIs that don't behave the way their documentation implies, debugging production data issues under real time pressure, and making pragmatic decisions that keep the codebase maintainable as the business's needs keep changing.

**Contact:** [your email] · [LinkedIn] · [portfolio site, if applicable]

---

## Screenshots

*(See `/screenshots`. Store names, product identifiers, and other business-identifying details have been redacted for confidentiality; the underlying workflows and data are real.)*
