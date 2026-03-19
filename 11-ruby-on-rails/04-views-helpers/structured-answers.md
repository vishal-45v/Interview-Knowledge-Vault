# Chapter 04 — Views & Helpers: Structured Answers

---

## Answer 1: Fragment Caching with Russian Doll Pattern

**Question:** Implement proper Russian doll caching for a blog post list where posts have comments and each comment has an author.

```erb
<%# app/views/posts/index.html.erb %>
<%# Outer cache: the entire list, keyed by the most-recently updated post %>
<% cache ["posts-index-v2", @posts.map(&:cache_key_with_version).join(",")] do %>
  <div class="post-list">
    <% @posts.each do |post| %>
      <%# Middle cache: each individual post %>
      <% cache post do %>
        <article class="post-card">
          <h2><%= link_to post.title, post_path(post) %></h2>
          <p class="meta">By <%= post.author.name %> · <%= time_ago_in_words(post.published_at) %> ago</p>

          <%# Inner cache: the comments section (keyed to latest comment) %>
          <% cache ["post-comments", post, post.comments.maximum(:updated_at)] do %>
            <div class="comments-preview">
              <% post.comments.approved.recent.limit(3).each do |comment| %>
                <%# Innermost cache: each comment %>
                <% cache comment do %>
                  <div class="comment-preview">
                    <strong><%= comment.author.name %>:</strong>
                    <%= truncate(comment.body, length: 80) %>
                  </div>
                <% end %>
              <% end %>
              <% if post.comments.count > 3 %>
                <p><%= link_to "View all #{post.comments.count} comments", post_path(post, anchor: "comments") %></p>
              <% end %>
            </div>
          <% end %>
        </article>
      <% end %>
    <% end %>
  </div>
<% end %>
```

```ruby
# Models with touch chain for cache invalidation:
class Comment < ApplicationRecord
  belongs_to :post, touch: true      # comment saved → post.updated_at changes
  belongs_to :author, class_name: "User"

  scope :approved, -> { where(approved: true) }
  scope :recent,   -> { order(created_at: :desc) }
end

class Post < ApplicationRecord
  belongs_to :author, class_name: "User"
  has_many :comments

  # Ensure cache_key includes schema version prefix:
  def cache_key_with_version
    "#{super}/v2"  # bump when template changes
  end
end

# Controller — single query with all associations:
def index
  @posts = Post.published
               .includes(:author, comments: :author)
               .order(published_at: :desc)
               .limit(15)
end
```

---

## Answer 2: Secure XSS Prevention Patterns

**Question:** Show a complete set of XSS prevention patterns for user-submitted content.

```erb
<%# Pattern 1: Plain text output (default — always safe) %>
<p><%= @user.bio %></p>
<%# ERB auto-escapes: < → &lt;, > → &gt;, " → &quot;, ' → &#39;, & → &amp; %>

<%# Pattern 2: Sanitized HTML (trusted formatting, untrusted users) %>
<%= sanitize @user.bio,
             tags: %w[p br b strong i em ul ol li a blockquote code pre],
             attributes: %w[href class target rel] %>
<%# Strips: <script>, onclick, <iframe>, style=, javascript: hrefs %>

<%# Pattern 3: Markdown → HTML (rendered safely) %>
<%= sanitize(
  Redcarpet::Markdown.new(
    Redcarpet::Render::HTML.new(safe_links_only: true, no_images: true)
  ).render(@user.bio)
) %>

<%# Pattern 4: ActionText (Rails built-in rich text, sanitized by Trix editor) %>
<%= @post.body %>  <%# ActionText has_rich_text automatically sanitizes %>

<%# Pattern 5: JSON data for JavaScript (prevent JS injection) %>
<script>
  const config = <%= raw @config.to_json %>;  <%# BAD: JS injection possible if config has </script> %>
  const config = JSON.parse('<%= j @config.to_json %>'); <%# BAD: double-encoding %>

  <%# CORRECT: use data attributes or a JSON script tag %>
</script>

<%# Safe JSON injection: %>
<div id="app" data-config="<%= @config.to_json %>"></div>
<%# JavaScript reads: JSON.parse(document.getElementById('app').dataset.config) %>

<%# Or: type="application/json" script tag (safe, not executed as JS): %>
<script id="initial-data" type="application/json">
  <%= @initial_data.to_json.html_safe %>
</script>
<%# Note: .html_safe is safe here because <script type="application/json"> is not executed %>
<%# But ensure no </script> in the JSON. Rails.application.config.content_security_policy handles this %>
```

