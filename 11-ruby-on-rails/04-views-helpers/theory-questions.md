# Chapter 04 — Views & Helpers: Theory Questions

---

**Q1. What is ERB and how does it work? What are the three ERB tags?**

ERB (Embedded Ruby) is a templating system that lets you embed Ruby code in any text file.

```erb
<%  %> — Execute Ruby code (no output)
<%= %> — Execute Ruby and output the ESCAPED result (HTML-safe by default)
<%- %> — Execute, strip leading whitespace from this line
<%-= %> — Execute, output, strip whitespace
<%# %> — Comment (not output)
<% if user.admin? %>
  <p>Admin panel: <%= link_to "Settings", admin_settings_path %></p>
<% end %>
```

---

**Q2. What is the difference between layouts, templates, and partials?**

```
Layout: wrapper for the entire page (app/views/layouts/application.html.erb)
  └── Contains: <html>, <head>, <body>, navigation, footer
  └── Uses <%= yield %> to inject the template content

Template: the main view for an action (app/views/articles/show.html.erb)
  └── Rendered by the action (articles#show → show.html.erb)
  └── Injected into the layout via yield

Partial: reusable fragment (app/views/articles/_article.html.erb)
  └── Named with underscore prefix
  └── Rendered with: render @article, render 'articles/article', render partial: 'form'
  └── Can accept locals: render 'form', locals: { article: @article }
```

---

**Q3. Explain `form_with`. What does it generate? How does it handle new vs existing records?**

```erb
<%# New record — action="POST /articles" %>
<%= form_with model: Article.new do |f| %>
  <%= f.text_field :title %>
<% end %>
<!-- <form action="/articles" method="post"> -->

<%# Existing record — action="PATCH /articles/1" + hidden _method field %>
<%= form_with model: @article do |f| %>
  <%= f.text_field :title %>
<% end %>
<!-- <form action="/articles/1" method="post">
     <input type="hidden" name="_method" value="patch"> -->

<%# Without model (custom action): %>
<%= form_with url: search_path, method: :get do |f| %>
  <%= f.text_field :q, placeholder: "Search..." %>
<% end %>

<%# In Rails 7, data-turbo="false" disables Turbo for this form: %>
<%= form_with model: @article, data: { turbo: false } do |f| %>
```

---

**Q4. What is `content_for` and `yield` in layouts? How do they communicate between templates and layouts?**

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
  <%= yield :head %>   <%# for stylesheets/scripts specific to this page %>
</head>
<body>
  <div class="sidebar">
    <%= yield :sidebar %>
  </div>
  <div class="main">
    <%= yield %>   <%# main template content here %>
  </div>
</body>
</html>

<%# app/views/articles/show.html.erb %>
<% content_for :title do %>
  <%= @article.title %> - MyApp
<% end %>

<% content_for :head do %>
  <%= stylesheet_link_tag "article-reader" %>
<% end %>

<% content_for :sidebar do %>
  <%= render 'related_articles', articles: @related %>
<% end %>

<h1><%= @article.title %></h1>
<div><%= @article.body %></div>
```

---

**Q5. What is the XSS vulnerability in Rails views? How do `html_safe`, `raw`, and `sanitize` differ?**

```erb
<%# Rails auto-escapes by default — SAFE %>
<%= @user.name %>
<%# If name = "<script>alert('xss')</script>"
    Output: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt; — harmless %>

<%# html_safe marks a string as safe — SKIPS escaping — DANGEROUS with user input %>
<%= @user.bio.html_safe %>
<%# If bio contains <script>, it executes in browser! NEVER use with user input %>

<%# raw is the same as html_safe — outputs without escaping — DANGEROUS %>
<%= raw @user.bio %>

<%# sanitize: allows safe HTML tags, strips dangerous ones — SAFE %>
<%= sanitize @user.bio, tags: %w[p br strong em a], attributes: %w[href class] %>
<%# strips <script>, onclick, etc. but keeps <p>, <strong>, <a href> %>

