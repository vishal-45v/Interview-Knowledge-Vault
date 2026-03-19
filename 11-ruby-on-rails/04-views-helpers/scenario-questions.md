# Chapter 04 — Views & Helpers: Scenario Questions

---

**S1. A user reports they can inject HTML into their profile bio which renders as actual HTML in the app. How do you fix this?**

```erb
<%# Bug: bio contains <script>alert('xss')</script> and it executes %>
<p><%= @user.bio.html_safe %></p>  <%# DANGEROUS — never html_safe user content %>
<%# OR %>
<p><%= raw @user.bio %></p>        <%# Same bug — raw is alias for html_safe %>

<%# Fix 1: Remove html_safe — Rails escapes by default %>
<p><%= @user.bio %></p>
<%# Output: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt; — harmless text %>

<%# Fix 2: If you want to ALLOW some HTML (markdown/rich text): %>
<p><%= sanitize @user.bio, tags: %w[p br strong em ul li a], attributes: %w[href] %></p>

<%# Fix 3: ActionText (rich text, sanitized automatically): %>
<%# In the model: has_rich_text :bio %>
<p><%= @user.bio %></p>  <%# ActionText renders sanitized HTML automatically %>

<%# Fix 4: Strip ALL HTML, render as plain text: %>
<p><%= strip_tags(@user.bio) %></p>
```

---

**S2. Your product listing page with 50 products takes 2 seconds to render. Each product card renders a partial that loads the product's image, price, and category name. How do you optimize it?**

```erb
<%# Problem: 50 partials × queries for category name = N+1 + no caching %>

<%# Step 1: Fix N+1 in controller %>
<%# @products = Product.includes(:category, :primary_image).published.limit(50) %>

<%# Step 2: Use collection rendering (batched partials) %>
<%# SLOW: %>
<% @products.each do |product| %>
  <%= render product %>
<% end %>

<%# FAST: collection rendering — single call, optimized by Rails %>
<%= render @products %>  <%# Uses _product.html.erb partial %>

<%# Step 3: Add fragment caching %>
<%# app/views/products/_product.html.erb %>
<% cache product do %>
  <div class="product-card">
    <%= image_tag product.primary_image.url, alt: product.name, loading: "lazy" %>
    <h3><%= product.name %></h3>
    <p class="price"><%= number_to_currency(product.price) %></p>
    <span class="category"><%= product.category.name %></span>
  </div>
<% end %>

<%# Step 4: Cache the collection too %>
<% cache ["products-list", @products.maximum(:updated_at)] do %>
  <%= render @products %>
<% end %>
```

---

**S3. You need to implement a real-time comment system where new comments appear without a full page refresh. How do you use Turbo Streams?**

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>

<div id="comments">
  <%= render @post.comments %>
</div>

<%= turbo_stream_from "post_#{@post.id}_comments" %>
<%# Subscribes this page to the "post_X_comments" stream via ActionCable %>

<%= render "comments/form", post: @post, comment: Comment.new %>

<%# app/views/comments/_comment.html.erb %>
<div id="<%= dom_id(comment) %>">
  <strong><%= comment.author.name %></strong>
  <p><%= comment.body %></p>
  <small><%= time_ago_in_words(comment.created_at) %> ago</small>
</div>

<%# app/views/comments/_form.html.erb %>
<%= form_with model: [@post, @comment] do |f| %>
  <%= f.text_area :body, rows: 3, placeholder: "Add a comment..." %>
  <%= f.submit "Post Comment" %>
<% end %>
```

```ruby
# app/controllers/comments_controller.rb
def create
  @comment = @post.comments.build(comment_params.merge(author: current_user))
  if @comment.save
    # Broadcast to all viewers of this post:
    Turbo::StreamsChannel.broadcast_append_to(
      "post_#{@post.id}_comments",
      target: "comments",
      partial: "comments/comment",
      locals: { comment: @comment }
    )

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.append("comments", @comment),
          turbo_stream.replace("new_comment_form",
            partial: "comments/form",
            locals: { post: @post, comment: Comment.new })
        ]
      end
      format.html { redirect_to @post }
    end
  else
    render "posts/show", status: :unprocessable_entity
  end
