# Chapter 07: Testing with RSpec — Scenario Questions

**S1. You have a `PaymentService` that calls an external Stripe API. How do you test it without making real HTTP calls?**

```ruby
class PaymentService
  def initialize(stripe_client: Stripe::Charge)
    @stripe_client = stripe_client
  end

  def charge(amount_cents:, token:, description:)
    charge = @stripe_client.create(
      amount: amount_cents,
      currency: "usd",
      source: token,
      description: description
    )
    { success: true, charge_id: charge.id }
  rescue Stripe::CardError => e
    { success: false, error: e.message }
  end
end

# Test with dependency injection + instance_double
RSpec.describe PaymentService do
  let(:stripe_double) { class_double(Stripe::Charge) }
  let(:service)       { PaymentService.new(stripe_client: stripe_double) }

  describe "#charge" do
    context "when charge succeeds" do
      let(:charge_response) { instance_double(Stripe::Charge, id: "ch_123") }

      before do
        allow(stripe_double).to receive(:create).and_return(charge_response)
      end

      it "returns success with charge id" do
        result = service.charge(amount_cents: 1000, token: "tok_visa", description: "Test")
        expect(result).to eq({ success: true, charge_id: "ch_123" })
      end

      it "calls Stripe with correct params" do
        service.charge(amount_cents: 1000, token: "tok_visa", description: "Order #42")
        expect(stripe_double).to have_received(:create).with(
          amount: 1000,
          currency: "usd",
          source: "tok_visa",
          description: "Order #42"
        )
      end
    end

    context "when card is declined" do
      before do
        allow(stripe_double).to receive(:create)
          .and_raise(Stripe::CardError.new("Card declined", nil, nil))
      end

      it "returns failure with error message" do
        result = service.charge(amount_cents: 1000, token: "tok_visa", description: "Test")
        expect(result).to eq({ success: false, error: "Card declined" })
      end
    end
  end
end
```

---

**S2. You're testing a `UserMailer` that should send a welcome email. Test that the correct email is queued/sent.**

```ruby
class UserMailer < ApplicationMailer
  def welcome(user)
    @user = user
    mail(
      to:      user.email,
      subject: "Welcome to our platform, #{user.name}!"
    )
  end
end

RSpec.describe UserMailer do
  describe "#welcome" do
    let(:user) { build_stubbed(:user, name: "Alice", email: "alice@example.com") }
    let(:mail) { UserMailer.welcome(user) }

    it "renders the correct subject" do
      expect(mail.subject).to eq("Welcome to our platform, Alice!")
    end

    it "sends to the user's email" do
      expect(mail.to).to eq(["alice@example.com"])
    end

    it "sends from the default from address" do
      expect(mail.from).to eq(["noreply@example.com"])
    end

    it "includes the user's name in the body" do
      expect(mail.body.encoded).to include("Alice")
    end
  end
end

# Testing that mailer is called from a service:
RSpec.describe UserRegistrationService do
  let(:mailer_spy) { class_spy(UserMailer) }
  let(:service)    { UserRegistrationService.new(mailer: mailer_spy) }

  it "sends welcome email after registration" do
    service.register(name: "Alice", email: "alice@example.com")
    expect(mailer_spy).to have_received(:welcome).once
  end
end
```

---

**S3. Write tests for an API endpoint that returns paginated JSON. Include status code, body structure, and pagination metadata.**

```ruby
RSpec.describe "GET /api/v1/articles", type: :request do
  let!(:articles) { create_list(:article, 15, published: true) }
  let(:headers)   { { "Accept" => "application/json" } }

  context "with default pagination (page 1, per 10)" do
    before { get "/api/v1/articles", headers: headers }

    it "returns 200 OK" do
      expect(response).to have_http_status(:ok)
    end

    it "returns JSON content type" do
      expect(response.content_type).to include("application/json")
    end

    it "returns 10 articles" do
      data = JSON.parse(response.body)
      expect(data["articles"].length).to eq(10)
    end

    it "includes pagination metadata" do
      data = JSON.parse(response.body)
      expect(data["meta"]).to include(
        "current_page" => 1,
        "total_pages"  => 2,
        "total_count"  => 15,
        "per_page"     => 10
      )
    end

    it "returns articles with required fields" do
      article = JSON.parse(response.body)["articles"].first
      expect(article.keys).to include("id", "title", "summary", "published_at")
    end
  end

  context "with page=2" do
    before { get "/api/v1/articles?page=2", headers: headers }

    it "returns the remaining 5 articles" do
      data = JSON.parse(response.body)
      expect(data["articles"].length).to eq(5)
    end
  end

  context "when page exceeds total" do
    before { get "/api/v1/articles?page=99", headers: headers }

    it "returns an empty articles array" do
      data = JSON.parse(response.body)
      expect(data["articles"]).to be_empty
    end
  end
end
```