<%# Safe way to include HTML from trusted source (you wrote it): %>
content = "<strong>Welcome</strong>".html_safe
<%= content %>  <%# displays bold text, not escaped %>
```

---

**Q6. What are ViewHelpers? How do you create a custom helper? What is the difference between a helper method and a Presenter/Decorator?**

```ruby
# app/helpers/articles_helper.rb — automatically included in articles views
module ArticlesHelper
  def article_status_badge(article)
    css_class = article.published? ? "badge-success" : "badge-warning"
    label = article.published? ? "Published" : "Draft"
    content_tag(:span, label, class: "badge #{css_class}")
  end

  def formatted_date(date)
    return "N/A" if date.nil?
    date.strftime("%B %d, %Y")
  end
end

# Usage in view:
# <%= article_status_badge(@article) %>

# Helper vs Presenter:
# Helper: stateless method in a module — good for simple formatting
# Presenter/Decorator (Draper gem or custom):
class ArticlePresenter < SimpleDelegator
  def status_badge
    css = published? ? "badge-success" : "badge-warning"
    ActionController::Base.helpers.content_tag(:span, status_label, class: "badge #{css}")
  end

  def status_label
    published? ? "Published" : "Draft"
  end
end
# @article_presenter = ArticlePresenter.new(@article)
# @article_presenter.status_badge
# @article_presenter.title (delegates to @article.title)
```

---

**Q7. What is `link_to`, `button_to`, and when do you use each?**

```erb
<%# link_to generates <a href> tags — GET requests %>
<%= link_to "View Article", article_path(@article) %>
<%= link_to "Visit Google", "https://google.com", target: "_blank", rel: "noopener" %>
<%= link_to article_path(@article), class: "card-link" do %>
  <h3><%= @article.title %></h3>
<% end %>

<%# button_to generates <form method="post"> — for non-GET actions %>
<%= button_to "Delete", article_path(@article), method: :delete,
              data: { confirm: "Are you sure?" },
              class: "btn btn-danger" %>
<%# Generates: <form action="/articles/1" method="post">
               <input type="hidden" name="_method" value="delete">
               <input type="hidden" name="authenticity_token" value="...">
               <button type="submit">Delete</button>
            </form> %>

<%# NEVER use link_to with method: :delete (deprecated in Rails 7) %>
<%# Always use button_to for state-changing actions %>
```

---

**Q8. What is `image_tag`, `asset_path`, and `asset_url`? How do they interact with the asset pipeline?**

```erb
<%# image_tag generates <img> with fingerprinted path %>
<%= image_tag "logo.png" %>
<%# → <img src="/assets/logo-abc123.png" alt="Logo"> %>

<%= image_tag @user.avatar_url, alt: @user.name, class: "avatar", size: "50x50" %>
<%# size: "50x50" → width="50" height="50" %>

<%# asset_path: returns path to asset (relative) %>
<%= asset_path "application.css" %>
# => "/assets/application-abc123.css"

<%# asset_url: returns full URL (for emails, CDN) %>
<%= asset_url "logo.png" %>
# => "https://cdn.example.com/assets/logo-abc123.png"

<%# In CSS (Sprockets): %>
/* app/assets/stylesheets/application.css */
.header { background-image: url(<%= asset_path "header-bg.jpg" %>); }
/* Or with SCSS helper: */
.header { background-image: asset-url("header-bg.jpg"); }
```

---

**Q9. What is ViewComponent? Why was it created and how does it differ from partials?**

```ruby
# Partials: ERB templates with instance variables from the controller
# ViewComponent: Ruby objects with their own template and test

# gem 'view_component'

# app/components/article_card_component.rb
class ArticleCardComponent < ViewComponent::Base
  def initialize(article:, show_author: true)
    @article = article
    @show_author = show_author
  end

  def formatted_date
    @article.published_at&.strftime("%B %d, %Y")
  end

  def reading_time
    words = @article.body.split.count
    "#{(words / 200.0).ceil} min read"
  end
end

# app/components/article_card_component.html.erb
# <div class="article-card">
#   <h2><%= link_to @article.title, article_path(@article) %></h2>
#   <p class="meta">
#     <%= formatted_date %> · <%= reading_time %>
#     <% if @show_author %>by <%= @article.author.name %><% end %>
#   </p>
#   <p><%= @article.excerpt %></p>
# </div>

