# Chapter 01 — Rails MVC Architecture: Diagram Explanations

---

## Diagram 1: Rails Request Lifecycle (Complete)

```
Browser (HTTP Client)
        │
        │  GET /users/1 HTTP/1.1
        │  Host: example.com
        │  Cookie: _session_id=abc...
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    PUMA Web Server                          │
│  Thread Pool: [Worker1] [Worker2] [Worker3] [Worker4]       │
│  Parses HTTP → builds Rack env hash                         │
└──────────────────────┬──────────────────────────────────────┘
                       │  env = { 'REQUEST_METHOD' => 'GET',
                       │          'PATH_INFO' => '/users/1',
                       │          'HTTP_COOKIE' => '_session_id=abc...',
                       │          'rack.input' => #<StringIO>, ... }
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  RACK MIDDLEWARE STACK                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ActionDispatch::HostAuthorization                    │   │
│  │   Blocks requests from non-allowed hosts             │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::SSL (if force_ssl: true)             │   │
│  │   Redirects HTTP → HTTPS                             │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::Static                               │   │
│  │   Check: does /users/1 match a file in public/?      │   │
│  │   If yes → serve file, skip rest of stack            │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │  (no static file found)          │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ Rack::Runtime                                        │   │
│  │   Records start time → adds X-Runtime header        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::RequestId                            │   │
│  │   Generates/reads X-Request-Id header                │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::Cookies                              │   │
│  │   Parses Cookie header → env['rack.cookies']         │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::Session::CookieStore                 │   │
│  │   Decrypts + deserializes session cookie             │   │
│  │   Stores in env['rack.session']                      │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ ActionDispatch::Flash                                │   │
│  │   Reads flash from session, makes it available       │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │ Rack::MethodOverride                                 │   │
│  │   Checks for _method param → overrides REQUEST_METHOD│   │
│  └──────────────────────┬──────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                ACTION DISPATCH ROUTER                       │
│                                                             │
│  Routes table (compiled from config/routes.rb):             │
│  GET  /users       → users#index                            │
│  POST /users       → users#create                           │
│  GET  /users/:id   → users#show      ← MATCH               │
│  ...                                                        │
│                                                             │
│  Extracts: { controller: 'users', action: 'show', id: '1' } │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                CONTROLLER PHASE                             │
│                                                             │
│  1. Instantiate UsersController                             │
│  2. Merge params: { id: '1', controller: 'users',           │
│                     action: 'show' }                        │
│  3. Run before_action chain:                                │
│     → authenticate_user!  (checks session)                  │
│     → set_locale           (i18n)                           │
│  4. Execute action: users#show                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  MODEL / DATABASE PHASE                     │
│                                                             │
│  @user = User.find(1)                                       │
│         ↓                                                   │
│  ActiveRecord builds SQL:                                   │
│  SELECT "users".* FROM "users" WHERE "users"."id" = 1       │
│         ↓                                                   │
│  Connection Pool: checkout connection                        │
│         ↓                                                   │
│  PostgreSQL executes query → returns row                    │
│         ↓                                                   │
│  ActiveRecord instantiates User object                      │
│         ↓                                                   │
│  Connection Pool: checkin connection                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   VIEW / RENDER PHASE                       │
│                                                             │
│  1. Determine template: app/views/users/show.html.erb       │
│  2. Determine layout: app/views/layouts/application.html.erb│
│  3. Check fragment cache for any cache blocks               │
│  4. Evaluate ERB: @user.name, @user.email, etc.             │
│  5. Render partials (e.g., _user_card.html.erb)             │
│  6. Wrap content in layout                                  │
│  7. Return HTML string                                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               RESPONSE ASSEMBLY                             │
│                                                             │
│  Status: 200                                                │
│  Headers: Content-Type: text/html; charset=utf-8            │
│           X-Request-Id: abc-123-def                         │
│           X-Runtime: 0.045                                  │
│           Set-Cookie: _session_id=...; HttpOnly; SameSite   │
│  Body: <html>...</html>                                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼ (middleware stack unwinds)
                  Puma writes HTTP response
                       │
                       ▼
                    Browser
```

---

## Diagram 2: Rack Middleware Stack — Innermost to Outermost

```
REQUEST enters at top, flows DOWN
RESPONSE returns from bottom, flows UP

  ┌─────────────────────────────────────────────┐
  │  Rack::Runtime         (outermost — first)  │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::RequestId                  │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::RemoteIp                   │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::Static                     │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::Cookies                    │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::Session::CookieStore        │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::Flash                      │
  ├─────────────────────────────────────────────┤
  │  Rack::MethodOverride                       │
  ├─────────────────────────────────────────────┤
  │  ActionDispatch::Routing::RouteSet           │
  ├─────────────────────────────────────────────┤
  │  ActionController::Base (your controller)   │  ← innermost
  └─────────────────────────────────────────────┘

Call direction: env travels DOWN (each middleware calls @app.call(env))
Return direction: [status, headers, body] travels UP

Each middleware can:
  ✓ Inspect env
  ✓ Modify env before passing down
  ✓ Modify [status, headers, body] before returning up
  ✓ Short-circuit: return response WITHOUT calling @app.call
  ✓ Wrap in begin/rescue for error handling
```

---

## Diagram 3: Zeitwerk File-to-Constant Mapping