---

## Answer 3: Custom Form Builder

**Question:** Create a custom FormBuilder that adds Bootstrap classes and error handling automatically.

```ruby
# app/form_builders/bootstrap_form_builder.rb
class BootstrapFormBuilder < ActionView::Helpers::FormBuilder
  delegate :content_tag, :tag, to: :@template

  def text_field(attribute, options = {})
    wrap_field(attribute, options) do |opts|
      super(attribute, opts)
    end
  end

  def email_field(attribute, options = {})
    wrap_field(attribute, options) do |opts|
      super(attribute, opts)
    end
  end

  def password_field(attribute, options = {})
    wrap_field(attribute, options) do |opts|
      super(attribute, opts)
    end
  end

  def text_area(attribute, options = {})
    wrap_field(attribute, options) do |opts|
      super(attribute, opts)
    end
  end

  def select(attribute, choices, options = {}, html_options = {})
    html_options[:class] = class_with_error(attribute, html_options[:class] || "form-select")
    wrap_field(attribute, options) do |_|
      super(attribute, choices, options, html_options)
    end
  end

  def submit(value = nil, options = {})
    options[:class] ||= "btn btn-primary"
    super(value, options)
  end

  private

  def wrap_field(attribute, options = {})
    options[:class] = class_with_error(attribute, options[:class] || "form-control")
    field_html = yield(options)

    content_tag(:div, class: "mb-3") do
      label(attribute, options.delete(:label), class: "form-label") +
      field_html +
      error_messages(attribute)
    end
  end

  def class_with_error(attribute, base_class)
    if has_error?(attribute)
      "#{base_class} is-invalid"
    else
      base_class
    end
  end

  def has_error?(attribute)
    object.respond_to?(:errors) && object.errors[attribute].any?
  end

  def error_messages(attribute)
    return "".html_safe unless has_error?(attribute)

    messages = object.errors[attribute].map { |msg|
      content_tag(:div, msg, class: "invalid-feedback")
    }
    safe_join(messages)
  end
end

# Helper to use it:
module ApplicationHelper
  def bootstrap_form_with(model:, **options, &block)
    options[:builder] = BootstrapFormBuilder
    form_with(model: model, **options, &block)
  end
end

# Usage:
# <%= bootstrap_form_with model: @user do |f| %>
#   <%= f.text_field :name %>
#   <%= f.email_field :email %>
#   <%= f.submit "Save" %>
# <% end %>
```

---

## Answer 4: ViewComponent with Slots and Variants

**Question:** Build a Card component with slots for header, body, footer, and variants for different styles.

```ruby
# app/components/card_component.rb
class CardComponent < ViewComponent::Base
  renders_one  :header
  renders_one  :footer
  renders_many :actions

  VARIANTS = %w[default primary success warning danger].freeze

  def initialize(variant: :default, elevated: false, padding: :default)
    @variant  = variant.to_s.in?(VARIANTS) ? variant.to_s : "default"
    @elevated = elevated
    @padding  = padding
  end

  def css_classes
    classes = ["card", "card--#{@variant}"]
    classes << "card--elevated" if @elevated
    classes << "card--#{@padding}-padding"
    classes.join(" ")
  end
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="<%= css_classes %>">
  <% if header? %>
    <div class="card__header">
      <%= header %>
    </div>
  <% end %>

  <div class="card__body">
    <%= content %>
  </div>

  <% if actions? %>
    <div class="card__actions">
      <% actions.each do |action| %>
        <%= action %>
      <% end %>
    </div>
  <% end %>

  <% if footer? %>
    <div class="card__footer">
      <%= footer %>
    </div>
  <% end %>
</div>

<%# Usage: %>
<%# <%= render CardComponent.new(variant: :primary, elevated: true) do |card| %>
<%#   <% card.with_header { tag.h2("User Profile") } %>
<%#   <p>Name: <%= @user.name %></p>
<%#   <p>Email: <%= @user.email %></p>
<%#   <% card.with_action { link_to "Edit", edit_user_path(@user), class: "btn" } %>
<%#   <% card.with_action { button_to "Delete", user_path(@user), method: :delete, class: "btn btn-danger" } %>
<%#   <% card.with_footer { tag.small("Last updated #{time_ago_in_words(@user.updated_at)} ago") } %>
<%# <% end %> %>
```