# Usage in any view:
# <%= render ArticleCardComponent.new(article: @article) %>
# <%= render ArticleCardComponent.new(article: @article, show_author: false) %>

# Testing (key benefit over partials):
RSpec.describe ArticleCardComponent, type: :component do
  it "renders the article title" do
    article = create(:article, title: "Test", published_at: 1.day.ago)
    render_inline(ArticleCardComponent.new(article: article))
    expect(page).to have_text("Test")
  end

  it "shows reading time" do
    article = create(:article, body: "word " * 400)  # 400 words
    render_inline(ArticleCardComponent.new(article: article))
    expect(page).to have_text("2 min read")
  end
end
```

---

**Q10. What is Hotwire? Explain Turbo Drive, Turbo Frames, and Turbo Streams.**

```
Hotwire = HTML Over The Wire (default in Rails 7)
Two main libraries: Turbo + Stimulus

TURBO DRIVE (formerly Turbolinks):
  - Intercepts ALL <a href> clicks and form submits
  - Makes AJAX request instead of full page load
  - Replaces only <body> content (keeps <head> assets loaded)
  - Result: Single Page App feel without writing JavaScript
  - Benefit: CSS/JS not reloaded on every navigation

TURBO FRAMES:
  - Scoped page updates — only update a specific section
  - <turbo-frame id="comments">...</turbo-frame>
  - Links/forms inside a frame update ONLY that frame
  - Server renders full page but only the matching frame is used
  Example:
  <turbo-frame id="article_1">
    <h2>Article Title</h2>
    <a href="/articles/1/edit">Edit</a>  ← replaces only this frame
  </turbo-frame>

TURBO STREAMS:
  - Real-time targeted updates via WebSocket or HTTP response
  - 8 actions: append, prepend, replace, update, remove, before, after, refresh
  - Response Content-Type: text/vnd.turbo-stream.html
  Example (in controller after save):
  render turbo_stream: turbo_stream.append("comments",
    partial: "comments/comment", locals: { comment: @comment })
```

---

**Q11. What is fragment caching in Rails views? How do you implement Russian doll caching?**

```erb
<%# Basic fragment cache — cache this block %>
<% cache @article do %>
  <article>
    <h1><%= @article.title %></h1>
    <%= render 'body', article: @article %>
    <p>Comments: <%= @article.comments.count %></p>
  </article>
<% end %>
<%# Cache key: "articles/1-20240115123456" (model + updated_at) %>
<%# Invalidation: @article.touch → updated_at changes → new cache key %>

<%# Russian Doll: outer caches contain inner caches %>
<% cache @article do %>                  <%# Outer cache %>
  <h1><%= @article.title %></h1>

  <% @article.comments.each do |comment| %>
    <% cache comment do %>               <%# Inner cache per comment %>
      <div class="comment">
        <strong><%= comment.author.name %></strong>
        <p><%= comment.body %></p>
      </div>
    <% end %>
  <% end %>
<% end %>

<%# When a comment is updated:
    - comment.updated_at changes → inner cache for that comment misses
    - comment.touch :article (via belongs_to :article, touch: true) → article.updated_at updates
    - article's outer cache key changes → entire article re-renders
    - BUT: other comments' inner caches are still valid! %>
```

---

**Q12. What are date/number helpers in ActionView? List 5 useful ones.**

```erb
<%# Number helpers %>
<%= number_to_currency(1234.56) %>              <%# "$1,234.56" %>
<%= number_to_currency(1234.56, unit: "€", separator: ",", delimiter: ".") %>  <%# "€1.234,56" %>
<%= number_to_percentage(33.33) %>              <%# "33.330%" %>
<%= number_to_percentage(33.33, precision: 1) %><%# "33.3%" %>
<%= number_with_delimiter(1234567) %>           <%# "1,234,567" %>
<%= number_to_human(1_234_567) %>               <%# "1.23 Million" %>
<%= number_to_human_size(1_024_000) %>          <%# "1000 KB" %>

