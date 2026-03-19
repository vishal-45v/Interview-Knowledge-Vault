# Chapter 06 — Authentication & Authorization: Diagram Explanations

---

## Diagram 1: Devise Request Flow (Sign In)

```
Browser                    Rails Router              Devise Sessions Controller
   │                            │                            │
   │  POST /users/sign_in        │                            │
   │  { email, password }        │                            │
   │──────────────────────────▶ │                            │
   │                            │  route: devise_for :users  │
   │                            │──────────────────────────▶ │
   │                            │                            │  User.find_by(email:)
   │                            │                            │──────────▶ DB
   │                            │                            │◀──────────
   │                            │                            │  user.valid_password?(pw)
   │                            │                            │  → BCrypt::Password.new(digest) == pw
   │                            │                            │
   │                            │                            │  Warden.set_user(user)
   │                            │                            │  → session[:warden_user_user_key] = [user.id]
   │                            │                            │  reset_session (session fixation prevention)
   │                            │                            │
   │◀──────────────────────────────────────────────────────── │
   │  302 Redirect to root_path  │                            │
   │  Set-Cookie: _session_id=   │                            │

─────────────────────────────────────────────────────────────────────

SUBSEQUENT REQUEST:

Browser                    Rails              ApplicationController
   │                          │                      │
   │  GET /dashboard           │                      │
   │  Cookie: _session_id=abc  │                      │
   │──────────────────────────▶│                      │
   │                          │──────────────────────▶│
   │                          │                       │  before_action :authenticate_user!
   │                          │                       │  → Warden.authenticated?(:user)
   │                          │                       │  → reads session[:warden_user_user_key]
   │                          │                       │  → current_user = User.find(id)
   │                          │                       │
   │◀──────────────────────── ──────────────────────── │
   │  200 OK + dashboard HTML  │                      │
```

---

## Diagram 2: Pundit Authorization Lookup

```
Controller Action
       │
       ▼
  authorize @post         ← or: authorize @post, :update?
       │
       ▼
  Pundit.authorize(current_user, @post, :update?)
       │
       ▼
  Policy class lookup:
  @post.class → Post
  Post + "Policy" → PostPolicy
  PostPolicy.new(current_user, @post)
       │
       ▼
  postpolicy.update?
  → returns true or false
       │
       ├── true  → continue action
       └── false → raise Pundit::NotAuthorizedError
                   → rescue_from in ApplicationController
                   → render 403 or redirect

─────────────────────────────────────────────────────────────────────

POLICY SCOPE LOOKUP:

policy_scope(Post)
       │
       ▼
  Pundit.policy_scope!(current_user, Post)
       │
       ▼
  Post + "Policy::Scope" → PostPolicy::Scope
  PostPolicy::Scope.new(current_user, Post)
       │
       ▼
  .resolve → returns ActiveRecord::Relation
  (filtered query, not all records)
       │
       ▼
  @posts = filtered relation
  → further chaining: .page(params[:page]).includes(:user)

─────────────────────────────────────────────────────────────────────

POLICY INHERITANCE:

ApplicationPolicy
├── PostPolicy          → defines: show?, create?, update?, destroy?
│   └── DraftPolicy     → inherits + overrides create? only
├── CommentPolicy       → custom scope only
└── AdminPolicy         → all permissions return: user.admin?
```

---

## Diagram 3: OAuth / OmniAuth Flow (Google)

```
Browser              Your Rails App           Google OAuth
   │                       │                       │
   │  Click "Sign in        │                       │
   │  with Google"          │                       │
   │──────────────────────▶│                       │
   │                       │  redirect_to           │
   │                       │  google_oauth_url      │
   │◀─────────────────────── │                       │
   │  302 to Google         │                       │
   │──────────────────────────────────────────────▶│
   │                       │                       │  Google login UI
   │◀─────────────────────────────────────────────── │
   │  user enters credentials                       │
   │──────────────────────────────────────────────▶│
   │                       │                       │  verify identity
   │                       │                       │  generate auth code
   │◀─────────────────────────────────────────────── │
   │  302 to your callback URL + code=abc123       │
   │──────────────────────▶│                       │
   │                       │  exchange code for token│
   │                       │──────────────────────▶│
   │                       │◀─────────────────────── │
   │                       │  { access_token, id_token }
   │                       │                       │
   │                       │  fetch user profile    │
   │                       │──────────────────────▶│
   │                       │◀─────────────────────── │
   │                       │  { email, name, uid }  │
   │                       │                       │
   │                       │  User.from_omniauth(auth)
   │                       │  → find or create user
   │                       │  → sign_in_and_redirect(user)
   │◀─────────────────────── │                       │
   │  302 to dashboard      │                       │
```

