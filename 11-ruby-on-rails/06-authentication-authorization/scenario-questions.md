# Chapter 06 — Authentication & Authorization: Scenario Questions

---

**Scenario 1:** A user reports they were logged out of your Rails app after clicking a "forgot password" link even though they didn't go through the reset process. You're using Devise. What could cause this, and how do you investigate?

---

**Scenario 2:** You're building a multi-tenant SaaS app. Each tenant has their own subdomain (e.g., `acme.myapp.com`). How do you implement authentication so users can only access their own tenant's data and can't cross tenants even with a valid session?

---

**Scenario 3:** Your Pundit policy for `PostsController#show` works correctly, but when you render `@post.comments` in the view, users can see comments on posts they shouldn't have access to. The comment policy inherits from PostPolicy. What's the bug?

---

**Scenario 4:** A penetration tester reports they can bypass your admin-only routes by manipulating the URL. You have `before_action :require_admin!` on the `Admin::BaseController`, but some admin controllers inherit from `ApplicationController` directly. How do you audit and fix this?

---

**Scenario 5:** You're implementing OmniAuth with Google. On first login, you create a user and log them in. But when an existing user who registered with email/password tries to connect their Google account, Devise raises `Email has already been taken`. How do you handle account linking?

---

**Scenario 6:** Implement a 2FA system where users are required to complete TOTP verification on every login, with a "remember this device for 30 days" option.

---

**Scenario 7:** Your API uses JWT. A security audit finds that your tokens never expire — someone intercepted a token 6 months ago and it still works. Describe the full remediation plan, including handling existing sessions.

---

**Scenario 8:** You need to implement an "impersonate user" feature for admins so they can debug user issues. Requirements: (1) Only super-admins can impersonate, (2) impersonation must be logged, (3) admin must be able to exit impersonation, (4) impersonation cannot be nested (can't impersonate while impersonating). Design and implement this.

---

**Scenario 9:** Your app uses Devise with the `:confirmable` module. A user complains they never received the confirmation email and their account is locked. Another user says they confirmed their account but can't log in. Debug both scenarios.

---

**Scenario 10:** You need to implement role-based access control where:
- Users can have multiple roles (editor, moderator, admin)
- Roles are scoped to resources (e.g., editor of magazine A but not magazine B)
- Roles can be granted and revoked
- Pundit policies must respect the role hierarchy

Design the data model and implement the Pundit integration.

---

**Scenario 11:** You're building an API that issues personal access tokens (like GitHub's PATs). Requirements: (1) Users can create multiple tokens with different scopes, (2) tokens should be shown only once at creation, (3) tokens must be revocable, (4) token usage must be logged with last-used timestamp. Implement the full system.

---

**Scenario 12:** During a security review, you discover that your password reset tokens are stored in plain text in the database. The reset link is `GET /users/password/edit?reset_password_token=<token>`. How do you fix this without breaking existing reset flows, and what else should you audit?

---