---

## Answer 5: Turbo Stream Real-Time Notifications

**Question:** Implement a real-time notification bell that updates without page refresh using Turbo Streams and ActionCable.

```ruby
# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end
end

# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :notifiable, polymorphic: true

  scope :unread, -> { where(read_at: nil) }

  after_create_commit :broadcast_to_user

  private

  def broadcast_to_user
    NotificationsChannel.broadcast_to(
      user,
      turbo_stream: render_to_string(
        partial: "notifications/notification",
        locals: { notification: self }
      )
    )
    # Also update the badge count:
    NotificationsChannel.broadcast_to(
      user,
      turbo_stream: turbo_stream_action_tag(
        "update",
        target: "notification-count",
        content: user.notifications.unread.count.to_s
      )
    )
  end
end

# app/controllers/notifications_controller.rb
class NotificationsController < ApplicationController
  def index
    @notifications = current_user.notifications.order(created_at: :desc).page(params[:page])
    @unread_count = current_user.notifications.unread.count
  end

  def mark_read
    @notification = current_user.notifications.find(params[:id])
    @notification.update!(read_at: Time.current)
    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.replace(@notification),
          turbo_stream.update("notification-count", current_user.notifications.unread.count)
        ]
      end
    end
  end

  def mark_all_read
    current_user.notifications.unread.update_all(read_at: Time.current)
    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.update("notification-count", "0"),
          turbo_stream.update("notifications-list", partial: "notifications/empty")
        ]
      end
    end
  end
end
```

```erb
<%# app/views/layouts/application.html.erb — notification bell %>
<div class="notification-bell" data-controller="notifications">
  <%= turbo_stream_from current_user, channel: "NotificationsChannel" %>
  <button class="bell-button">
    🔔 <span id="notification-count" class="badge">
      <%= current_user.notifications.unread.count %>
    </span>
  </button>
  <div id="notifications-dropdown" hidden>
    <%= turbo_frame_tag "notifications-list",
                         src: notifications_path,
                         loading: "lazy" %>
  </div>
</div>
```

---

## Answer 6: Accessible Form Implementation

**Question:** Build a form that is fully accessible (ARIA labels, error announcements, focus management).

```erb
<%# app/views/users/new.html.erb %>
<main>
  <h1>Create Account</h1>

  <%# Announce errors to screen readers: %>
  <% if @user.errors.any? %>
    <div role="alert" aria-live="assertive" class="error-summary" id="error-summary">
      <h2>Please fix the following errors:</h2>
      <ul>
        <% @user.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <%= form_with model: @user, id: "registration-form",
                aria: { describedby: @user.errors.any? ? "error-summary" : nil } do |f| %>

    <%# Name field with visible label and inline error %>
    <div class="field" role="group">
      <%= f.label :name, "Full Name", class: "field__label" %>
      <%= f.text_field :name,
                       required: true,
                       autocomplete: "name",
                       "aria-required": "true",
                       "aria-invalid": @user.errors[:name].any? ? "true" : nil,
                       "aria-describedby": @user.errors[:name].any? ? "name-error" : "name-hint",
                       class: "field__input #{'field__input--error' if @user.errors[:name].any?}" %>
      <% if @user.errors[:name].any? %>
        <p id="name-error" class="field__error" role="alert">
          <%= @user.errors[:name].first %>
        </p>
      <% else %>
        <p id="name-hint" class="field__hint">
          Enter your full legal name as it appears on your ID.
        </p>
      <% end %>
    </div>

    <%# Password with strength indicator %>
    <div class="field" data-controller="password-strength">
      <%= f.label :password, "Password" %>
      <%= f.password_field :password,
                           required: true,
                           minlength: 8,
                           "aria-describedby": "password-strength password-requirements",
                           data: { action: "input->password-strength#check",
                                   "password-strength-target": "input" } %>
      <div id="password-requirements" class="sr-only">
        Password must be at least 8 characters and contain a number and uppercase letter.
      </div>
      <div id="password-strength" role="status" aria-live="polite"
           data-password-strength-target="indicator">
      </div>
    </div>

    <%# Submit with explicit disabled state messaging %>
    <button type="submit" class="btn btn--primary"
            aria-label="Create your account">
      Create Account
    </button>

  <% end %>
</main>
```

---

