# Chapter 07 — Background Jobs: Analogy Explanations

---

## Analogy 1: Background Jobs — The Restaurant Kitchen Order System

A web request is a diner placing an order at the table:
- The waiter (web server) takes the order and immediately responds: "Your order has been received" (HTTP 200)
- The waiter doesn't cook the food — they hand the ticket to the kitchen (job queue)
- The kitchen (Sidekiq workers) processes orders asynchronously
- When the food is ready, it comes out — the diner isn't waiting at the counter

Without background jobs: the waiter stands at the grill, cooking the food before returning to the table. Every customer waits while the waiter is cooking. The restaurant only serves one customer at a time.

ActiveJob is the standardized ticket format — it doesn't matter if the kitchen is Sidekiq, Delayed::Job, or GoodJob. The ticket format is the same.

---

## Analogy 2: Job Idempotency — The Vending Machine

A good vending machine is idempotent for coin insertion:
- You insert $1 and select chips → chips dispense, $0 credited
- If you insert $1 again → chips dispense again, correctly

A BAD vending machine:
- You insert $1, press the button, network hiccup
- You press the button again → machine dispenses twice, charges $2

An idempotent job is like adding a "have you already dispensed this order?" check:
- First button press: dispense = yes, mark order as dispensed
- Second button press: check → "already dispensed order #1234" → do nothing

Stripe's idempotency keys are like order numbers on the vending machine receipt. Even if you press the button 5 times with the same receipt number, it only charges once.

---

## Analogy 3: Sidekiq Queues — The Hospital Triage System

A hospital emergency department has different queues for different urgency levels:

- **Critical queue**: cardiac arrest, severe trauma → treated immediately
- **Urgent queue**: broken bones, moderate bleeding → treated within 1 hour
- **Standard queue**: minor cuts, cold symptoms → treated when capacity allows

Sidekiq queues work the same way:
- `queue :critical` → Sidekiq checks this queue 10x more often
- `queue :default` → Processes at normal rate
- `queue :low` → Only processed when critical and default are empty

The triage nurse (Sidekiq scheduler) always checks the critical room first. If there's always someone in the critical room, patients in the waiting room (low priority queue) might never get seen. This is queue starvation — a real problem to design around.

---

## Analogy 4: Sidekiq Retry with Backoff — The Car Jumper Cables

Your car won't start. You try jumper cables:
- Attempt 1 (immediate): try → failed
- Attempt 2 (25 seconds later): check if the other car is charged up → failed
- Attempt 3 (5 minutes later): let the battery build up more → success!

Sidekiq's exponential backoff is the increasing wait times:
- `(retry_count ** 4) + 15 + rand(10) * (retry_count + 1)` seconds
- Retry 1: ~15 seconds
- Retry 5: ~60 seconds
- Retry 10: ~3 hours
- Retry 25: ~21 days

The increasing wait time is respectful of whatever caused the failure:
- Transient network blip → usually fixed within seconds
- External service down → may take hours
- Database overloaded → backoff prevents making things worse

After 25 retries (default), the job goes to the Dead Queue (the mechanic's impound lot) — too many failures, needs human inspection.

---

## Analogy 5: Job Enqueueing Inside a Transaction — The Check That Bounces

Imagine you write a check to a contractor before your salary deposit clears:
- You hand them the check (enqueue the job inside a transaction)
- They run to the bank immediately
- The check bounces (transaction rolls back) — the money was never there
- Contractor did work for a non-existent check

`after_create_commit` is waiting until the salary deposit clears before writing the check:
- Transaction commits → money is in account → write the check
- If transaction fails → check is never written → contractor not called

The outbox pattern is keeping a "checks to write" ledger inside your account:
- When salary clears, you write all pending checks from the ledger atomically
- The ledger and the salary are part of the same account record

---

## Analogy 6: GlobalID — The Federal Express Tracking Number

When you ship a package with FedEx:
- You have the actual package (the ActiveRecord object in memory)
- FedEx gives you a tracking number: `7489489302` (the GlobalID)
- You can send the tracking number to anyone
- They can look up the package status at any time: FedEx.lookup("7489489302")

GlobalID works the same way:
- `user.to_global_id` → `"gid://myapp/User/42"`
- You pass this string (not the object) to the job queue (Redis)
- When the job runs: `GlobalID::Locator.locate("gid://myapp/User/42")` → `User.find(42)`

The danger: the package might be picked up (user deleted) by the time the tracking number is resolved. A tracking number for a delivered package still exists, but there's nothing to pick up.

---

## Analogy 7: Dead Job Queue — The Lost and Found Office

After 25 failed delivery attempts, the postal service takes your undeliverable package to the Lost and Found (Dead Job Queue):
- Every undelivered item is kept for a set period
- Supervisor reviews them periodically
- Some can be re-sent: the address was wrong but can be corrected (bug fixed, retry job)
- Some are discarded: recipient no longer exists (user deleted, discard)

The Dead Job Queue in Sidekiq is Sidekiq Web's "Lost and Found":
- Sidebar shows count of dead jobs
- Click into each to see error and payload
- "Retry" button: re-enqueue with current payload
- "Delete" button: discard permanently

Good practice: set up monitoring alerts when dead queue grows (something is systematically failing) and review dead jobs daily as part of operations.

---

## Analogy 8: Connection Pool vs Sidekiq Threads — The Parking Lot Problem

A restaurant has 10 employees (Sidekiq threads) but only 5 parking spots (DB connections):

Without planning:
- All 10 employees arrive at 9am
- Only 5 can park
- Other 5 circle the lot waiting
- After 5 minutes waiting (connection timeout), 5 employees give up and go home (ConnectionTimeoutError)
- Restaurant runs with half staff — or jobs fail

With planning:
- Ensure parking lot has enough spots for all workers: 10 threads → 12 parking spots (10 + 2 overflow)
- Or use a parking garage (PgBouncer connection pooler) that efficiently manages more virtual spots than physical ones

The formula: DB connection pool ≥ Sidekiq concurrency + web thread overhead

PgBouncer is the parking garage — it multiplexes many app connections onto fewer real DB connections using transaction-mode pooling, so 100 app "connections" might only use 20 real DB connections at any time.

---

## Analogy 9: Batch Jobs — The Mail Sorting Conveyor Belt

Sending 500,000 emails one at a time is like a postal worker hand-delivering each letter individually — they never finish before more mail arrives.

Batch jobs are the conveyor belt system:
- `NewsletterCampaignJob` is the loader that puts bundles onto the belt (enqueues 1000 at a time)
- `NewsletterEmailJob` workers are at different stations, each processing one letter from the belt
- Multiple workers at different stations → parallel processing
- Belt moves continuously → no single point of bottleneck

The "batch coordinator job" pattern:
1. Coordinator enqueues 1000 individual jobs, then schedules next coordinator in 5 seconds
2. Next coordinator enqueues next 1000, and so on
3. Workers independently process their assigned jobs

This way: coordinator does minimal work (just enqueueing), workers do the actual sending, progress is naturally distributed across all workers.

---
