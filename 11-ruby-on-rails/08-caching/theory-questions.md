# Chapter 08 — Caching: Theory Questions

---

**Q1.** What are the different caching strategies in Rails? Explain the difference between page caching, action caching, fragment caching, and HTTP caching.

**Q2.** How does Rails.cache work? What are the built-in cache store implementations and when would you use each?

**Q3.** Explain fragment caching with `cache` in views. How does Rails generate the cache key for a model object? What does it include?

**Q4.** What is Russian Doll caching? How does it work and what makes it efficient for nested data structures?

**Q5.** What is `touch` in ActiveRecord? How does it integrate with Russian Doll caching to propagate cache invalidation up the parent chain?

**Q6.** What is HTTP caching? Explain the difference between ETags (conditional GET) and Last-Modified headers. When does each approach have advantages?

**Q7.** How does Rails' `stale?` method work? Write a controller action that correctly implements ETag-based caching.

**Q8.** What is a cache store's `fetch` method? How does it differ from `read` + `write`? What is the "dogpile effect" (cache stampede) and how do you prevent it?

**Q9.** Explain the `expires_in` and `race_condition_ttl` options in Rails.cache. How does `race_condition_ttl` prevent cache stampedes?

**Q10.** What is `counter_cache` and how does it relate to caching? What are its limitations vs storing counts in a separate Redis key?

**Q11.** How do you cache the result of an expensive database query? What cache key strategy should you use to ensure freshness?

**Q12.** What is cache warming? Why is it important on application startup? Give an example of when cache warming prevents performance degradation.

**Q13.** What is low-level caching vs view-level caching? When would you use `Rails.cache.fetch` in a model or service instead of the `cache` view helper?

**Q14.** How does ActionController::Caching::Pages page caching work? Why was it removed from Rails core and what replaced it?

**Q15.** What are the tradeoffs between Redis cache store and Memcache store for Rails.cache? What features does Redis offer that Memcache doesn't?

**Q16.** Explain the concept of cache invalidation. What are the three hard problems in computer science and how does Rails' auto-expiry approach help?

**Q17.** How do you cache in multi-server deployments? Why can't you use `:memory_store` in production with multiple Puma workers or servers?

**Q18.** What is the `cache_key` vs `cache_key_with_version` method on ActiveRecord objects? What changed in Rails 5.2?

**Q19.** How do you implement conditional caching — cache only for guest users but always fresh for logged-in users?

**Q20.** What is the `Vary` HTTP header? How does it affect CDN caching of responses that differ by `Accept-Language` or `Authorization`?

---
