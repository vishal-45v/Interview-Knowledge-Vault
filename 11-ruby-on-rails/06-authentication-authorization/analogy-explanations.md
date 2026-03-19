# Chapter 06 — Authentication & Authorization: Analogy Explanations

---

## Analogy 1: Authentication vs Authorization — The Nightclub Entry System

**Authentication** is the bouncer at the door checking your ID:
- "Who are you? Is this ID real?"
- Doesn't matter what you want to do inside — just verifying your identity
- Devise, `has_secure_password`, JWT verification all answer: "Are you who you claim to be?"

**Authorization** is the section control inside the nightclub:
- "You're verified as a customer, but the VIP lounge requires a VIP wristband"
- Even authenticated guests can't access every area
- Pundit, CanCanCan answer: "Now that we know who you are, what are you allowed to do?"

Two separate systems. Two separate failures:
- Authentication failure: "I don't know who you are" → 401 Unauthorized
- Authorization failure: "I know who you are, but you can't go there" → 403 Forbidden

---

## Analogy 2: bcrypt — The One-Way Meat Grinder

Storing a plaintext password is like writing your house key's pattern on a post-it note and sticking it to your door. Anyone who reads it can make a copy.

bcrypt is a one-way meat grinder:
- You put the password in → you get "ground meat" (the hash) out
- You can NEVER turn the ground meat back into the original ingredients
- To verify: you grind the input password through the same grinder settings (salt + cost factor) → if the results match, the original passwords matched
- The "cost factor" (bcrypt work factor) makes the grinder progressively slower — so grinding one password takes 100ms, and an attacker grinding millions takes centuries

SHA-256 is a fast blender, not a meat grinder — it produces output in microseconds. An attacker can try billions per second. bcrypt is intentionally slow. That's the point.

---

## Analogy 3: Pundit Policies — The HR Policy Manual

Pundit policies are like an HR policy manual for each department:

- `PostPolicy` = the HR manual for the Posts department
- `authorize @post` = "HR, check the manual: can this employee (`user`) do this action (`update?`) with this resource (`@post`)?"
- `policy_scope(Post)` = "HR, give me a list of posts this employee is allowed to see"

The `PostPolicy::Scope` is the "employee directory access filter" — it defines which rows of the spreadsheet the employee can view, not just whether they can view a specific row.

Without `policy_scope`, you hand the employee the entire HR database. With only `policy_scope`, you give them a filtered list but don't check if they can open each individual file. You need both.

---

## Analogy 4: OAuth / OmniAuth — The Hotel Key Card System

Traditional login is having your own house key — you manage it, you lose it, you replace it yourself.

OAuth is the hotel key card system:
- Google is the hotel's central key-card system
- "Sign in with Google" = "Use your hotel keycard to open this door" — you trust the hotel's security
- Your app never sees the actual key (password) — just a token the hotel says is valid
- When the hotel says "this card is valid for room 204 (your email)", you believe them

OmniAuth is the universal card reader that speaks to any hotel's key system:
- It handles the "insert card, verify with hotel, report back" sequence
- Your app just gets: "verified guest identity: name, email, profile photo"

The OAuth flow is the verification phone call: "Hotel? I have a guest presenting card 12345. Can you confirm their identity and which rooms they have access to?" The hotel sends back a "yes, and here's their info."

---

## Analogy 5: Session Fixation — The Pre-Stamped Visitor Badge

Imagine a building where visitors get stamped entry badges:

**Without session fixation protection:**
1. An attacker stamps their own blank badge with a known ID: `BADGE-12345`
2. They trick a victim into entering the building while wearing that same badge ID
3. When the victim checks in with reception (logs in), the badge ID `BADGE-12345` is now associated with the victim's identity
4. The attacker walks in using the same badge ID — they're recognized as the authenticated victim