<%# Time/Date helpers %>
<%= time_ago_in_words(3.days.ago) %>            <%# "3 days" %>
<%= distance_of_time_in_words(1.hour.ago, Time.current) %>  <%# "about 1 hour" %>
<%= l @article.published_at, format: :long %>   <%# Localized: "January 15, 2024" %>

<%# Text helpers %>
<%= truncate(@article.body, length: 100) %>     <%# "Long text..." %>
<%= pluralize(@articles.count, "article") %>    <%# "3 articles" or "1 article" %>
<%= highlight(@article.body, "Rails") %>        <%# Wraps "Rails" in <mark> %>
<%= word_wrap(@article.body, line_width: 80) %> <%# Wraps at 80 chars %>
```

---

**Q13. What is Stimulus.js in the Hotwire stack? How does it differ from React/Vue?**

```javascript
// Stimulus: modest JavaScript — sprinkles behaviour on server-rendered HTML
// NOT a replacement for React; adds interactivity without replacing the view layer

// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]
  static values = { open: Boolean }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged() {
    this.menuTarget.hidden = !this.openValue
    this.buttonTarget.setAttribute("aria-expanded", this.openValue)
  }
}
```

```erb
<%# HTML with Stimulus data attributes %>
<div data-controller="dropdown">
  <button data-dropdown-target="button"
          data-action="click->dropdown#toggle">
    Menu
  </button>
  <ul data-dropdown-target="menu" hidden>
    <li>Profile</li>
    <li>Settings</li>
  </ul>
</div>
```

Stimulus philosophy:
- HTML is the source of truth (server renders it)
- JavaScript enhances HTML via `data-controller`, `data-action`, `data-target`
- No virtual DOM, no component lifecycle — just DOM augmentation
- Works with Turbo Drive (controllers connect/disconnect automatically on navigation)

---

**Q14. What is `ActionView::Helpers::TagHelper`? What does `content_tag` do?**

```ruby
# content_tag builds HTML tags programmatically — useful in helpers
content_tag(:div, "Hello", class: "greeting")
# => <div class="greeting">Hello</div>

content_tag(:ul, class: "nav") do
  content_tag(:li, link_to("Home", root_path)) +
  content_tag(:li, link_to("About", about_path))
end
# => <ul class="nav"><li><a href="/">Home</a></li><li>...</li></ul>

# tag.div, tag.span, etc. (Rails 5.1+ builder syntax):
tag.div(class: "card") do
  tag.h2("Title") + tag.p("Body")
end

# In helpers (access via helpers module):
def status_badge(status)
  colors = { active: "green", inactive: "red", pending: "yellow" }
  tag.span(status.humanize, class: "badge badge--#{colors[status.to_sym]}")
end
```

---

**Q15. What is the `simple_form` gem and how does it improve on `form_with`?**

```erb
<%# Built-in form_with — verbose for complex forms %>
<%= form_with model: @user do |f| %>
  <div class="field">
    <%= f.label :email %>
    <%= f.email_field :email, class: "form-control #{'is-invalid' if @user.errors[:email].any?}" %>
    <% @user.errors[:email].each do |error| %>
      <span class="invalid-feedback"><%= error %></span>
    <% end %>
  </div>
<% end %>

<%# simple_form gem — terse, handles errors/labels/wrapper automatically %>
<%= simple_form_for @user do |f| %>
  <%= f.input :email %>                    <%# label + input + error message auto %>
  <%= f.input :role, collection: User.roles.keys %>  <%# select from enum %>
  <%= f.input :bio, as: :text %>           <%# textarea %>
  <%= f.input :birthdate, as: :date %>     <%# date picker %>
  <%= f.input :active, as: :boolean %>     <%# checkbox %>
  <%= f.button :submit, "Save Profile", class: "btn btn-primary" %>
<% end %>
```

---

**Q16. Explain Turbo Streams — the 8 actions and when to use each.**

```ruby
# append: add to end of target element
turbo_stream.append("messages", partial: "messages/message", locals: { message: @message })

# prepend: add to beginning of target element
turbo_stream.prepend("notifications", @notification)