## Answer 7: Hotwire Architecture for a SaaS Dashboard

**Question:** Describe how you'd architect a real-time SaaS dashboard using Hotwire, with live metrics, notifications, and in-place editing.

```erb
<%# app/views/dashboard/show.html.erb %>

<%# 1. Subscribe to multiple real-time streams %>
<%= turbo_stream_from "#{current_account.id}_metrics" %>
<%= turbo_stream_from current_user %>

<%# 2. Main metrics (auto-refreshing) %>
<section id="metrics-section">
  <div id="metrics-grid">
    <%= render "dashboard/metrics", metrics: @metrics %>
  </div>
</section>

<%# 3. Activity feed (append-only stream) %>
<section>
  <h2>Recent Activity</h2>
  <div id="activity-feed">
    <%= render @activities %>
  </div>
</section>

<%# 4. Inline editing with Turbo Frames %>
<section>
  <h2>Account Settings</h2>
  <turbo-frame id="account-settings">
    <%= render "dashboard/account_info", account: current_account %>
  </turbo-frame>
  <%# Clicking "Edit" loads the edit form into this frame: %>
  <%# No full page reload — just this section updates %>
</section>

<%# 5. Notification panel %>
<aside id="notification-panel">
  <%= turbo_frame_tag "notifications", src: notifications_path, loading: "lazy" %>
</aside>
```

```ruby
# app/jobs/broadcast_metrics_job.rb
class BroadcastMetricsJob < ApplicationJob
  queue_as :real_time

  def perform(account_id)
    account = Account.find(account_id)
    metrics = MetricsCalculator.new(account).current_metrics

    Turbo::StreamsChannel.broadcast_update_to(
      "#{account_id}_metrics",
      target: "metrics-grid",
      partial: "dashboard/metrics",
      locals: { metrics: metrics }
    )
  end
end

# Scheduled via ActiveJob:
# BroadcastMetricsJob.set(wait: 30.seconds).perform_later(account.id)
# Or use sidekiq-cron for recurring broadcasts
```

---

## Answer 8: I18n in Views — Complete Implementation

**Question:** Implement full internationalization support in views with locale switching.

```ruby
# config/application.rb
config.i18n.default_locale = :en
config.i18n.available_locales = [:en, :es, :fr, :de, :ja]
config.i18n.fallbacks = true  # Falls back to :en if key missing in locale

# config/routes.rb
scope "(:locale)", locale: /#{I18n.available_locales.join("|")}/ do
  resources :articles
  root to: 'home#index'
end

# ApplicationController
class ApplicationController < ActionController::Base
  before_action :set_locale

  private

  def set_locale
    I18n.locale = extract_locale || I18n.default_locale
  end

  def extract_locale
    # Priority: URL param > user preference > Accept-Language > default
    locale_from_param   = params[:locale]&.to_sym
    locale_from_user    = current_user&.locale&.to_sym
    locale_from_header  = extract_from_accept_language_header

    [locale_from_param, locale_from_user, locale_from_header]
      .find { |l| I18n.available_locales.include?(l) }
  end

  def extract_from_accept_language_header
    request.env['HTTP_ACCEPT_LANGUAGE']&.scan(/[a-z]{2}(?=-[A-Z]|,|;|$)/)
            &.map(&:to_sym)
            &.find { |l| I18n.available_locales.include?(l) }
  end

  def default_url_options
    { locale: I18n.locale }  # Append locale to all URL helpers
  end
end
```

```erb
<%# config/locales/en.yml %>
<%# en:
#   nav:
#     home: "Home"
#     products: "Products"
#   articles:
#     index:
#       title: "Articles"
#       count:
#         one: "1 article"
#         other: "%{count} articles"
#     show:
#       by: "By %{author}" %>

<%# In view — implicit scope via t('.'): %>
<%# When in articles/index.html.erb, t('.title') = t('articles.index.title') %>
<h1><%= t('.title') %></h1>
<p><%= t('.count', count: @articles.size) %></p>
<p><%= t('articles.show.by', author: @article.author.name) %></p>

<%# Locale switcher partial: %>
<nav class="locale-switcher">
  <% I18n.available_locales.each do |locale| %>
    <%= link_to locale.to_s.upcase,
                url_for(locale: locale),
                class: "locale-link #{'active' if I18n.locale == locale}",
                hreflang: locale %>
  <% end %>
</nav>
```