end
```

---

**S4. You need to build a reusable button component that handles loading state, disabled state, and different styles (primary, secondary, danger). How do you implement this with ViewComponent?**

```ruby
# app/components/button_component.rb
class ButtonComponent < ViewComponent::Base
  VARIANTS = %w[primary secondary danger outline ghost].freeze

  def initialize(
    label:,
    url: nil,
    method: :get,
    variant: :primary,
    size: :md,
    loading: false,
    disabled: false,
    icon: nil,
    **html_options
  )
    @label = label
    @url = url
    @method = method
    @variant = variant.to_s.in?(VARIANTS) ? variant.to_s : "primary"
    @size = size
    @loading = loading
    @disabled = disabled || loading
    @icon = icon
    @html_options = html_options
  end

  def css_classes
    base = "btn btn--#{@variant} btn--#{@size}"
    base += " btn--loading" if @loading
    base += " btn--disabled" if @disabled
    base += " #{@html_options.delete(:class)}" if @html_options[:class]
    base
  end
end
```

```erb
<%# app/components/button_component.html.erb %>
<% if @url %>
  <%= link_to @url, class: css_classes, **@html_options,
              aria: { disabled: @disabled } do %>
    <% if @loading %>
      <span class="btn__spinner" aria-hidden="true"></span>
    <% end %>
    <% if @icon %>
      <%= inline_svg_tag @icon, class: "btn__icon" %>
    <% end %>
    <span class="btn__label"><%= @label %></span>
  <% end %>
<% else %>
  <button class="<%= css_classes %>"
          <%= "disabled" if @disabled %>
          **@html_options>
    <% if @loading %>
      <span class="btn__spinner" aria-hidden="true"></span>
    <% end %>
    <span class="btn__label"><%= @label %></span>
  </button>
<% end %>

<%# Usage: %>
<%# <%= render ButtonComponent.new(label: "Save", variant: :primary) %> %>
<%# <%= render ButtonComponent.new(label: "Saving...", loading: true) %> %>
<%# <%= render ButtonComponent.new(label: "Delete", url: article_path(@article), method: :delete, variant: :danger) %> %>
```

---

**S5. A form has a dynamic field (quantity can be 1–10, adds more fields). How do you implement this without a JavaScript framework using Stimulus?**

```javascript
// app/javascript/controllers/dynamic_fields_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["fieldset", "template", "addButton"]
  static values = { max: Number }

  connect() {
    this.updateAddButton()
  }

  add() {
    if (this.fieldsetTargets.length >= this.maxValue) return

    const template = this.templateTarget.innerHTML
    const index = this.fieldsetTargets.length
    const newField = template.replace(/NEW_RECORD/g, new Date().getTime())

    this.templateTarget.insertAdjacentHTML('beforebegin', newField)
    this.updateAddButton()
  }

  remove(event) {
    const fieldset = event.target.closest('[data-dynamic-fields-target="fieldset"]')
    const destroyInput = fieldset.querySelector('input[name*="_destroy"]')

    if (destroyInput) {
      destroyInput.value = "1"
      fieldset.hidden = true
    } else {
      fieldset.remove()
    }
    this.updateAddButton()
  }

  updateAddButton() {
    const atMax = this.fieldsetTargets.filter(f => !f.hidden).length >= this.maxValue
    this.addButtonTarget.disabled = atMax
  }
}
```

```erb
<%# app/views/orders/_line_items.html.erb %>
<div data-controller="dynamic-fields" data-dynamic-fields-max-value="10">
  <div id="line-items">
    <% @order.line_items.each.with_index do |item, i| %>
      <div data-dynamic-fields-target="fieldset" class="line-item">
        <%= f.fields_for :line_items, item do |lf| %>
          <%= lf.hidden_field :id %>
          <%= lf.text_field :product_name, placeholder: "Product" %>
          <%= lf.number_field :quantity, min: 1, max: 100 %>
          <%= lf.hidden_field :_destroy %>
          <button type="button" data-action="click->dynamic-fields#remove">Remove</button>
        <% end %>
      </div>
    <% end %>
  </div>

  <%# Template for new items %>
  <template data-dynamic-fields-target="template">
    <div data-dynamic-fields-target="fieldset" class="line-item">
      <%= f.fields_for :line_items, LineItem.new, child_index: "NEW_RECORD" do |lf| %>
        <%= lf.text_field :product_name, placeholder: "Product" %>
        <%= lf.number_field :quantity, min: 1, max: 100 %>
        <button type="button" data-action="click->dynamic-fields#remove">Remove</button>
      <% end %>
    </div>
  </template>

  <button type="button" data-dynamic-fields-target="addButton"
          data-action="click->dynamic-fields#add"
          class="btn btn--secondary">
    + Add Item
  </button>