---

**S4. You have a class with a complex caching layer. How do you test the cache hit and cache miss paths independently?**

```ruby
class ProductRepository
  def initialize(cache: Rails.cache)
    @cache = cache
  end

  def find(id)
    @cache.fetch("product:#{id}", expires_in: 1.hour) do
      Product.find(id)
    end
  end

  def invalidate(id)
    @cache.delete("product:#{id}")
  end
end

RSpec.describe ProductRepository do
  let(:cache)      { instance_double(ActiveSupport::Cache::Store) }
  let(:product)    { build_stubbed(:product, id: 42) }
  let(:repo)       { ProductRepository.new(cache: cache) }

  describe "#find" do
    context "cache hit" do
      before do
        allow(cache).to receive(:fetch).with("product:42", expires_in: 1.hour)
          .and_return(product)
      end

      it "returns the cached product" do
        expect(repo.find(42)).to eq(product)
      end

      it "does not query the database" do
        expect(Product).not_to receive(:find)
        repo.find(42)
      end
    end

    context "cache miss" do
      before do
        allow(cache).to receive(:fetch).with("product:42", expires_in: 1.hour)
          .and_yield  # simulate cache miss by yielding to the block
          .and_return(product)
        allow(Product).to receive(:find).with(42).and_return(product)
      end

      it "fetches from database" do
        repo.find(42)
        expect(Product).to have_received(:find).with(42)
      end
    end
  end

  describe "#invalidate" do
    it "deletes the cache key" do
      allow(cache).to receive(:delete)
      repo.invalidate(42)
      expect(cache).to have_received(:delete).with("product:42")
    end
  end
end
```

---

**S5. Write shared examples for any object that implements the `Searchable` interface.**

```ruby
RSpec.shared_examples "searchable" do |factory_name|
  let(:searchable_instance) { create(factory_name) }

  it "responds to .search" do
    expect(described_class).to respond_to(:search)
  end

  it "returns a relation/collection from .search" do
    results = described_class.search("")
    expect(results).to respond_to(:each)
  end

  it "finds records matching the query" do
    create(factory_name, name: "Unique Search Term XYZ")
    results = described_class.search("Unique Search Term XYZ")
    expect(results.count).to be >= 1
  end

  it "returns empty when no match" do
    results = described_class.search("zzz_no_match_999")
    expect(results).to be_empty
  end

  it "is case-insensitive" do
    create(factory_name, name: "Ruby on Rails")
    expect(described_class.search("ruby on rails").count).to be >= 1
    expect(described_class.search("RUBY ON RAILS").count).to be >= 1
  end
end

RSpec.describe Article do
  it_behaves_like "searchable", :article
end

RSpec.describe User do
  it_behaves_like "searchable", :user
end

RSpec.describe Product do
  it_behaves_like "searchable", :product
end
```

---

**S6. Test a background job that sends notifications to multiple users.**

```ruby
class NotificationJob < ApplicationJob
  queue_as :default

  def perform(event_id)
    event = Event.find(event_id)
    event.attendees.each do |user|
      NotificationMailer.event_reminder(user, event).deliver_now
      user.update!(last_notified_at: Time.current)
    end
  end
end

RSpec.describe NotificationJob do
  describe "#perform" do
    let(:event)     { create(:event, name: "Ruby Conf") }
    let(:users)     { create_list(:user, 3) }
    let(:mailer_spy) { class_spy(NotificationMailer) }
    let(:mail_spy)   { instance_spy(ActionMailer::MessageDelivery) }

    before do
      event.attendees = users
      allow(NotificationMailer).to receive(:event_reminder).and_return(mail_spy)
      allow(mail_spy).to receive(:deliver_now)
    end

    it "sends a notification to each attendee" do
      NotificationJob.perform_now(event.id)
      expect(NotificationMailer).to have_received(:event_reminder)
        .exactly(3).times
    end

    it "sends to the correct event" do
      NotificationJob.perform_now(event.id)
      users.each do |user|
        expect(NotificationMailer).to have_received(:event_reminder)
          .with(user, event)
      end
    end

    it "updates last_notified_at for each user" do
      expect {
        NotificationJob.perform_now(event.id)
      }.to change { users.first.reload.last_notified_at }.from(nil)
    end

    it "raises and does not partially update on DB error" do
      allow(users.last).to receive(:update!).and_raise(ActiveRecord::RecordInvalid)
      expect { NotificationJob.perform_now(event.id) }.to raise_error(ActiveRecord::RecordInvalid)
    end
  end
end
```

