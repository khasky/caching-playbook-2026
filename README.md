# Application Caching Playbook

Practical guide for application caching, HTTP caching, data-layer caches, invalidation, and performance-aware design in Node.js, React, and TypeScript products.

> *If I were setting caching defaults for a product team today, I would treat staleness cost and invalidation events as part of the domain model — not as a Redis-shaped afterthought.*

Everyone says caching is easy until they ship stale pricing, broken permissions, double writes, or a "fast" system that is only fast when the cache is already warm.

This playbook exists because caching is not a single trick.
It is a stack of decisions:

- what to cache,
- where to cache it,
- how long it lives,
- what makes it invalid,
- who can trust it,
- and what happens when it is wrong.

If you do not model those tradeoffs deliberately, the cache stops being an optimization and starts becoming a second data system that lies.

---

## Table of Contents

- [Application Caching Playbook](#application-caching-playbook)
  - [Table of Contents](#table-of-contents)
  - [Companion playbooks](#companion-playbooks)
  - [What this playbook covers](#what-this-playbook-covers)
  - [The first rule of caching](#the-first-rule-of-caching)
  - [The caching layers I care about](#the-caching-layers-i-care-about)
    - [1) Browser and edge cache](#1-browser-and-edge-cache)
    - [2) HTTP response cache](#2-http-response-cache)
    - [3) Application data cache](#3-application-data-cache)
    - [4) Client state / frontend cache](#4-client-state--frontend-cache)
    - [5) Derived or materialized cache](#5-derived-or-materialized-cache)
  - [The main cache goals](#the-main-cache-goals)
  - [My default decision order](#my-default-decision-order)
  - [Cache patterns that are actually useful](#cache-patterns-that-are-actually-useful)
    - [Cache-aside](#cache-aside)
    - [Read-through](#read-through)
    - [Write-through](#write-through)
    - [Write-behind / async update](#write-behind--async-update)
    - [Stale-while-revalidate](#stale-while-revalidate)
  - [Key design matters more than cache technology](#key-design-matters-more-than-cache-technology)
  - [TTL is not a strategy](#ttl-is-not-a-strategy)
  - [What I cache carefully](#what-i-cache-carefully)
  - [Frontend caching principles](#frontend-caching-principles)
    - [Server state](#server-state)
    - [UI state](#ui-state)
    - [Session-scoped state](#session-scoped-state)
  - [Backend caching principles](#backend-caching-principles)
    - [Cache expensive reads, not unclear behavior](#cache-expensive-reads-not-unclear-behavior)
    - [Avoid caching mutable business truth without a plan](#avoid-caching-mutable-business-truth-without-a-plan)
    - [Protect upstreams](#protect-upstreams)
  - [Stampede and hot-key protection](#stampede-and-hot-key-protection)
  - [Caching and multitenancy](#caching-and-multitenancy)
  - [Caching and deployments](#caching-and-deployments)
  - [Anti-patterns](#anti-patterns)
  - [Review checklist](#review-checklist)
    - [Scope](#scope)
    - [Safety](#safety)
    - [Invalidation](#invalidation)
    - [Performance](#performance)
    - [Operations](#operations)
  - [Final note](#final-note)
  - [License](#license)

---

## Companion playbooks

These repositories form one playbook suite:

- [Auth & Identity Playbook](https://github.com/khasky/auth-identity-playbook) — sessions, tokens, OAuth, and identity boundaries across the stack
- [Backend Architecture Playbook](https://github.com/khasky/backend-architecture-playbook) — APIs, boundaries, OpenAPI, persistence, and errors
- [Best of JavaScript](https://github.com/khasky/best-of-javascript) — curated JS/TS tooling and stack defaults
- **Caching Playbook** — HTTP, CDN, and application caches; consistency and invalidation
- [Code Review Playbook](https://github.com/khasky/code-review-playbook) — PR quality, ownership, and review culture
- [DevOps Delivery Playbook](https://github.com/khasky/devops-delivery-playbook) — CI/CD, environments, rollout safety, and observability
- [Engineering Lead Playbook](https://github.com/khasky/engineering-lead-playbook) — standards, RFCs, and technical leadership habits
- [Frontend Architecture Playbook](https://github.com/khasky/frontend-architecture-playbook) — React structure, performance, and consuming API contracts
- [Marketing and SEO Playbook](https://github.com/khasky/marketing-and-seo-playbook) — growth, SEO, experimentation, and marketing surfaces
- [Monorepo Architecture Playbook](https://github.com/khasky/monorepo-architecture-playbook) — workspaces, package boundaries, and shared code at scale
- [Node.js Runtime & Performance Playbook](https://github.com/khasky/nodejs-runtime-performance-playbook) — event loop, streams, memory, and production Node performance
- [Testing Strategy Playbook](https://github.com/khasky/testing-strategy-playbook) — unit, integration, contract, E2E, and CI-friendly test layers

---

## What this playbook covers

This repository is for engineers designing caching in:

- React frontends
- Node.js APIs and BFFs
- server-rendered and hybrid applications
- data-heavy dashboards and SaaS products
- high-read internal tools
- APIs with expensive queries or third-party integrations

It covers caching as a **system design discipline**, not just as a Redis tutorial.

---

## The first rule of caching

Do not start by asking:

> "Where should I put Redis?"

Start by asking:

> "What kind of repeated work am I trying to avoid, and what level of staleness is acceptable?"

That one shift improves almost every caching decision.

---

## The caching layers I care about

I think about caching as a layered system.

### 1) Browser and edge cache

Good for:
- static assets
- immutable bundles
- images
- CDN-served content
- public content with stable freshness rules

Main concern:
- cache-control strategy
- invalidation by URL/versioning
- content hashing
- stale assets after deploys

---

### 2) HTTP response cache

Good for:
- public API responses
- generated pages
- documentation
- content endpoints
- query results that tolerate bounded staleness

Main concern:
- cacheability semantics
- ETag / Last-Modified behavior
- private vs public responses
- auth-sensitive variation
- surrogate invalidation

---

### 3) Application data cache

Good for:
- expensive read paths
- computed summaries
- repeated query results
- reference datasets
- slow upstream API responses

Main concern:
- key design
- invalidation
- stampede protection
- consistency expectations
- tenant/user scoping

---

### 4) Client state / frontend cache

Good for:
- server-state reuse
- deduping requests
- optimistic UI
- pagination results
- background refresh

Main concern:
- cache lifetime and revalidation
- stale UI boundaries
- mutation invalidation
- user/tenant/session isolation

---

### 5) Derived or materialized cache

Good for:
- search documents
- analytics aggregates
- dashboards
- timeline views
- recommendation outputs
- precomputed joins

Main concern:
- freshness pipeline
- recomputation timing
- backfill strategy
- source-of-truth clarity

---

## The main cache goals

A cache can optimize different things.
Know which one you are buying.

- **Latency**
- **Throughput**
- **Cost**
- **Upstream protection**
- **Resilience**
- **User experience smoothness**

The mistake is acting like all caches are about the same thing.

A cache that protects a rate-limited third-party API is not the same design problem as a cache that makes a table render faster.

---

## My default decision order

When adding a cache, I walk through this order:

1. What work is expensive?
2. Is that work deterministic enough to cache?
3. Who is allowed to see the result?
4. What event makes the cached value unsafe or obsolete?
5. Can bounded staleness be tolerated?
6. Can the system recover safely from a cache miss or stale hit?
7. What is the authoritative source of truth?
8. How will the team debug wrong-cache behavior?

If you cannot answer #4 and #8, you probably are not ready to ship the cache.

---

## Cache patterns that are actually useful

### Cache-aside

The classic default.

Flow:
- read from cache,
- on miss, read source,
- write to cache,
- return result.

Good when:
- reads dominate,
- the source of truth is elsewhere,
- occasional misses are acceptable.

Watch out for:
- dogpiles,
- stale reads after writes,
- key inconsistency,
- inconsistent TTLs across codepaths.

---

### Read-through

Useful when the caching layer abstracts loading behavior and callers should not repeat miss logic.

Good when:
- you want a single loading contract,
- many codepaths need the same cached resource.

Watch out for:
- hiding expensive fallback work,
- accidentally making cache logic too magical.

---

### Write-through

Write cache and source as part of the same write path.

Good when:
- read-after-write freshness matters,
- data shape is simple,
- write path can tolerate the extra coordination.

Watch out for:
- extra write latency,
- cache becoming too coupled to domain writes.

---

### Write-behind / async update

Useful for buffered or derived systems, but dangerous if misunderstood.

Good when:
- throughput matters,
- derived data is acceptable,
- failure/replay is well designed.

Watch out for:
- delayed persistence assumptions,
- silent data loss,
- debugging complexity.

---

### Stale-while-revalidate

One of the highest-leverage patterns for UI and content systems.

Good when:
- freshness matters, but not on every millisecond,
- fast perceived response matters,
- background refresh can reconcile.

Watch out for:
- hiding source-of-truth issues with too much stale tolerance,
- users seeing stale permissions or billing state.

---

## Key design matters more than cache technology

A lot of cache bugs are really key bugs.

Good keys usually encode:
- entity or query shape
- tenant / org / workspace
- user scope if needed
- locale / currency / feature version when relevant
- schema version
- data source or representation variant

Example:

```text
workspace:{workspaceId}:invoices:list:v3:status=open:currency=usd
```

Weak keys usually look like:

```text
invoices
```

That is not a key.
That is a future incident.

---

## TTL is not a strategy

TTL is a guardrail.
Not a complete invalidation model.

A mature cache design distinguishes:

- **freshness policy** — How long can we serve this confidently?
- **invalidation events** — What domain changes make it wrong immediately?
- **eviction policy** — What disappears under memory pressure or lifecycle rules?

If you only have TTLs, you do not really have invalidation.
You have hopeful expiration.

---

## What I cache carefully

Some data is high-risk to cache without strong guardrails:

- permissions and authorization-sensitive views
- billing state
- inventory / stock / availability
- pricing
- account or subscription status
- security-sensitive user profile state
- anything users will call "wrong" faster than they call it "fast"

I am not saying "never cache".
I am saying "be explicit about staleness cost".

---

## Frontend caching principles

For React products, I like these distinctions:

### Server state

Fetched from the server.
Potentially shared.
Revalidated over time.

Examples:
- lists
- detail views
- search results
- dashboards
- reference data

This belongs in a server-state layer, not random component state.

---

### UI state

Purely local interaction state.

Examples:
- modal open/closed
- current tab
- filter controls before apply
- draft input state

Do not overengineer this into remote caching.

---

### Session-scoped state

Tenant, workspace, role context, current user, selected environment.

Be careful here:
- cross-user contamination is bad,
- cross-tenant contamination is worse.

When auth context changes, related caches often need to be invalidated aggressively.

---

## Backend caching principles

### Cache expensive reads, not unclear behavior

Examples of good candidates:
- configuration lookups
- reference tables
- expensive aggregate queries
- third-party enrichment responses
- repeated document reads
- rendered fragments

### Avoid caching mutable business truth without a plan

Examples:
- active subscription status
- permissions
- payment settlement state
- mutable checkout/cart state

### Protect upstreams

Caching is often less about speed and more about:
- protecting a database from repeated fan-out,
- shielding a vendor API,
- surviving burst traffic.

This is a valid reason.
Treat it seriously.

---

## Stampede and hot-key protection

If a thousand requests miss the same expensive key at once, your cache did not save you.

You need some combination of:
- single-flight loading
- request coalescing
- jittered TTLs
- refresh-ahead for hot keys
- bounded background recompute
- negative caching where appropriate
- rate limiting on fallback path

This is where "works in dev" caching dies in production.

---

## Caching and multitenancy

Never treat tenant scope as optional in shared systems.

Ask:
- can this data be shared across tenants?
- does tenant plan affect representation?
- do permissions vary per tenant?
- does region matter?
- can support or internal roles see different data?

Missing tenant scope in cache keys is one of the most embarrassing classes of production bugs.
For good reason.

---

## Caching and deployments

A lot of cache problems are release problems.

Plan for:
- representation/version mismatch across deploys
- schema changes
- different services interpreting the same cached payload differently
- invalid caches surviving longer than the code that wrote them

I like explicit cache key versioning for anything nontrivial.

---

## Anti-patterns

- adding Redis before understanding access patterns
- caching permission-sensitive data with shared keys
- using TTL as the only invalidation strategy
- caching everything because the database is slow
- no instrumentation for hit rate, miss rate, fallback latency, or stale serves
- no cache busting after writes that obviously change read models
- treating stale data bugs as "just refresh"
- putting too much faith in local in-memory cache inside horizontally scaled systems

---

## Review checklist

### Scope

- What exactly is being cached?
- Why is it expensive enough to justify this?
- Is the primary goal latency, cost, resilience, or upstream protection?

### Safety

- Who is allowed to see this data?
- Is the key scoped by user, tenant, locale, plan, or auth context where needed?
- What are the consequences of stale data?

### Invalidation

- What domain events invalidate it?
- Is TTL just a fallback or the only policy?
- How is cache versioning handled across deploys?

### Performance

- What is the expected hit rate?
- What happens during a hot miss?
- Is stampede protection needed?

### Operations

- Can we observe cache hit/miss/stale/fallback behavior?
- How do we debug wrong data?
- How is the cache warmed or repopulated after flushes?

---

## Final note

Great caching is not about making everything fast.

It is about deciding, with discipline, what can safely be reused, for how long, at what scope, and with what recovery behavior when reality changes underneath it.

---

## License

MIT is a sensible default for a repository like this, but choose the license that fits how you want others to reuse the material.