</div>
```

---

**S6. You need to implement infinite scroll for a product listing page. How do you do this with Turbo Frames?**

```erb
<%# app/views/products/index.html.erb %>
<h1>Products</h1>

<div id="products">
  <%= render @products %>
</div>

<% if @products.next_page %>
  <turbo-frame id="products_pagination"
               src="<%= products_path(page: @products.next_page) %>"
               loading="lazy">
    <%# This frame loads when it scrolls into view (loading="lazy") %>
    <div class="loading-spinner">Loading...</div>
  </turbo-frame>
<% end %>
```

```ruby
# app/controllers/products_controller.rb
def index
  @products = Product.published.order(created_at: :desc).page(params[:page]).per(12)
end
```

```erb
<%# When lazy-loaded, full page renders but only turbo-frame id="products_pagination" is used %>
<%# In the response for page 2: %>
<div id="products">
  <%# Already on page — ignored %>
</div>

<turbo-frame id="products_pagination">
  <%# This content REPLACES the spinner in the original frame %>
  <%= render @products %>

  <% if @products.next_page %>
    <turbo-frame id="products_pagination"
                 src="<%= products_path(page: @products.next_page) %>"
                 loading="lazy">
      <div class="loading-spinner">Loading more...</div>
    </turbo-frame>
  <% end %>
</turbo-frame>
```

---

**S7. You need to implement a search-as-you-type feature that updates results live as the user types. How do you do this with Stimulus and Turbo Frames?**

```javascript
// app/javascript/controllers/live_search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "results"]
  static values = { url: String, delay: { type: Number, default: 300 } }

  search() {
    clearTimeout(this.debounceTimer)
    this.debounceTimer = setTimeout(() => {
      this.performSearch()
    }, this.delayValue)
  }

  async performSearch() {
    const query = this.inputTarget.value
    if (query.length < 2) {
      this.resultsTarget.innerHTML = ""
      return
    }

    const url = new URL(this.urlValue, window.location.origin)
    url.searchParams.set("q", query)

    const response = await fetch(url, {
      headers: { Accept: "text/vnd.turbo-stream.html" }
    })
    const html = await response.text()
    Turbo.renderStreamMessage(html)
  }

  disconnect() {
    clearTimeout(this.debounceTimer)
  }
}
```

```erb
<%# app/views/products/index.html.erb %>
<div data-controller="live-search"
     data-live-search-url-value="<%= search_products_path %>">
  <input type="search"
         data-live-search-target="input"
         data-action="input->live-search#search"
         placeholder="Search products..."
         autocomplete="off">

  <turbo-frame id="search-results">
    <%= render @products %>
  </turbo-frame>
</div>
```

```ruby
# app/controllers/products_controller.rb
def search
  @products = Product.search(params[:q]).limit(20)
  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: turbo_stream.update("search-results", partial: "products/list", locals: { products: @products })
    end
    format.html { render :index }
  end
