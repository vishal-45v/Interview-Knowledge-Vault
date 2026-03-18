# Chapter 08 — Caching: Scenario Questions

---

**Scenario 1:** Your Rails app renders a product listing page. Each product card shows: product name, price, category, and the count of reviews. The page is slow because each product card triggers a `COUNT(*)` query for reviews. You have 200 products per page. How do you cache this?

---

**Scenario 2:** You implement Russian Doll caching for a blog post with comments. A commenter edits their username — the post's cache should invalidate. But it doesn't. What's missing from your model setup?

---

**Scenario 3:** A news website uses fragment caching for article cards. After publishing a new article, the homepage still shows old articles for several minutes. The cache has a 10-minute TTL. How do you implement cache invalidation so the homepage updates within seconds of a new publish?

---

**Scenario 4:** Your API serves a resource that rarely changes (a list of countries). The endpoint is called 10,000 times/minute. Implement HTTP caching so repeat requests don't hit your app at all.

---

**Scenario 5:** Your cache stampede scenario: you have `Rails.cache.fetch("expensive_data", expires_in: 1.hour)`. At 3am, 500 concurrent requests arrive right as the cache expires. All 500 requests execute the expensive query simultaneously, overloading the database. Fix this.

---

**Scenario 6:** You're caching user-specific content (e.g., "unread notifications count") in a view. After the user reads a notification, the cached count still shows the old number. Design a cache invalidation strategy for user-specific cached data.

---

**Scenario 7:** Your admin dashboard runs a complex aggregation query (`SUM`, `GROUP BY`, multiple JOINs) that takes 8 seconds. The dashboard is viewed 50 times/minute by multiple admins. Results only need to be accurate to within 15 minutes. Implement the caching solution.

---

**Scenario 8:** You're deploying to production. After deploy, some users still see the old UI because their browser cached old JavaScript/CSS. Some users see broken pages because the new JS references changed API endpoints. How do you solve this at the asset level and the API level?

---

**Scenario 9:** Your app runs on 5 Puma servers behind a load balancer. Each server has its own in-memory cache (`:memory_store`). Server 1 invalidates a cache key but servers 2-5 still serve stale data. What are your options?

---

**Scenario 10:** You have a `Product#show` page that renders the same HTML for all users (guest and authenticated), except logged-in users see a "Buy Now" button and their personal price tier. How do you cache this page without leaking the personalized sections?

---

**Scenario 11:** Your mobile API returns a JSON response that is identical for all users at the same point in time (e.g., public product catalog). You want to cache it at the CDN level to reduce origin hits. What HTTP headers do you need to set and what are the pitfalls?

---

**Scenario 12:** Implement a "trending posts" feature that shows the top 10 posts by view count in the last 24 hours. The list should update every 5 minutes, and calculating it requires a slow query. Design the caching architecture including cache warming and staleness handling.

---
