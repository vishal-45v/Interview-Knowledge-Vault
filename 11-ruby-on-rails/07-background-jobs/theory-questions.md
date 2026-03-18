# Chapter 07 — Background Jobs: Theory Questions

---

**Q1.** What is ActiveJob and what problem does it solve? How does it relate to Sidekiq?

**Q2.** Explain the difference between `perform_later`, `perform_now`, and `perform_in`. When would you use each?

**Q3.** What is job idempotency? Why is it critical for background jobs? Give three concrete examples of jobs that must be idempotent.

**Q4.** How does Sidekiq's retry mechanism work? What is the default retry schedule? How do you configure custom retry behavior?

**Q5.** What are Sidekiq queues? How do you prioritize certain jobs over others? What happens if a high-priority queue is always full?

**Q6.** Explain the difference between Sidekiq's `sidekiq_options` and ActiveJob's `queue_as`. Which takes precedence when using ActiveJob with the Sidekiq adapter?

**Q7.** What is a "poison pill" job? How do you detect and handle jobs that consistently fail and pollute the dead queue?

**Q8.** How does Sidekiq handle job uniqueness (preventing duplicate jobs)? What gem provides this and what are its limitations?

**Q9.** What is the purpose of `around_perform` in ActiveJob? Give a real-world use case beyond logging.

**Q10.** Explain Sidekiq's threading model. How many threads does Sidekiq use by default? How does this interact with ActiveRecord's connection pool?

**Q11.** What is the Dead Job Queue in Sidekiq? When does a job end up there? How do you programmatically retry or discard dead jobs?

**Q12.** How do you test background jobs in RSpec? What is the difference between testing with `perform_enqueued_jobs` vs directly calling `perform_now`?

**Q13.** What is a Sidekiq Pro feature called "batches"? What problem does it solve? Can you replicate it in open-source Sidekiq?

**Q14.** How do you handle large payloads in background jobs? What are the risks of passing ActiveRecord objects vs IDs?

**Q15.** What is the purpose of `GlobalID` in Rails? How does it enable passing ActiveRecord objects in job arguments?

**Q16.** What is Sidekiq's `sidekiq_throttle` or rate-limiting? How would you implement custom rate-limiting for jobs that call an external API?

**Q17.** Explain the "outbox pattern" for reliable job enqueueing. Why might a job enqueued inside a transaction not execute?

**Q18.** What is `GoodJob`? How does it differ from Sidekiq in terms of infrastructure requirements?

**Q19.** How do you implement job chaining in ActiveJob? What are the tradeoffs of job chaining vs a single monolithic job?

**Q20.** What metrics should you monitor for background job health? What are the warning signs of a struggling job queue?

---