---

**S7. Test a module that you've included into multiple classes to ensure consistent behavior.**

```ruby
module Auditable
  def self.included(base)
    base.before_save :set_audit_timestamps
  end

  def set_audit_timestamps
    self.created_at ||= Time.current
    self.updated_at   = Time.current
  end

  def audit_log
    "#{self.class}##{id} modified at #{updated_at}"
  end
end

RSpec.shared_examples "auditable" do
  subject { described_class.new }

  it "responds to audit_log" do
    expect(subject).to respond_to(:audit_log)
  end

  it "sets updated_at on save" do
    record = create(described_class.name.underscore.to_sym)
    original_time = record.updated_at

    travel_to 1.minute.from_now do
      record.touch
      expect(record.reload.updated_at).to be > original_time
    end
  end

  it "includes class name in audit log" do
    record = create(described_class.name.underscore.to_sym)
    expect(record.audit_log).to include(described_class.name)
  end
end

RSpec.describe User,    type: :model do
  it_behaves_like "auditable"
end

RSpec.describe Product, type: :model do
  it_behaves_like "auditable"
end
```

---

**S8. Write tests for a class that uses memoization with `||=`. Make sure to test the falsy value trap.**

```ruby
class ConfigLoader
  def initialize(store)
    @store = store
  end

  def feature_enabled?(feature)
    @_cache ||= {}
    unless @_cache.key?(feature)
      @_cache[feature] = @store.get("feature:#{feature}")
    end
    @_cache[feature]
  end
end

RSpec.describe ConfigLoader do
  let(:store)  { instance_double(Redis) }
  let(:loader) { ConfigLoader.new(store) }

  describe "#feature_enabled?" do
    context "when feature is enabled (true)" do
      before { allow(store).to receive(:get).with("feature:dark_mode").and_return(true) }

      it "returns true" do
        expect(loader.feature_enabled?(:dark_mode)).to be true
      end

      it "only calls the store once (cached)" do
        loader.feature_enabled?(:dark_mode)
        loader.feature_enabled?(:dark_mode)
        expect(store).to have_received(:get).once
      end
    end

    context "when feature is disabled (false)" do
      before { allow(store).to receive(:get).with("feature:beta").and_return(false) }

      it "returns false" do
        expect(loader.feature_enabled?(:beta)).to be false
      end

      it "only calls the store once even though value is false (key? check avoids falsy trap)" do
        loader.feature_enabled?(:beta)
        loader.feature_enabled?(:beta)
        # Would call TWICE with naive ||= because false is falsy
        expect(store).to have_received(:get).once
      end
    end

    context "when feature is not set (nil)" do
      before { allow(store).to receive(:get).and_return(nil) }

      it "returns nil and caches it" do
        loader.feature_enabled?(:missing)
        loader.feature_enabled?(:missing)
        expect(store).to have_received(:get).once
      end
    end
  end
end
```

---

**S9. Test a service object that processes a CSV file and returns structured results.**

```ruby
class CsvImporter
  ImportResult = Struct.new(:imported, :failed, :errors, keyword_init: true)
  ImportError  = Struct.new(:row, :message, keyword_init: true)

  def call(csv_path)
    imported = 0
    failed   = 0
    errors   = []

    CSV.foreach(csv_path, headers: true) do |row|
      User.create!(name: row["name"], email: row["email"])
      imported += 1
    rescue ActiveRecord::RecordInvalid => e
      failed += 1
      errors << ImportError.new(row: row.to_h, message: e.message)
    end

    ImportResult.new(imported: imported, failed: failed, errors: errors)
  end
end

RSpec.describe CsvImporter do
  subject(:importer) { CsvImporter.new }

  describe "#call" do
    context "with a valid CSV" do
      let(:csv_content) { "name,email\nAlice,alice@example.com\nBob,bob@example.com" }
      let(:csv_file)    { Tempfile.new(['test', '.csv']).tap { |f| f.write(csv_content); f.rewind } }

      it "imports all rows successfully" do
        result = importer.call(csv_file.path)
        expect(result.imported).to eq(2)
        expect(result.failed).to   eq(0)
        expect(result.errors).to   be_empty
      end

      it "creates User records in the database" do
        expect { importer.call(csv_file.path) }.to change { User.count }.by(2)
      end
    end

    context "with some invalid rows" do
      let(:csv_content) { "name,email\nAlice,alice@example.com\n,not-an-email" }
      let(:csv_file)    { Tempfile.new(['test', '.csv']).tap { |f| f.write(csv_content); f.rewind } }

      it "reports partial success" do
        result = importer.call(csv_file.path)
        expect(result.imported).to eq(1)
        expect(result.failed).to   eq(1)
      end

      it "includes error details for failed rows" do
        result = importer.call(csv_file.path)
        expect(result.errors.first).to have_attributes(
          row: { "name" => nil, "email" => "not-an-email" }
        )
      end
    end
  end
end
```

