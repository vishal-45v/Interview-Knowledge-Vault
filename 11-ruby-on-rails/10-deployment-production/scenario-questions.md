# Chapter 10 — Deployment & Production: Scenario Questions

---

**Scenario 1:** You deploy a new version of your Rails app at 2pm. At 2:03pm, you get alerts that 15% of requests are returning 500 errors. Error logs show `ActiveRecord::StatementInvalid: column "user_tier" does not exist`. What happened and how do you fix it immediately?

---

**Scenario 2:** You need to add a NOT NULL column `user_type` to the `users` table which has 5 million rows. If you use a standard `add_column :users, :user_type, :string, null: false` migration, what happens? How do you safely add this column with zero downtime?

---

**Scenario 3:** Your Rails app's `SECRET_KEY_BASE` was accidentally rotated (changed). What breaks and how do you fix it with minimal user disruption?

---

**Scenario 4:** Your Puma server is running with 2 workers and 5 threads per worker. Under load, you see `Puma worker timeout` errors and Puma is restarting workers. The CPU is at 30% but response times are over 30 seconds. What is likely happening and how do you fix it?

---

**Scenario 5:** You're deploying a Rails app to production for the first time. Write the checklist of production configuration settings, security headers, logging, error tracking, and monitoring you need to have in place before going live.

---

**Scenario 6:** Your app has a critical bug that was deployed 3 hours ago. You need to roll back to the previous version immediately. Walk through the rollback procedure, including the database migration consideration.

---

**Scenario 7:** You need to ship a feature that requires a significant schema change (renaming a column). Both the old and new column names are referenced in code. You can't have downtime. Describe the deployment strategy step by step.

---

**Scenario 8:** Your production app is running out of database connections. There are 5 Puma workers × 5 threads = 25 connections per app server, and you have 3 app servers = 75 connections. Your PostgreSQL is configured with `max_connections = 100`. You're near the limit. How do you solve this without just raising `max_connections`?

---

**Scenario 9:** A new team member needs to run the app locally. They need the production-equivalent environment variables including the database credentials and third-party API keys. How do you securely share these without putting them in the repository?

---

**Scenario 10:** Your Rails app deploys via Docker. After a deploy, you notice that some requests are handled by the old container and some by the new container (mixing old and new code). A user who signed in on the old code can't access pages on the new code. What configuration issue causes this and how do you prevent it?

---

**Scenario 11:** You want to gradually roll out a new checkout flow to 10% of users, then 50%, then 100% over 3 weeks. The team needs to be able to instantly disable it if something goes wrong. Implement this using feature flags.

---

**Scenario 12:** Your staging environment and production environment have different behaviors even though they run the same code. `Rails.env.staging?` doesn't exist. How do you correctly configure a staging environment that behaves exactly like production?

---
