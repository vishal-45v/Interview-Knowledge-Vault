# Chapter 07 — Background Jobs: Scenario Questions

---

**Scenario 1:** Your app sends a welcome email when a user registers. The email job is enqueued in an `after_create` callback. A user registers, but the database transaction rolls back (due to another callback failure). Yet the job already ran and sent a welcome email to a user who doesn't exist in the DB. How do you fix this?

---

**Scenario 2:** You have a `ProcessPaymentJob` that charges a customer's credit card via Stripe. Due to a network hiccup, Sidekiq retries the job 3 times. The customer gets charged 3 times. Fix this design.

---

**Scenario 3:** Your background jobs are piling up — the queue depth is growing faster than jobs are being processed. The Sidekiq dashboard shows 50,000 jobs in the queue. Walk through your diagnosis and remediation plan.

---

**Scenario 4:** You need to send a newsletter to 500,000 users. If you enqueue one job per user, you'll have 500,000 jobs. Design an efficient batching strategy.

---

**Scenario 5:** A job imports a large CSV file (100,000 rows). Halfway through, the Sidekiq process is killed (SIGKILL during deploy). When the job retries, it re-processes already-imported rows, creating duplicates. How do you make this job resumable?

---

**Scenario 6:** You have a `SyncExternalDataJob` that calls an external API that has a rate limit of 100 requests/minute. Your queue can process 500 jobs/minute. How do you implement rate limiting so jobs wait and retry without overwhelming the external API?

---

**Scenario 7:** Your `GenerateReportJob` takes 45 minutes to run. Your Sidekiq worker timeout is set to 25 minutes. Jobs are getting killed mid-execution. What are your options without just raising the timeout?

---

**Scenario 8:** You need to implement a multi-step workflow: 1) Fetch user data from external API, 2) Process and transform data, 3) Save to database, 4) Send notification. Each step can fail independently. Design this as a job pipeline.

---

**Scenario 9:** A bug in production caused 10,000 jobs to be enqueued with bad data and fail immediately. They're all in the dead queue. You've deployed a fix. How do you re-enqueue them efficiently?

---

**Scenario 10:** Your app uses Sidekiq, and developers can't reproduce background job behavior locally because Redis isn't set up on all machines. How do you configure the test environment to run jobs inline and the development environment to offer a choice?

---

**Scenario 11:** You're building a feature where users can export their data. Exports can take 1-10 minutes. The UI should show export progress (0%, 25%, 50%, 100%) and provide a download link when complete. Design the job and the progress-reporting mechanism.

---

**Scenario 12:** Your Sidekiq jobs connect to the database, but under heavy load, jobs are failing with `ActiveRecord::ConnectionTimeoutError: could not obtain a database connection within 5 seconds`. What's happening and how do you fix it?

---