---

**S10. Test a pub/sub event system where subscribers receive messages.**

```ruby
RSpec.describe EventBus do
  let(:bus) { EventBus.new }

  describe "#subscribe and #publish" do
    it "delivers published events to subscribers" do
      received = []
      bus.subscribe(:order_created) { |event| received << event }
      bus.publish(:order_created, order_id: 42, amount: 100)

      expect(received).to contain_exactly(hash_including(order_id: 42))
    end

    it "delivers to multiple subscribers" do
      spy1 = spy("handler1")
      spy2 = spy("handler2")

      bus.subscribe(:user_registered, &spy1.method(:call))
      bus.subscribe(:user_registered, &spy2.method(:call))
      bus.publish(:user_registered, user_id: 1)

      expect(spy1).to have_received(:call).once
      expect(spy2).to have_received(:call).once
    end

    it "does not deliver to subscribers of different events" do
      payment_handler = spy("payment_handler")
      bus.subscribe(:order_cancelled, &payment_handler.method(:call))
      bus.publish(:order_created, order_id: 1)

      expect(payment_handler).not_to have_received(:call)
    end

    it "continues delivery even if one handler raises" do
      spy1 = spy("good_handler")
      bus.subscribe(:event) { raise "handler error" }
      bus.subscribe(:event, &spy1.method(:call))

      expect { bus.publish(:event, data: 1) }.not_to raise_error
      expect(spy1).to have_received(:call)
    end
  end
end
```

---

**S11. Write tests using Factory Bot traits for complex model scenarios.**

```ruby
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    name  { "Test User" }
    role  { :viewer }
    active { true }

    trait :admin do
      role  { :admin }
      name  { "Admin" }
    end

    trait :inactive do
      active { false }
    end

    trait :with_orders do
      after(:create) do |user|
        create_list(:order, 3, user: user)
      end
    end

    trait :with_recent_order do
      after(:create) do |user|
        create(:order, user: user, created_at: 1.day.ago)
      end
    end
  end
end

RSpec.describe UserPolicy do
  describe "admin permissions" do
    let(:admin) { create(:user, :admin) }
    let(:policy) { UserPolicy.new(admin) }

    it "can delete other users" do
      target = create(:user)
      expect(policy.delete?(target)).to be true
    end
  end

  describe "inactive user access" do
    let(:inactive) { create(:user, :inactive) }

    it "cannot log in" do
      expect(inactive.can_login?).to be false
    end
  end

  describe "user retention" do
    let(:active_buyer) { create(:user, :with_recent_order) }

    it "is marked as retained" do
      expect(active_buyer.retained?).to be true
    end
  end
end
```

---

**S12. Test a complex query object that chains scopes and filters.**

```ruby
class ProductQuery
  def initialize(params = {})
    @params = params
    @scope  = Product.all
  end

  def call
    filter_by_category
    filter_by_price_range
    sort_results
    @scope
  end

  private

  def filter_by_category
    return unless @params[:category]
    @scope = @scope.where(category: @params[:category])
  end

  def filter_by_price_range
    @scope = @scope.where("price >= ?", @params[:min_price]) if @params[:min_price]
    @scope = @scope.where("price <= ?", @params[:max_price]) if @params[:max_price]
  end

  def sort_results
    @scope = @scope.order(@params.fetch(:sort_by, "created_at") => @params.fetch(:direction, "desc"))
  end
end

RSpec.describe ProductQuery do
  let!(:cheap_electronics) { create(:product, category: "electronics", price: 50) }
  let!(:expensive_electronics) { create(:product, category: "electronics", price: 500) }
  let!(:cheap_clothing) { create(:product, category: "clothing", price: 30) }

  it "returns all products with no filters" do
    result = ProductQuery.new.call
    expect(result.count).to eq(3)
  end

  it "filters by category" do
    result = ProductQuery.new(category: "electronics").call
    expect(result).to include(cheap_electronics, expensive_electronics)
    expect(result).not_to include(cheap_clothing)
  end

  it "filters by price range" do
    result = ProductQuery.new(min_price: 40, max_price: 200).call
    expect(result).to include(cheap_electronics)
    expect(result).not_to include(cheap_clothing, expensive_electronics)
  end

  it "sorts by price ascending" do
    result = ProductQuery.new(sort_by: "price", direction: "asc").call
    expect(result.first.price).to be < result.last.price
  end
end
```

