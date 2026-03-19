# Chapter 03 — Routing & Controllers: Diagram Explanations

---

## Diagram 1: Full Routing Table for `resources :articles`

```
resources :articles do
  member     { get :preview; post :publish }
  collection { get :search; get :archived }
  resources :comments, shallow: true
end

Generated Routes:
┌─────────┬───────────────────────────────────┬───────────────────────────┬───────────────────────────┐
│ Method  │ Path                              │ Controller#Action         │ Helper                    │
├─────────┼───────────────────────────────────┼───────────────────────────┼───────────────────────────┤
│ GET     │ /articles                         │ articles#index            │ articles_path             │
│ GET     │ /articles/new                     │ articles#new              │ new_article_path          │
│ POST    │ /articles                         │ articles#create           │ articles_path             │
│ GET     │ /articles/:id                     │ articles#show             │ article_path(:id)         │
│ GET     │ /articles/:id/edit                │ articles#edit             │ edit_article_path(:id)    │
│ PATCH   │ /articles/:id                     │ articles#update           │ article_path(:id)         │
│ PUT     │ /articles/:id                     │ articles#update           │ article_path(:id)         │
│ DELETE  │ /articles/:id                     │ articles#destroy          │ article_path(:id)         │
├─────────┼───────────────────────────────────┼───────────────────────────┼───────────────────────────┤
│ GET     │ /articles/:id/preview             │ articles#preview          │ preview_article_path(:id) │
│ POST    │ /articles/:id/publish             │ articles#publish          │ publish_article_path(:id) │
├─────────┼───────────────────────────────────┼───────────────────────────┼───────────────────────────┤
│ GET     │ /articles/search                  │ articles#search           │ search_articles_path      │
│ GET     │ /articles/archived                │ articles#archived         │ archived_articles_path    │
├─────────┼───────────────────────────────────┼───────────────────────────┼───────────────────────────┤
│ GET     │ /articles/:article_id/comments    │ comments#index            │ article_comments_path     │
│ GET     │ /articles/:article_id/comments/new│ comments#new              │ new_article_comment_path  │
│ POST    │ /articles/:article_id/comments    │ comments#create           │ article_comments_path     │
│ GET     │ /comments/:id                     │ comments#show (shallow)   │ comment_path(:id)         │
│ GET     │ /comments/:id/edit                │ comments#edit (shallow)   │ edit_comment_path(:id)    │
│ PATCH   │ /comments/:id                     │ comments#update (shallow) │ comment_path(:id)         │
│ DELETE  │ /comments/:id                     │ comments#destroy (shallow)│ comment_path(:id)         │
└─────────┴───────────────────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## Diagram 2: namespace vs scope vs scope module

```
routes.rb configuration → Generated URL → Controller used → Named helper prefix

──────────────────────────────────────────────────────────────────────────────────

namespace :admin do                  route: /admin/users
  resources :users                   controller: Admin::UsersController
end                                  helper: admin_users_path
                                                  ↑ prefix

──────────────────────────────────────────────────────────────────────────────────

scope :admin do                      route: /admin/users
  resources :users                   controller: UsersController    ← no module!
end                                  helper: users_path             ← no prefix!

──────────────────────────────────────────────────────────────────────────────────

scope module: :admin do              route: /users
  resources :users                   controller: Admin::UsersController ← has module!
end                                  helper: users_path             ← no prefix

──────────────────────────────────────────────────────────────────────────────────

scope '/api/v1', module: :api_v1,    route: /api/v1/users
      as: :api_v1 do                 controller: ApiV1::UsersController
  resources :users                   helper: api_v1_users_path
end

──────────────────────────────────────────────────────────────────────────────────

Summary matrix:
             path prefix │ module prefix │ helper prefix
namespace      ✓         │      ✓        │      ✓
scope(path)    ✓         │      ✗        │      ✗
scope(module)  ✗         │      ✓        │      ✗
scope(as)      ✗         │      ✗        │      ✓
(combine them for full control)
```

---

## Diagram 3: Controller Action Filter Chain

```
REQUEST: POST /articles

  ApplicationController filters (run first — inherited):
  ┌──────────────────────────────────────────────────────────┐
  │  1. prepend_before_action (runs FIRST)                   │
  │     └─ set_request_id                                    │
  │  2. before_action :set_locale                            │
  │  3. before_action :authenticate_user!                    │
  │     └─ if fails: redirect_to login → CHAIN STOPS        │
  └──────────────────────────────────┬───────────────────────┘
                                     │ (authenticated)
  ArticlesController filters:
  ┌──────────────────────────────────▼───────────────────────┐
  │  4. before_action :track_analytics                       │
  │  5. around_action :set_current_tenant                    │
  │     └─ runs setup code                                   │
  └──────────────────────────────────┬───────────────────────┘
                                     │
  ┌──────────────────────────────────▼───────────────────────┐
  │  6. ACTION: articles#create                              │
  │     @article = Article.new(article_params)               │
  │     @article.save                                        │
  │     redirect_to @article                                 │
  └──────────────────────────────────┬───────────────────────┘
                                     │
  around_action completes:
  ┌──────────────────────────────────▼───────────────────────┐
  │  7. around_action :set_current_tenant (after yield)      │
  └──────────────────────────────────┬───────────────────────┘
                                     │
  ┌──────────────────────────────────▼───────────────────────┐
  │  8. after_action :log_action                             │
  │  9. after_action :cleanup                                │
  └──────────────────────────────────────────────────────────┘

  FILTER CHAIN ABORT:
  If before_action calls redirect_to or render:
  → Marks response as committed
  → All subsequent before_actions are skipped
  → Action itself is NOT called
  → around_action post-yield code still runs if it was entered
  → after_actions still run
