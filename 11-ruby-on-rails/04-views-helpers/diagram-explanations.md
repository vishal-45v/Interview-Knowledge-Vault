# Chapter 04 — Views & Helpers: Diagram Explanations

---

## Diagram 1: Rails View Rendering Pipeline

```
Controller Action Completes
         │
         ▼
  ActionController::Renderer
         │
         ├── Determine template:
         │   app/views/[controller]/[action].html.erb
         │   (or .json.jbuilder, .xml.builder, etc.)
         │
         ▼
  Template Rendering
  app/views/articles/show.html.erb
         │
         ├── ERB evaluates Ruby code
         ├── Renders nested partials:
         │   render @article (→ _article.html.erb)
         │   render partial: 'sidebar', locals: { ... }
         ├── Checks fragment caches
         ├── Calls helper methods
         │
         ▼
  Template HTML String produced
         │
         ▼
  Layout Wrapping
  app/views/layouts/application.html.erb
         │
         ├── ERB evaluates layout
         ├── yield → injects template HTML
         ├── yield :title → content_for(:title) from template
         ├── yield :sidebar → content_for(:sidebar) from template
         │
         ▼
  Final HTML Response
  (single string sent to browser)

Partial Resolution:
  render 'article'              → app/views/[current_controller]/_article.html.erb
  render 'articles/article'    → app/views/articles/_article.html.erb
  render @article               → app/views/articles/_article.html.erb (based on class)
  render @articles              → same partial for each, collection
  render ArticleCardComponent   → ViewComponent (Ruby object + template)
```

---

## Diagram 2: Hotwire Architecture

```
FULL HOTWIRE STACK (Rails 7 default):

Browser
  │
  │  User clicks link
  ▼
Turbo Drive (intercepts clicks & form submits)
  │
  ├── Same origin? → AJAX fetch instead of full page load
  │   ├── Response 200: extract <body>, replace current <body>
  │   ├── Response 3xx: follow redirect with Turbo
  │   └── Response 4xx/5xx: render as full page
  │
  ├── data-turbo-frame set on link → delegate to Turbo Frames
  │
  └── Full page navigation (data-turbo="false" or external URL)

Turbo Frames (scoped updates):
  <turbo-frame id="comments">
    │
    │ Link inside frame is clicked
    ▼
  AJAX request made → server returns full page
    │
    │ Turbo extracts matching <turbo-frame id="comments"> from response
    ▼
  Only that frame's content replaced (rest of page unchanged)

Turbo Streams (targeted updates):
  Server sends text/vnd.turbo-stream.html:
  <turbo-stream action="append" target="messages">
    <template><div>New message</div></template>
  </turbo-stream>
    │
    ▼
  Turbo finds #messages in current DOM
  Appends new content → DOM updated without page load

  Real-time via ActionCable:
  ActionCable WebSocket → broadcasts Turbo Stream → browser DOM updates

Stimulus (behaviour layer):
  data-controller="dropdown"  → Stimulus loads DropdownController
  data-action="click->dropdown#toggle" → Stimulus binds event listener
  data-dropdown-target="menu" → Stimulus tracks this DOM element
  (Stimulus auto-connects/disconnects during Turbo navigation)
```

---

## Diagram 3: Fragment Cache Hierarchy (Russian Doll)

```
Page: /posts/1

  ┌─────────────────────────────────────────────────────────────┐
  │ OUTER CACHE: cache @post                                    │
  │ Key: "posts/1-20240115120000"                               │
  │                                                             │
  │  <h1>Post Title</h1>                                        │
  │  <p>Post body...</p>                                        │
  │                                                             │
  │  ┌────────────────────────────────────────────────────────┐ │
  │  │ MIDDLE CACHE: cache ["post-comments", @post,           │ │
  │  │               @post.comments.maximum(:updated_at)]     │ │
  │  │                                                        │ │
  │  │  <div id="comments">                                   │ │
  │  │                                                        │ │
  │  │  ┌──────────────────────────────────────────────────┐  │ │
  │  │  │ INNER CACHE: cache comment_1                      │  │ │
  │  │  │ Key: "comments/1-20240110150000"                  │  │ │
  │  │  │  <div>Alice: First comment</div>                  │  │ │
  │  │  └──────────────────────────────────────────────────┘  │ │
  │  │                                                        │ │
  │  │  ┌──────────────────────────────────────────────────┐  │ │
  │  │  │ INNER CACHE: cache comment_2                      │  │ │
  │  │  │ Key: "comments/2-20240112090000"  ← CHANGED      │  │ │
  │  │  │  <div>Bob: Updated comment</div>  ← re-rendered  │  │ │
  │  │  └──────────────────────────────────────────────────┘  │ │
  │  │                                                        │ │
  │  └────────────────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────────────────┘

WHEN comment_2 IS UPDATED:
  1. comment_2.save → comment_2.updated_at changes → inner cache MISS for comment_2
  2. belongs_to :post, touch: true → post.updated_at changes → outer cache MISS
  3. Re-render outer: checks middle cache → middle MISS (post.updated_at changed)
  4. Re-render middle: checks each inner cache
     - comment_1 inner cache: HIT (still valid, reused!)
     - comment_2 inner cache: MISS (re-rendered)
  5. New outer and middle cache stored

RESULT: Only comment_2 was actually re-rendered.
        comment_1's HTML was served from cache.
        Efficiency increases with more comments.
```

