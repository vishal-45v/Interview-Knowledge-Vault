# Chapter 10 — Deployment & Production: Analogy Explanations

---

## Analogy 1: Puma Workers vs Threads — The Restaurant Kitchen

**Puma workers** are separate kitchen stations (separate OS processes):
- Station A handles Italian dishes
- Station B handles Sushi
- They don't share the same pans or ingredients (memory is separate)
- If Station A catches fire, Station B keeps working

**Puma threads** are the cooks within one kitchen station:
- 5 cooks at Station A, each handling a different order simultaneously
- They share the same refrigerator (memory, connections)
- If one cook is waiting for an oven to preheat (I/O wait), another cook uses the stove

**Configuration:**
- Workers: set based on how many stations you can fit (RAM)
- Threads: set based on how I/O-heavy your work is
- CPU-bound work (heavy computation): more workers, fewer threads
- I/O-bound work (database queries, API calls): more threads, fewer workers

**`preload_app!`** = all stations use the same base mise en place (shared memory for the Ruby VM via copy-on-write). Without it, each worker loads the full application independently.

---

## Analogy 2: Zero-Downtime Deploy — The Open-Heart Surgery Rule

A hospital can't close for renovations. They renovate one wing at a time while patients are in other wings.

**Phased restart (Puma's SIGUSR1):**
- Old worker A: still serving patients (requests)
- New worker B starts up, begins accepting new patients
- Once new worker B is serving: old worker A finishes current patients, then retires
- Old worker C continues serving while new worker D starts
- Eventually: all old workers replaced by new workers, no one was turned away

**Blue-Green deployment:**
- Blue environment: current production (100% traffic)
- Green environment: new version deployed and warmed up
- Load balancer flip: 100% traffic → green (atomic switchover)
- Blue: kept running for 10 minutes in case rollback needed
- Then: blue is decommissioned

The key constraint: during the transition, old code AND new code are running simultaneously against the SAME database. Your schema changes must be compatible with BOTH old and new code — hence Expand, Migrate, Contract.

---

## Analogy 3: Database Migrations in Production — Road Construction

You can't close the highway to repave it. You have to repave one lane at a time while traffic flows.

**Standard migration (dangerous):**
- Close all lanes (lock the table)
- Repave the whole road (add the column + constraint)
- Open all lanes
- Drivers (requests) wait 10 minutes → frustrated (timeouts)

**`CONCURRENTLY` index creation:**
- Leave all lanes open
- Work in the shoulder
- Takes 3x longer but traffic flows uninterrupted

**`NOT NULL` column addition (PostgreSQL 11+ optimization):**
- Add column with a constant DEFAULT → instant (metadata only, no table rewrite)
- In older PostgreSQL: add column (nullable) → backfill → add constraint

**Renaming a column (Expand, Migrate, Contract):**
- Phase 1: Open a new lane NEXT to the old one (new column)
- Both lanes open simultaneously (write to both columns)
- Phase 2: Redirect all traffic to new lane (switch reads to new column)
- Phase 3: Close old lane (drop old column)
- Total: 3 deploys over 2-3 days. No disruption.

---

## Analogy 4: `SECRET_KEY_BASE` — The Master Locksmith's Key

Your app uses a master key to lock and unlock all sessions, signed cookies, and encrypted data:
- Session cookie: "here's your identity card, stamped with our seal" (signed with master key)
- Encrypted cookie: "here's a locked safe only you can open" (encrypted with master key)

If the master locksmith leaves and you change all the locks (rotate `SECRET_KEY_BASE`):
- All old stamped identity cards are now unrecognized → everyone gets logged out
- All old sealed envelopes can't be opened → password reset links fail
- All old locked safes can't be unlocked → encrypted data inaccessible

**Graceful rotation** is like overlapping locksmiths:
- New locksmith starts → begins making new stamps
- Old locksmith stays for 1 week → still recognizes old stamps
- After 1 week: all old stamps have naturally expired (sessions timed out)
- Old locksmith leaves → no disruption

In Rails, this is `cookies_rotations` — new cookies use new key, but old cookies with old key are still accepted during the transition period.

---

## Analogy 5: Feature Flags — The Light Switch Panel

Traditional releases are light switches wired to the circuit breaker:
- To turn off a buggy feature: break the circuit (deploy a hotfix)
- Takes 5-30 minutes to "turn off"
- Everyone is affected simultaneously

Feature flags are wireless light switches on every circuit:
- To turn off: flip the switch (change one line in Redis/database)
- Takes 1 second
- Can be toggled per user, per group, per percentage

Gradual rollout is a dimmer switch:
- 10% → "it works!" → 25% → "still good" → 50% → 100%
- Problem detected at 25%: flip to 0% instantly — 75% of users never see the bug

**The factory:**
```ruby
Flipper.enable_percentage_of_actors(:new_checkout, 10)
# User 42 gets "user_42" → hash → mod 100 → 42 → 42 < 10? → No → disabled for this user
# User 7  gets "user_7"  → hash → mod 100 → 7  → 7  < 10? → Yes → enabled for this user
```

Deterministic: user 42 always gets the same result. They don't see the feature appear and disappear randomly.

---

## Analogy 6: PgBouncer — The Hotel Concierge Desk

Your hotel (PostgreSQL) has 100 rooms (max_connections). Each room can only have one guest at a time.

Without PgBouncer: every app thread holds a room key permanently, even when they're not in the room. With 200 app threads, you need 200 rooms → PostgreSQL max_connections = 200.

With PgBouncer (transaction mode): the concierge desk has 100 room keys. App threads check out a key from the concierge when they need a room, immediately return it when done.

- 200 app threads, 100 room keys
- At any moment, only ~20 threads are actively using rooms (others are doing computation, not DB work)
- PgBouncer recycles room keys: "Thread 5 finished with room 12 → give room 12 key to Thread 47"
- PostgreSQL only sees 100 connections (the concierge's pool), not 200

**Modes:**
- **Session mode**: key is held for the entire session (like a weekend stay) — most compatible but least efficient
- **Transaction mode**: key held only during a transaction (most efficient, requires compatible code — no `SET`, `PREPARE`, etc.)
- **Statement mode**: key held only during one statement (rarely used, very restrictive)

---

## Analogy 7: Content Security Policy — The Club's Approved Vendor List

A nightclub (your browser) has a strict policy: only approved vendors (domains) can bring in supplies (scripts, images, fonts).

Without CSP:
- Any vendor can walk in (any domain can load scripts → XSS attack possible)
- A rogue vendor injects bad substance into drinks (injected script steals cookies)

With CSP:
```
"Only these vendors are approved:
- Scripts: only from our own bar (self) and Stripe (js.stripe.com)
- Images: from our bar and any HTTPS source
- Fonts: from Google Fonts only
- Nothing from unknown vendors"
```

The nonce is like a day pass issued each morning:
- Club generates a new secret nonce each day (each request)
- Only vendors with today's pass can enter
- Attacker tries to inject a script: "INSERT <script src='evil.com'>" — no day pass → browser blocks it
- Your legitimate inline script has the nonce embedded → allowed

`'unsafe-inline'` is like "any person in a hoodie can enter" — defeats the whole purpose of the bouncer.

---

## Analogy 8: Graceful Shutdown — The Flight Crew Handoff

When a flight crew's shift ends mid-flight, they don't just walk off the plane:
1. New crew boards and gets briefed on the current flight state
2. Old crew stays until new crew is ready
3. Old crew hands over controls (in-flight requests complete)
4. New crew takes over fully
5. Old crew deploys

**Puma's graceful shutdown (SIGTERM):**
1. Puma receives SIGTERM (deploy signal)
2. Puma stops accepting new connections to this worker
3. Load balancer routes new requests to new worker
4. Old worker finishes all in-flight requests (up to 60 second timeout)
5. Old worker exits

Without graceful shutdown: old worker exits immediately → active requests get `Connection reset by peer` → users see errors mid-checkout, mid-form submission, mid-upload.

The `shutdown_timeout` is the "maximum delay" for the crew handoff. If old crew hasn't finished in 60 seconds, new deploy can't proceed. Long-running requests need this configured thoughtfully.

---