```

---

## Diagram 4: Strong Parameters Flow

```
HTTP POST /users
Body: {
  "user": {
    "name": "Alice",
    "email": "alice@example.com",
    "password": "secret",
    "role": "admin",       ← attacker trying to escalate
    "stripe_id": "cus_xxx" ← attacker trying to set billing
  }
}

     ↓
ActionController::Parameters
(not a plain Hash — has permitted? flag)
     ↓
params.require(:user)
→ Returns sub-parameters for :user key
→ Raises ActionController::ParameterMissing if :user absent
     ↓
.permit(:name, :email, :password)
→ Creates new Parameters with ONLY permitted keys
→ Unpermitted keys are logged as warnings

Permitted:
  {
    "name" => "Alice",
    "email" => "alice@example.com",
    "password" => "secret"
  }

DROPPED (silently, with log warning):
  "role" => "admin"     ← security: ignored
  "stripe_id" => "cus_xxx" ← security: ignored

     ↓
User.new(user_params)
→ Only sets :name, :email, :password
→ role stays at default ("user")
→ stripe_id stays nil
```

---

## Diagram 5: Redirect vs Render — HTTP Flow

```
RENDER :new (re-display the form with errors):

  Browser                    Rails Server
     │                           │
     │──POST /users ────────────▶│
     │   (form data)             │
     │                           │ @user = User.new(params)
     │                           │ @user.save → returns false
     │                           │ render :new (status 422)
     │◀──422 Unprocessable ──────│
     │   (HTML with error form)  │
     │                           │
  URL still: /users (POST URL)
  Form has user's data pre-filled
  Validation errors shown


REDIRECT_TO @user (PRG pattern — successful create):

  Browser                    Rails Server
     │                           │
     │──POST /users ────────────▶│
     │   (form data)             │
     │                           │ @user = User.new(params)
     │                           │ @user.save → returns true
     │                           │ redirect_to @user (status 302)
     │◀──302 Found ──────────────│
     │   Location: /users/42     │
     │                           │
     │──GET /users/42 ──────────▶│
     │                           │ render :show
     │◀──200 OK ─────────────────│
     │   (user show page)        │
  URL now: /users/42 (GET URL)
  Refreshing won't re-POST
  flash[:notice] from previous request shown
```

---

## Diagram 6: Cookie vs Session Security Properties

```
Comparison of storage options for user identity:

┌──────────────────┬────────────┬────────────┬──────────────┬──────────────────────┐
│ Storage Type     │ Readable   │ Tamperable │ JS Access    │ Size Limit           │
│                  │ by User?   │ by User?   │ (XSS risk)   │                      │
├──────────────────┼────────────┼────────────┼──────────────┼──────────────────────┤
│ Plain cookie     │ YES        │ YES        │ YES          │ 4KB                  │
│ Signed cookie    │ YES        │ NO         │ YES (if set) │ 4KB                  │
│ Encrypted cookie │ NO         │ NO         │ NO (httponly)│ 4KB                  │
│ CookieStore sess │ NO         │ NO         │ NO (httponly)│ ~4KB (serialized)    │
│ Redis session    │ NO (server)│ NO         │ NO           │ Unlimited            │
│ DB session       │ NO (server)│ NO         │ NO           │ Unlimited            │
└──────────────────┴────────────┴────────────┴──────────────┴──────────────────────┘

Cookie security attributes:
  HttpOnly:   Prevents JavaScript access → stops XSS token theft
  Secure:     HTTPS only → stops HTTP interception (man-in-the-middle)
  SameSite=Lax:    Sent on top-level navigations from other sites → reduces CSRF
  SameSite=Strict: Never sent cross-site → strongest CSRF protection
  Expires:    Max-Age determines session vs persistent

Rails default session cookie:
  _app_session  HttpOnly=true, Secure=true(prod), SameSite=Lax, Encrypted
```