**With session fixation protection (reset_session on login):**
1. The attacker pre-stamps badge ID `BADGE-12345`
2. Victim presents that badge at login
3. Reception DESTROYS the old badge and issues a completely new badge: `BADGE-99999`
4. The attacker's pre-stamped `BADGE-12345` is worthless — the new session ID is secret

`reset_session` in Rails is reception destroying the old badge on login.

---

## Analogy 6: TOTP Two-Factor Authentication — The Synchronized Clock Code

TOTP (Time-based One-Time Password) is like a synchronized pair of digital watches that display the same secret code:

- During 2FA setup, your app and your phone both receive the same master secret (the QR code scan)
- Every 30 seconds, both independently compute the same 6-digit code from: (master secret + current time window)
- No network communication needed — they're mathematically synchronized
- To verify: "Does your watch show the same code as my watch right now?"

Why it's secure:
- An intercepted code is useless after 30 seconds (±drift window)
- The master secret never leaves the authenticator app — only codes travel over the network
- "Drift window" (±15 seconds) accounts for clocks not being perfectly synchronized

Backup codes are sealed envelopes in a fireproof safe — a one-time emergency use when you lose your watch (phone). Each envelope is burned after use. You only get them once when setting up the safe.

---

## Analogy 7: Devise Modules — The Swiss Army Knife vs Pocket Knife

Devise is a Swiss Army knife for authentication. You don't open all the tools at once — you choose which blades to deploy:

- `:database_authenticatable` = the main knife blade (email + password login)
- `:registerable` = the can opener (user self-registration)
- `:recoverable` = the corkscrew (forgot password flow)
- `:rememberable` = the pen that doesn't retract (remember me cookie)
- `:confirmable` = the magnifying glass (must verify email before entering)
- `:lockable` = the safety lock (too many failed attempts → lock account)
- `:trackable` = the ruler (records sign_in_count, last_sign_in_at, IP)
- `:timeoutable` = the timer (auto-logout after inactivity)
- `:omniauthable` = the USB adapter (connect to external identity providers)

Using all modules when you only need three is like carrying a 50-tool Swiss army knife to spread butter. Choose what you need.

---

## Analogy 8: Personal Access Tokens — The Hotel Safe Code Card

Hotel safes have a "set your own combination" feature for each guest:
- You (the user) choose the combination (create a token) for your specific session
- You can set multiple safes in the room (multiple tokens with different purposes)
- Each safe (token) can be limited: "this safe only opens the mini-bar, not the main safe" (scopes)
- If you need to show the combination to room service (API client), they use it; but housekeeping doesn't need to know it

The combination is shown to you ONCE when you set it:
- The hotel doesn't store "4-7-2-9" — they store a hashed version they can verify but not read
- If you forget it, you don't "retrieve" it — you destroy the safe and create a new one
- The hotel keeps a log: "Safe #3 was opened at 2pm from IP 1.2.3.4" (last_used_at)

Token prefix (`pat_abc123...`) = the safe's model number — identifies it without revealing the combination.

---

## Analogy 9: Role-Based Access Control — The Building Access Card System

RBAC is a corporate building's card access system:

- **Roles** are card types: Visitor, Employee, Manager, Security, Executive
- **Permissions** are door locks: Floor 1 lobby, Floor 5 offices, Server room, Executive suite
- **Role assignment** is IT issuing you a card type when you join

The key insight: you don't program each person's card for each door — you program roles:
- "Employee cards can open all Floor 5 office doors"
- When you promote someone to Manager, you change their card type — they instantly get all Manager permissions

In Rails:
- Roles are stored in a `roles` table
- Permissions are enforced in Pundit policies: `user.role?('manager')`
- Changing a role is `user.roles << Role.find_by(name: 'manager')` — instantly effective

Without RBAC (per-user permissions): it's like programming every single door for every single employee. With 500 employees and 200 doors, that's 100,000 configurations. RBAC reduces this to "10 role types × 200 doors = 2,000 rules."

---