---

## Diagram 4: XSS Prevention in Rails

```
User submits: name = "<script>alert('xss')</script>"

                    BROWSER DISPLAY
                    ─────────────────────────────────────────────
No protection:      <script>alert('xss')</script>
(raw / html_safe)   → Script EXECUTES in browser  ← DANGEROUS

Default ERB         &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
<%= @name %>        → Displays as TEXT: <script>alert('xss')</script>
                    → Script does NOT execute  ← SAFE

sanitize(@name,     (empty — no allowed tags contain script)
  tags: ['b','p'])  → Empty or stripped output  ← SAFE

strip_tags(@name)   alert('xss')
                    → Only text content  ← SAFE

─────────────────────────────────────────────────────────────────────────

SAFE interpolation in html_safe strings:
"<b>#{@name}</b>".html_safe
→ ERB's string interpolation escapes @name BEFORE html_safe:
→ "<b>&lt;script&gt;...&lt;/script&gt;</b>"
→ The <b> tags render as bold, the @name content is escaped  ← SAFE

DANGEROUS:
"<b>#{@name.html_safe}</b>".html_safe
→ @name.html_safe prevents escaping during interpolation
→ "<b><script>...</script></b>"  ← script executes!  DANGEROUS
```

---

## Diagram 5: ViewComponent vs Partial — Structure Comparison

```
PARTIAL APPROACH:
─────────────────────────────────────────────────────────────────────────
app/views/
  articles/
    _article_card.html.erb    ← template only (logic mixed in)
  shared/
    _article_card.html.erb    ← or shared location (confusing)

Rendering:
  render 'article_card', article: @article
  render @article              (implicit — must match filename)

Testing:
  Full integration test needed (controller + view)
  Hard to test in isolation

Reuse:
  Pass locals — any variable can be passed (no type checking)

─────────────────────────────────────────────────────────────────────────
VIEWCOMPONENT APPROACH:
─────────────────────────────────────────────────────────────────────────
app/components/
  article_card_component.rb      ← Ruby class (interface, logic)
  article_card_component.html.erb← template (minimal logic)

spec/components/
  article_card_component_spec.rb ← isolated unit tests

Rendering:
  render ArticleCardComponent.new(article: @article)
  render ArticleCardComponent.new(article: @article, show_author: false)

Testing:
  render_inline(ArticleCardComponent.new(article: @article))
  expect(page).to have_css(".article-card")
  # No Rails request context needed — runs in milliseconds

Reuse:
  Strong interface: initialize(article:, show_author: true)
  TypeErrors caught at component instantiation
  Self-documenting parameters

Performance (Rails 7):
  ViewComponent: ~10x faster rendering than partials for complex components
  (No ERB binding overhead per-render)
```

---

## Diagram 6: Turbo Streams — 8 Actions Reference

```
Current DOM:
  <ul id="messages">
    <li id="message_1">Hello</li>
    <li id="message_2">World</li>
  </ul>
  <div id="footer">Footer content</div>
  <span id="count">2 messages</span>

─────────────────────────────────────────────────────────────────────────
ACTION        │ TURBO STREAM CODE                    │ RESULT
─────────────────────────────────────────────────────────────────────────
append        │ turbo_stream.append("messages",       │ <li>New</li> added
              │   "<li>New</li>")                     │ at END of #messages
─────────────────────────────────────────────────────────────────────────
prepend       │ turbo_stream.prepend("messages",      │ <li>First</li> added
              │   "<li>First</li>")                   │ at START of #messages
─────────────────────────────────────────────────────────────────────────
replace       │ turbo_stream.replace("message_1",     │ Entire <li id="message_1">
              │   "<li id='message_1'>Hi!</li>")      │ element replaced (incl. wrapper)
─────────────────────────────────────────────────────────────────────────
update        │ turbo_stream.update("count",          │ Content of <span id="count">
              │   "3 messages")                       │ changed, wrapper <span> kept
─────────────────────────────────────────────────────────────────────────
remove        │ turbo_stream.remove("message_2")      │ <li id="message_2"> removed
─────────────────────────────────────────────────────────────────────────
before        │ turbo_stream.before("footer",         │ <p>Notice</p> inserted
              │   "<p>Notice</p>")                    │ BEFORE #footer element
─────────────────────────────────────────────────────────────────────────
after         │ turbo_stream.after("footer",          │ <p>Tip</p> inserted
              │   "<p>Tip</p>")                       │ AFTER #footer element
─────────────────────────────────────────────────────────────────────────
refresh       │ turbo_stream.action(:refresh)         │ Full page refresh
(Rails 7.1+)  │                                       │ (via session token)
─────────────────────────────────────────────────────────────────────────

Wire format (HTTP response or ActionCable message):
  Content-Type: text/vnd.turbo-stream.html

  <turbo-stream action="append" target="messages">
    <template>
      <li id="message_3">New message content</li>
    </template>
  </turbo-stream>
```