---

**S13. Write tests for a class using VCR cassettes for real HTTP interactions.**

```ruby
RSpec.describe GithubClient, :vcr do
  let(:client) { GithubClient.new(token: "test_token") }

  describe "#get_repo" do
    # First run: makes real HTTP call to Github API, records cassette
    # Subsequent runs: plays back recorded cassette
    it "returns repository details" do
      repo = client.get_repo("ruby/ruby")
      expect(repo[:name]).to eq("ruby")
      expect(repo[:language]).to eq("Ruby")
      expect(repo[:stargazers_count]).to be_a(Integer)
    end
  end

  describe "#create_issue" do
    it "creates an issue and returns it", vcr: { cassette_name: "github/create_issue" } do
      issue = client.create_issue(
        repo: "myorg/myrepo",
        title: "Test Issue",
        body: "Created by automated tests"
      )
      expect(issue[:number]).to be_a(Integer)
      expect(issue[:state]).to eq("open")
    end
  end

  describe "#list_repos" do
    it "paginates through all repos" do
      repos = client.list_repos("test_user")
      expect(repos).to be_an(Array)
      expect(repos.first[:name]).to be_a(String)
    end
  end
end
```

---

**S14. You have a model with callbacks. Test that callbacks fire at the right time and can be bypassed in tests.**

```ruby
class Order < ApplicationRecord
  before_create :assign_order_number
  after_create  :send_confirmation_email
  before_destroy :check_cancellable

  private

  def assign_order_number
    self.order_number = "ORD-#{SecureRandom.hex(4).upcase}"
  end

  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end

  def check_cancellable
    throw :abort unless status.in?(%w[pending processing])
  end
end

RSpec.describe Order do
  describe "callbacks" do
    describe "before_create" do
      it "assigns an order number" do
        order = create(:order)
        expect(order.order_number).to match(/\AORD-[A-F0-9]{8}\z/)
      end
    end

    describe "after_create" do
      it "enqueues a confirmation email" do
        expect { create(:order) }
          .to have_enqueued_mail(OrderMailer, :confirmation)
      end
    end

    describe "before_destroy" do
      it "prevents destruction of shipped orders" do
        order = create(:order, status: "shipped")
        expect { order.destroy }.not_to change { Order.count }
      end

      it "allows destruction of pending orders" do
        order = create(:order, status: "pending")
        expect { order.destroy }.to change { Order.count }.by(-1)
      end
    end

    # Bypassing callbacks in tests (when they're expensive and irrelevant):
    describe "expensive callback bypass" do
      it "creates without sending email" do
        order = build(:order)
        # Skip after_create callbacks
        expect { order.save!(validate: false) }
          .not_to have_enqueued_mail(OrderMailer, :confirmation)
        # OR: use update_columns / insert directly
      end
    end
  end
end
```

---

**S15. Test concurrent behavior and thread safety.**

```ruby
class Counter
  def initialize
    @mutex = Mutex.new
    @count = 0
  end

  def increment
    @mutex.synchronize { @count += 1 }
  end

  def value
    @mutex.synchronize { @count }
  end
end

RSpec.describe Counter do
  describe "thread safety" do
    it "handles concurrent increments correctly" do
      counter = Counter.new
      threads = 100.times.map do
        Thread.new { counter.increment }
      end
      threads.each(&:join)

      expect(counter.value).to eq(100)
    end

    it "is not thread-safe without mutex (demonstrates the bug)" do
      # This test documents what WOULD happen without the mutex
      # (non-deterministic — might pass sometimes due to scheduling)
      count = 0  # no mutex
      threads = 1000.times.map do
        Thread.new { count += 1 }  # race condition
      end
      threads.each(&:join)

      # count might NOT equal 1000 due to race conditions
      # We don't assert specific value here — just document the behavior
      expect(count).to be_between(1, 1000)
    end
  end
end
```