end
```

---

**S8. How do you implement a modal dialog with Turbo Frames that loads its content lazily from the server?**

```erb
<%# app/views/shared/_modal.html.erb — reusable wrapper %>
<div id="modal" class="modal-backdrop"
     data-controller="modal"
     data-action="click->modal#close">
  <turbo-frame id="modal-content" class="modal-dialog"
               data-action="click->modal#stopPropagation">
    <%# Lazy-loaded turbo frame content goes here %>
  </turbo-frame>
</div>

<%# Any link that should open in modal: %>
<%= link_to "Edit Profile", edit_profile_path,
            data: { turbo_frame: "modal-content" },
            class: "btn" %>
<%# Clicking this link loads edit_profile page into modal-content frame %>
<%# Only the <turbo-frame id="modal-content"> section from that page is used %>
```

```erb
<%# app/views/profiles/edit.html.erb %>
<turbo-frame id="modal-content">
  <div class="modal-header">
    <h2>Edit Profile</h2>
    <%= link_to "✕", "#", data: { action: "click->modal#close" } %>
  </div>
  <%= render "form", profile: @profile %>
</turbo-frame>
<%# Only this frame is used in the modal — rest of page ignored %>
```

```javascript
// app/javascript/controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]

  close() {
    this.element.remove()
  }

  stopPropagation(event) {
    event.stopPropagation()
  }
}
```

---

**S9. Implement a view helper that generates a breadcrumb trail from the current page's hierarchy.**

```ruby
# app/helpers/breadcrumbs_helper.rb
module BreadcrumbsHelper
  def breadcrumbs(*crumbs, &block)
    content_tag(:nav, aria: { label: "Breadcrumb" }) do
      content_tag(:ol, class: "breadcrumbs") do
        items = crumbs.map.with_index do |(label, url), index|
          is_last = index == crumbs.length - 1
          content_tag(:li, class: "breadcrumbs__item #{'breadcrumbs__item--active' if is_last}",
                          aria: { current: is_last ? "page" : nil }) do
            if url && !is_last
              link_to(label, url, class: "breadcrumbs__link")
            else
              content_tag(:span, label, class: "breadcrumbs__label")
            end
          end
        end
        safe_join(items)
      end
    end
  end
end

# Usage in views:
# <%= breadcrumbs ["Home", root_path], ["Products", products_path], ["Widget", nil] %>

# Better: centralized breadcrumb building with before_action
class ApplicationController < ActionController::Base
  before_action :set_breadcrumbs
  helper_method :breadcrumbs_data

  private

  def add_breadcrumb(label, url = nil)
    @breadcrumbs ||= [["Home", root_path]]
    @breadcrumbs << [label, url]
  end

  def set_breadcrumbs
    @breadcrumbs = [["Home", root_path]]
  end

  def breadcrumbs_data
    @breadcrumbs || []
  end
end

class ProductsController < ApplicationController
  def index
    add_breadcrumb "Products"
  end

  def show
    @product = Product.find(params[:id])
    add_breadcrumb "Products", products_path
    add_breadcrumb @product.name
  end
end
```

---

**S10. How do you handle time zones in views when users are in different time zones?**

```ruby
# config/application.rb
config.time_zone = "UTC"  # App's default time zone

# User model stores their preferred timezone:
class User < ApplicationRecord
  validates :timezone, inclusion: { in: ActiveSupport::TimeZone.all.map(&:name) }
end

# ApplicationController — set user's timezone for duration of request:
class ApplicationController < ActionController::Base
  around_action :set_user_time_zone

  private

  def set_user_time_zone
    if current_user&.timezone
      Time.use_zone(current_user.timezone) { yield }
    else
      yield
    end
  end
end
```

```erb
<%# In views — l() uses current timezone: %>
<%= l @article.published_at, format: :long %>
<%# If user's timezone is "Pacific Time (US & Canada)":
    Shows: "January 14, 2024 04:00 PM" (converted from UTC) %>

<%# Explicit conversion: %>
<%= @article.published_at.in_time_zone(current_user.timezone).strftime("%B %d at %I:%M %p %Z") %>

<%# Timezone selector in settings form: %>
<%= f.select :timezone, ActiveSupport::TimeZone.all.map { |tz| [tz.to_s, tz.name] }, include_blank: false %>