```
app/
├── models/
│   ├── user.rb                 →  User
│   ├── admin_user.rb           →  AdminUser
│   └── admin/
│       ├── user.rb             →  Admin::User
│       └── dashboard_report.rb →  Admin::DashboardReport
├── controllers/
│   ├── application_controller.rb  →  ApplicationController
│   ├── users_controller.rb        →  UsersController
│   └── api/
│       └── v1/
│           └── users_controller.rb →  Api::V1::UsersController
├── services/
│   ├── checkout_service.rb    →  CheckoutService
│   └── pdf/
│       └── invoice_builder.rb →  Pdf::InvoiceBuilder
└── jobs/
    └── send_welcome_email_job.rb →  SendWelcomeEmailJob

TRANSLATION RULE:
  directory/file_name.rb → Module::ClassName
  underscore_words → CamelCase
  nesting depth → Module nesting depth

ACRONYM OVERRIDE (config/initializers/inflections.rb):
  inflect.acronym 'HTML'
  inflect.acronym 'PDF'
  inflect.acronym 'API'
  → app/models/html_document.rb → HTMLDocument (not HtmlDocument)
  → app/services/pdf_generator.rb → PDFGenerator (not PdfGenerator)
  → app/controllers/api/base_controller.rb → API::BaseController

DEVELOPMENT (reloading enabled):
  File changed? → Zeitwerk unloads constant → reloads on next reference

PRODUCTION (eager_load: true):
  Boot time → Rails.application.eager_load! → requires ALL .rb files
  → All constants loaded into memory
  → Zero autoloading during request handling
```

---

## Diagram 4: Rails Boot Sequence

```
$ bundle exec rails server

     1. Bundler
        ├── Reads Gemfile.lock
        ├── Activates gem versions
        └── Requires 'rails' gem

     2. config/boot.rb
        ├── Require 'bundler/setup'
        └── Require 'bootsnap/setup' (if present)

     3. config/application.rb
        ├── require_relative 'boot'
        ├── require 'rails/all'
        ├── Bundler.require(*Rails.groups)
        └── class Application < Rails::Application
              config.* settings
            end

     4. config/environment.rb
        ├── require_relative 'application'
        └── Rails.application.initialize!
              │
              ├── Run railties initializers
              ├── Run config/initializers/*.rb (alphabetical)
              ├── Set up Zeitwerk autoload paths
              ├── Database connection pool setup
              └── Middleware stack assembly

     5. Puma starts
        ├── Forks worker processes (if cluster mode)
        ├── Starts thread pool per worker
        └── Listens on port 3000

     6. READY: First request can be handled
```

---

## Diagram 5: MVC Data Flow with Database

```
                    config/routes.rb
                         │
         GET /users/1    │  matches resources :users
              ↓          │
    ┌─────────────────┐  │
    │     ROUTER      │──┘
    │  controller:    │
    │  users          │
    │  action: show   │
    │  id: 1          │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │   CONTROLLER    │          ┌─────────────────┐
    │ UsersController │          │      MODEL      │
    │  #show          │          │  User (AR)      │
    │                 │ ────────▶│                 │
    │  before_action: │  find(1) │  validates :*   │
    │  authenticate!  │          │  belongs_to :*  │
    │                 │◀──────── │  has_many :*    │
    │  @user = User   │  User    │                 │
    │       .find(1)  │  object  └────────┬────────┘
    └────────┬────────┘                   │
             │                            │ SQL: SELECT * FROM users WHERE id=1
             │                            ▼
             │                   ┌─────────────────┐
             │                   │    DATABASE     │
             │                   │  PostgreSQL     │
             │                   │                 │
             │                   │  users table:   │
             │                   │  id|name|email  │
             │                   │  1 |Alice|a@b.c │
             │                   └─────────────────┘
             │
             ▼
    ┌─────────────────┐
    │      VIEW       │
    │  show.html.erb  │
    │                 │
    │  <%= @user.name │
    │  <%= @user.email│
    │                 │
    │  layout:        │
    │  application    │
    │  .html.erb      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │    RESPONSE     │
    │  200 OK         │
    │  Content-Type:  │
    │  text/html      │
    │                 │
    │  <html>         │
    │    <h1>Alice</h1│
    │  </html>        │
    └─────────────────┘
             │
             ▼
          Browser
```

---

## Diagram 6: Asset Pipeline Flow

```
Development Mode (config.assets.debug = true):
  Browser requests /assets/application.js
       ↓
  ActionDispatch::Static checks public/assets/ → NOT FOUND
       ↓
  Sprockets::Rails::QuietAssets middleware
       ↓
  Sprockets processes app/assets/javascripts/application.js
  (reads //= require directives, concatenates files)
       ↓
  Returns JS, NOT fingerprinted
  Served uncompressed for debugging

Production Mode (after rails assets:precompile):
  $ RAILS_ENV=production bundle exec rails assets:precompile

  app/assets/                           public/assets/
  ├── javascripts/                      ├── application-[md5].js
  │   └── application.js    ──────────▶ ├── application-[md5].js.gz
  ├── stylesheets/                      ├── application-[md5].css
  │   └── application.css   ──────────▶ ├── application-[md5].css.gz
  └── images/                           └── logo-[md5].png
      └── logo.png           ──────────▶

  Browser requests /assets/application-abc123.js
       ↓
  ActionDispatch::Static finds public/assets/application-abc123.js
       ↓
  Served directly (no Ruby processing)
  Headers: Cache-Control: public, max-age=31536000 (1 year)

  Rails view helper maps logical name → fingerprinted name:
  <%= javascript_include_tag 'application' %>
  → <script src="/assets/application-abc123.js"></script>
```

