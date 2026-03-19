# Chapter 10 — Deployment & Production: Theory Questions

---

**Q1.** Explain the difference between Puma's clustered mode and single mode. What are workers vs threads? When would you use each configuration?

**Q2.** What is zero-downtime deployment? What techniques does Rails/Puma support for it? What is `SIGUSR1` in Puma's phased restart?

**Q3.** What is the Rails credentials system? How does `credentials.yml.enc` work? How do you add credentials for different environments (development, staging, production)?

**Q4.** What is Kamal? How does it differ from Heroku, Capistrano, and other deployment tools? What infrastructure does it require?

**Q5.** What is a database migration in production? What are the risks of running `db:migrate` during a deployment? What is the "expand, migrate, contract" pattern?

**Q6.** What is a reverse proxy? What is the typical Nginx → Puma architecture? What does Nginx handle that Puma shouldn't?

**Q7.** Explain the `RAILS_ENV=production` startup requirements. What environment variables must be set before a Rails app can start in production?

**Q8.** What is asset precompilation? What does `rails assets:precompile` do? What is the Sprockets asset pipeline vs Propshaft?

**Q9.** What is health checking in production? What is the difference between a liveness probe and a readiness probe? What should each check?

**Q10.** What is log aggregation? What is the difference between structured (JSON) logging and unstructured logging? How do you configure Rails to output structured logs?

**Q11.** What is APM (Application Performance Monitoring)? What tools are commonly used with Rails (New Relic, Datadog, Scout)? What key metrics should you monitor?

**Q12.** What is the preboot feature in Heroku? What is `WEB_CONCURRENCY` and `RAILS_MAX_THREADS`? How do you calculate the right values?

**Q13.** Explain the concept of feature flags. What are the benefits? What tools (Flipper, Unleash, LaunchDarkly) integrate with Rails?

**Q14.** What is database connection pooling (PgBouncer)? When do you need it? What are the modes: session, transaction, statement?

**Q15.** What is error tracking in production? How do you integrate Sentry or Bugsnag with Rails? What information should error reports include?

**Q16.** What is the `SECRET_KEY_BASE` environment variable? What happens if it changes in production? What data is affected?

**Q17.** How do you handle long-running database migrations safely in production without downtime? What is `disable_ddl_transaction!` and when is it needed?

**Q18.** What is Content Security Policy (CSP)? How do you configure it in Rails 7? What is a CSP nonce and why is it needed with inline scripts?

**Q19.** What is graceful shutdown in Puma? What happens to in-flight requests when a SIGTERM is received? How do you configure the shutdown timeout?

**Q20.** What is the difference between blue-green deployment and rolling deployment? How would you implement each with a Rails app on Docker/Kubernetes?

---