<%# Display relative times (always in user's zone via time_ago_in_words): %>
<time datetime="<%= @post.created_at.iso8601 %>"
      title="<%= l @post.created_at, format: :long %>">
  <%= time_ago_in_words(@post.created_at) %> ago
</time>
```

---

**S11. You need to implement a multi-step onboarding wizard with progress indication. Each step is a separate page but shares a common layout. How do you structure the views?**

```ruby
# app/controllers/onboarding_controller.rb
class OnboardingController < ApplicationController
  layout 'onboarding'
  before_action :authenticate_user!
  before_action :prevent_completed_onboarding

  STEPS = %w[welcome profile preferences notifications done].freeze

  def show
    @step    = params[:step].presence_in(STEPS) || STEPS.first
    @step_index = STEPS.index(@step)
    render @step
  end

  def update
    @step = params[:step]
    @step_index = STEPS.index(@step)

    if save_step_data
      next_step = STEPS[@step_index + 1]
      if next_step == "done"
        current_user.update!(onboarding_completed_at: Time.current)
        redirect_to dashboard_path, notice: "Welcome! Setup complete."
      else
        redirect_to onboarding_path(step: next_step)
      end
    else
      render @step, status: :unprocessable_entity
    end
  end

  private

  def prevent_completed_onboarding
    redirect_to dashboard_path if current_user.onboarding_completed?
  end

  def save_step_data
    case @step
    when "profile"
      current_user.update(profile_params)
    when "preferences"
      current_user.preferences.update(preference_params)
    else
      true
    end
  end
end
```

```erb
<%# app/views/layouts/onboarding.html.erb %>
<!DOCTYPE html>
<html>
<head><title>Setup - MyApp</title></head>
<body class="onboarding-layout">
  <div class="onboarding-container">
    <div class="progress-bar">
      <% OnboardingController::STEPS.each_with_index do |step, i| %>
        <div class="progress-step
                    <%= 'completed' if i < @step_index %>
                    <%= 'active' if i == @step_index %>"
             aria-label="Step <%= i + 1 %>: <%= step.humanize %>">
        </div>
      <% end %>
    </div>
    <div class="step-content">
      <%= yield %>
    </div>
  </div>
</body>
</html>

<%# app/views/onboarding/profile.html.erb %>
<h1>Tell us about yourself</h1>
<p class="step-counter">Step <%= @step_index + 1 %> of <%= OnboardingController::STEPS.size - 1 %></p>

<%= form_with url: onboarding_path(step: @step), method: :patch do |f| %>
  <%= f.text_field :name, placeholder: "Your full name" %>
  <%= f.text_field :username, placeholder: "Choose a username" %>
  <%= f.file_field :avatar %>
  <%= f.submit "Continue →", class: "btn btn--primary btn--full" %>
<% end %>
```

---

**S12. How do you implement print-friendly views in Rails?**

```ruby
# config/routes.rb
resources :invoices do
  member { get :print }
end

# app/controllers/invoices_controller.rb
def print
  @invoice = Invoice.find(params[:id])
  render layout: 'print'  # minimal layout with print CSS
end
# Or: use respond_to
def show
  @invoice = Invoice.find(params[:id])
  respond_to do |format|
    format.html
    format.pdf do
      pdf = InvoicePdf.new(@invoice).render
      send_data pdf, filename: "invoice_#{@invoice.number}.pdf",
                     type: "application/pdf", disposition: "inline"
    end
  end
end
```

```erb
<%# app/views/layouts/print.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title>Invoice #<%= @invoice.number %></title>
  <style>
    @media print {
      .no-print { display: none; }
      body { font-family: serif; font-size: 12pt; }
    }
    @page { margin: 1cm; }
  </style>
</head>
<body>
  <div class="no-print">
    <button onclick="window.print()">Print</button>
    <%= link_to "← Back", invoice_path(@invoice) %>
  </div>
  <%= yield %>
</body>
</html>
```