---

## Diagram 4: TOTP 2FA Verification Flow

```
SETUP (one time):
  Server generates OTP secret → sends QR code to browser
  User scans QR with Authenticator app
  Both Server and App now share the same secret

─────────────────────────────────────────────────────────────────────

LOGIN FLOW WITH 2FA:

Browser              SessionsController     TwoFactorsController
   │                       │                       │
   │  POST /sessions        │                       │
   │  { email, password }   │                       │
   │──────────────────────▶│                       │
   │                       │  authenticate user     │
   │                       │  user.otp_required? → true
   │                       │  session[:pending_user_id] = user.id
   │◀─────────────────────── │                       │
   │  302 to /two_factor    │                       │
   │──────────────────────────────────────────────▶│
   │◀─────────────────────────────────────────────── │
   │  200 Enter OTP form    │                       │
   │                       │                       │
   │  POST /two_factor      │                       │
   │  { otp_attempt: 123456 }                       │
   │──────────────────────────────────────────────▶│
   │                       │                       │  user.verify_otp("123456")
   │                       │                       │  ROTP::TOTP.new(user.otp_secret)
   │                       │                       │  .verify("123456", drift: ±15s)
   │                       │                       │
   │                       │                       │  ✓ Valid: complete_sign_in(user)
   │                       │                       │  ✗ Invalid: render :new with alert
   │◀─────────────────────────────────────────────── │
   │  302 to dashboard (success) OR 422 (fail)     │

─────────────────────────────────────────────────────────────────────

TOTP CODE GENERATION (shared computation):

  Server side:                    Authenticator app:
  otp_secret = "JBSWY3DP..."      same otp_secret from QR scan
  current_time_window = 59xxx      same time window (synchronized clocks)
  HMAC(otp_secret, time_window)    HMAC(otp_secret, time_window)
  → "123456"                       → "123456"   ← same result, no network call
```

---

## Diagram 5: Session vs Token Authentication Comparison

```
SESSION-BASED AUTH (Devise default):
┌──────────────────────────────────────────────────────────────┐
│ Browser                                       Server          │
│   Login → POST /sessions                                      │
│   ←── Set-Cookie: _session_id=abc123                         │
│                                                               │
│   Each request: Cookie: _session_id=abc123                    │
│                         │                                     │
│                         ▼                                     │
│              session store (Redis/DB/cookie)                  │
│              → session_data = { user_id: 42 }                 │
│              → User.find(42) on every request                 │
│                                                               │
│ Pros: easy revocation, server controls sessions               │
│ Cons: requires session storage, doesn't scale to microservices│
└──────────────────────────────────────────────────────────────┘

JWT / TOKEN-BASED AUTH (API mode):
┌──────────────────────────────────────────────────────────────┐
│ Client                                        Server          │
│   Login → POST /auth/sessions                                 │
│   ←── { access_token: "eyJ...", refresh_token: "eyJ..." }    │
│                                                               │
│   Each request: Authorization: Bearer eyJ...                  │
│                         │                                     │
│                         ▼                                     │
│              JWT.decode(token, secret)                        │
│              → payload = { user_id: 42, exp: ... }            │
│              → User.find(42) if not expired                   │
│              → NO session store lookup                        │
│                                                               │
│ Pros: stateless, scales to microservices, mobile-friendly     │
│ Cons: hard to revoke, must handle refresh token rotation      │
└──────────────────────────────────────────────────────────────┘

COMPARISON:
┌────────────────────┬─────────────────────┬────────────────────┐
│ Property           │ Session             │ JWT                │
├────────────────────┼─────────────────────┼────────────────────┤
│ Storage            │ Server-side         │ Client-side        │
│ Revocation         │ Instant             │ Wait for expiry    │
│ Scalability        │ Needs shared store  │ Stateless          │
│ Mobile support     │ Awkward (cookies)   │ Natural (header)   │
│ Microservices      │ Shared session store│ Each service verifies│
│ Size               │ Small cookie        │ Larger header      │
│ CSRF risk          │ Yes (needs token)   │ No (header-based)  │
└────────────────────┴─────────────────────┴────────────────────┘
```

---