# replace: replace entire target element (including element itself)
turbo_stream.replace(@post)  # replaces <div id="post_1"> entirely

# update: replace CONTENT of target element (keep wrapper)
turbo_stream.update("post_title", @post.title)

# remove: delete target element
turbo_stream.remove(@comment)  # removes <div id="comment_42">

# before: insert before target element
turbo_stream.before("sidebar", partial: "ads/banner")

# after: insert after target element
turbo_stream.after("header", partial: "alerts/notice")

# refresh (Rails 7.1+): full page refresh via session refresh token
turbo_stream.action(:refresh)

# In controller:
def create
  @message = Message.create!(message_params)
  render turbo_stream: [
    turbo_stream.append("messages", @message),
    turbo_stream.update("message_count", current_user.messages.count)
  ]
end
```

---

**Q17. What is `render collection:` optimization? How does it batch partial rendering?**

```erb
<%# Slow: N ERB evaluations with N individual render calls %>
<% @articles.each do |article| %>
  <%= render article %>   <%# or render partial: 'article', locals: { article: article } %>
<% end %>

<%# Fast: single render with collection (batched) %>
<%= render @articles %>
<%# Rails renders all articles with _article.html.erb partial in one pass %>
<%# Local variable name matches partial name (article in _article.html.erb) %>

<%# With separator: %>
<%= render partial: "article", collection: @articles, spacer_template: "article_separator" %>

<%# With custom local variable name: %>
<%= render partial: "item", collection: @articles, as: :item %>
<%# _item.html.erb receives local: item %>

<%# With counter: render method provides article_counter automatically %>
<%# In _article.html.erb: article_counter (0-indexed) %>
```

---

**Q18. What are asset helpers in views? How do `javascript_include_tag` and `stylesheet_link_tag` work?**

```erb
<%# In app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <%# Generates: <link rel="stylesheet" href="/assets/application-[md5].css"> %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>

  <%# Import maps (Rails 7 default — no bundling needed): %>
  <%= javascript_importmap_tags %>

  <%# OR: jsbundling (esbuild/webpack): %>
  <%= javascript_include_tag "application", "data-turbo-track": "reload", defer: true %>
</head>
```

```ruby
# For import maps (config/importmap.rb):
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from "app/javascript/controllers", under: "controllers"
```

---

**Q19. How does Rails handle different layouts for different controllers or actions?**

```ruby
# Default: app/views/layouts/application.html.erb

# Per controller:
class AdminController < ApplicationController
  layout 'admin'  # uses app/views/layouts/admin.html.erb
end

# Conditional layout:
class ArticlesController < ApplicationController
  layout :choose_layout

  private

  def choose_layout
    current_user&.admin? ? 'admin' : 'application'
  end
end

# Per action:
class UsersController < ApplicationController
  def registration
    render layout: 'minimal'  # uses layouts/minimal.html.erb
  end

  def print_profile
    render layout: false  # no layout — raw HTML
  end
end

# No layout:
render layout: false  # useful for partial HTML responses, print views, emails
```

---

**Q20. What is `number_to_currency` and `l` (localize) helper? How does i18n work in views?**

```ruby
# config/locales/en.yml
en:
  activerecord:
    models:
      article: "Article"
  views:
    articles:
      title: "Article Collection"
  currency: "$"
  date:
    formats:
      default: "%m/%d/%Y"
      long: "%B %d, %Y"
  number:
    currency:
      format:
        unit: "$"
        precision: 2
        delimiter: ","
        separator: "."
```

```erb
<%# t() — translate %>
<h1><%= t('views.articles.title') %></h1>
<%# Or with scope: t('.title') uses current view scope automatically %>

<%# l() — localize (dates, times, numbers) %>
<%= l @article.published_at, format: :long %>    <%# "January 15, 2024" %>
<%= l Date.today, format: :default %>            <%# "01/15/2024" %>

<%# number helpers use locale settings: %>
<%= number_to_currency(@product.price) %>        <%# Uses locale for unit, separator %>

<%# Pluralization: %>
<%= t('articles.count', count: @articles.size) %>
# en.yml: articles: { count: { one: "1 article", other: "%{count} articles" } }
```

