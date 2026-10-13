# Case studies — systems in production

Engineering case studies from my own products and consulting engagements. Client-specific code
stays in private repositories; what follows is the problem, the approach, the stack and the
trade-offs. Happy to go deeper on any of them in a call.

Owner: **Ade Nur Hidayat** — founder and lead engineer at [PT ArvaTech Solusi Digital](https://arvatech.id).

---

## 1. Wiyata — placing Indonesian students in Japanese companies

**Problem.** A placement operator had to coordinate three sides at once: Japanese employers with
openings, Indonesian partner campuses with candidates, and the students themselves. Job blasting,
applications, interviews, selection and the Certificate of Eligibility (CoE) paperwork all lived in
spreadsheets and chat threads.

**Approach.** One fullstack application with a state machine per application: a posting is published
to campuses, students apply, the operator runs the selection pipeline (screening → interview →
verdict), and the system generates the CoE document set per candidate and tracks post-arrival
evaluation. Role-scoped views for campus staff, partner companies, and the internal team, all on
the same data model.

**Stack.** Next.js (single-repo fullstack app) · tRPC · Drizzle ORM · PostgreSQL · S3-compatible
object storage for documents.

**Trade-offs.** Server-side rendering plus tRPC end-to-end typing was chosen over a separate REST
API: the team is small, and one deployable artifact with compile-time contracts beat a distributed
boundary nobody had time to version properly.

## 2. Indah Mall — B2B wholesale marketplace

**Problem.** A wholesale business needed a marketplace where sellers list at wholesale price points,
buyers order per seller, and payments are held until fulfilment — with an admin console and a mobile
client that could not be allowed to drift from the same rules.

**Approach.** A domain backend owning the schema (sellers, products, orders, payments, escrow,
wallet, affiliate) with self-hosted authentication (Argon2id password hashing, JWT access tokens,
rotating refresh tokens), a storefront and a separate admin service reading the same database, plus a
mobile client. Payment lifecycle modelled explicitly rather than as a single status flag.

**Stack.** FastAPI · async SQLAlchemy · Alembic · PostgreSQL (asyncpg) · Redis · Cloudflare R2 ·
Flutter (mobile) · React 19 / TanStack Start (storefront).

**Trade-offs.** Two backend services share one schema instead of duplicating tables per surface —
cheaper to operate, at the cost of migrations always landing in the owning service first. That rule
is written down in the repos.

## 3. GarasiMini — catalogue with photo-based search

**Problem.** Collectors identify a diecast car from a photo, not from a model name. Text search was
useless for the main use case.

**Approach.** Every catalogue photo is embedded with a CLIP model at ingest and stored in `pgvector`;
a user photo is embedded on the fly and matched with an approximate nearest-neighbour query, with OCR
on the carded packaging to disambiguate variants. A separate batch pipeline normalises, grades and
uploads media to object storage so a bad photo never becomes a search reference.

**Stack.** Go API · PostgreSQL + `pgvector` · Cloudflare R2 · Flutter client.

**Trade-offs.** Embeddings live in PostgreSQL instead of a dedicated vector database: one fewer
service to run, and the catalogue is small enough that filtered ANN queries stay fast.

## 4. POS ArvaTech — point of sale for a coffee franchise

**Problem.** Branch staff needed a POS that understood per-branch menus and pricing, while the owner
needed loyalty, ingredient inventory and reports without buying a separate ERP.

**Approach.** Cashier flow with per-branch catalogue overrides, ingredient-level stock that is
decremented by recipes rather than by product, customer loyalty accrual, and reporting built as
first-class queries instead of nightly exports.

**Stack.** Next.js (App Router) · PostgreSQL · live at [pos.arvatech.id](https://pos.arvatech.id).

## 5. ARTeX — autonomous penetration-testing console

**Problem.** Offensive security work is repetitive: reconnaissance, scanning, exploitation attempts,
and then writing the report. The parts worth a human's time are the decisions in between.

**Approach.** An agent orchestration loop over a tool runtime: the console drives recon and scanning,
records traffic, reasons over findings with an LLM (bring your own provider — Anthropic or any
OpenAI-compatible gateway), and produces a structured report. Deployed as a self-hosted service
bound to a private network, never exposed publicly by default.

**Stack.** Go backend · Next.js console · PostgreSQL · Docker · AGPL-3.0.

**Trade-offs.** Model-agnostic by design: the agent talks to a provider profile rather than a
specific vendor, so a dead API key or a cheaper model is a config change — not a rewrite.

---

### Notes on how these were run

- Schema changes went through versioned migrations; no hand-edited production databases.
- Deploys were verified against the running artefact (health endpoints, real requests) rather than
  assumed from a green build.
- Secrets stayed in environment configuration; none of these repositories contain credentials.
