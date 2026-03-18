# Chapter 07: Testing with RSpec — Structured Answers

## Answer 1: How to Structure a Comprehensive Test Suite for a Service Object

**Question:** "Walk me through how you'd design and write tests for a `CreateOrder` service object that validates, saves, charges a card, and sends a confirmation."

**Answer:**

```ruby
# The service under test
class CreateOrder
  OrderResult = Struct.new(:success?, :order, :errors, keyword_init: true)

  def initialize(payment_gateway: StripeGateway.new, mailer: OrderMailer)
    @payment_gateway = payment_gateway
    @mailer = mailer
  end

  def call(user:, items:, payment_token:)
    order = Order.new(user: user, items: items)
    return OrderResult.new(success?: false, order: order, errors: order.errors) unless order.valid?

    ActiveRecord::Base.transaction do
      order.save!
      charge = @payment_gateway.charge(
        amount_cents: order.total_cents,
        token: payment_token,
        description: "Order ##{order.id}"
      )
      order.update!(charge_id: charge.id, status: "confirmed")
      @mailer.confirmation(order).deliver_later
    end

    OrderResult.new(success?: true, order: order, errors: {})
  rescue PaymentGateway::CardDeclinedError => e
    order.update!(status: "payment_failed") if order.persisted?
    OrderResult.new(success?: false, order: order, errors: { payment: e.message })
  rescue ActiveRecord::RecordInvalid => e
    OrderResult.new(success?: false, order: nil, errors: { base: e.message })
  end
end

# Tests
RSpec.describe CreateOrder do
  let(:user)       { create(:user) }
  let(:item)       { create(:product, price_cents: 1000) }
  let(:gateway)    { instance_double(StripeGateway) }
  let(:mailer)     { class_spy(OrderMailer) }
  let(:mail_spy)   { spy("mail delivery") }
  let(:service)    { CreateOrder.new(payment_gateway: gateway, mailer: mailer) }

  let(:valid_params) { { user: user, items: [item], payment_token: "tok_visa" } }

  before do
    allow(mailer).to receive(:confirmation).and_return(mail_spy)
  end

  describe "#call" do
    context "happy path" do
      let(:charge_response) { double("Charge", id: "ch_test123") }

      before do
        allow(gateway).to receive(:charge).and_return(charge_response)
      end

      it "returns a successful result" do
        result = service.call(**valid_params)
        expect(result.success?).to be true
      end

      it "persists the order" do
        expect { service.call(**valid_params) }.to change { Order.count }.by(1)
      end

      it "sets the order status to confirmed" do
        result = service.call(**valid_params)
        expect(result.order.status).to eq("confirmed")
      end

      it "stores the charge ID" do
        result = service.call(**valid_params)
        expect(result.order.charge_id).to eq("ch_test123")
      end

      it "charges with the correct amount" do
        service.call(**valid_params)
        expect(gateway).to have_received(:charge).with(
          hash_including(amount_cents: 1000, token: "tok_visa")
        )
      end

      it "sends a confirmation email" do
        service.call(**valid_params)
        expect(mailer).to have_received(:confirmation)
        expect(mail_spy).to have_received(:deliver_later)
      end
    end

    context "when payment is declined" do
      before do
        allow(gateway).to receive(:charge)
          .and_raise(PaymentGateway::CardDeclinedError, "Insufficient funds")
      end

      it "returns a failed result" do
        result = service.call(**valid_params)
        expect(result.success?).to be false
      end

      it "includes the payment error" do
        result = service.call(**valid_params)
        expect(result.errors[:payment]).to include("Insufficient funds")
      end

      it "marks the order as payment_failed" do
        result = service.call(**valid_params)
        expect(result.order.status).to eq("payment_failed")
      end

      it "does not send a confirmation email" do
        service.call(**valid_params)
        expect(mail_spy).not_to have_received(:deliver_later)
      end
    end

    context "when order validation fails" do
      let(:invalid_params) { { user: user, items: [], payment_token: "tok_visa" } }

      it "returns a failed result with validation errors" do
        result = service.call(**invalid_params)
        expect(result.success?).to be false
        expect(result.errors).not_to be_empty
      end

      it "does not charge the card" do
        service.call(**invalid_params)
        expect(gateway).not_to have_received(:charge)
      end
    end
  end
end
```

---

## Answer 2: Properly Using Doubles, Spies, and Verifying Doubles

**Question:** "When should you use `double` vs `instance_double` vs `spy`? Show a concrete example of each."

**Answer:**

```ruby
# 1. double — use when the real class doesn't exist yet or is external
#    No verification against real class interface
http_response = double("HTTPResponse",
  status: 200,
  body: '{"data": []}',
  headers: { "Content-Type" => "application/json" }
)

# 2. instance_double — use for any class that exists in your codebase
#    Verifies method names and argument counts against the real class
class UserRepository
  def find(id)       = User.find(id)
  def save(user)     = user.save!
  def delete(id)     = User.destroy(id)
end

repo = instance_double(UserRepository)
allow(repo).to receive(:find).with(42).and_return(build_stubbed(:user))
allow(repo).to receive(:nonexistent)  # => raises: method doesn't exist on UserRepository

# 3. spy — use when you want to assert afterward AND the double should accept any message
class AuditService
  def log(action, metadata = {}) = AuditLog.create!(action: action, metadata: metadata)
  def alert(message)             = SlackNotifier.send(message)
end

audit_spy = instance_spy(AuditService)  # verifying spy

service.process(data)

# Assert after the fact:
expect(audit_spy).to have_received(:log).with(:data_processed, anything)
expect(audit_spy).not_to have_received(:alert)  # no alerts for normal processing

# 4. Practical choosing guide:
#
# "I want to control return value and assert it's called"
#   → allow + expect, or instance_double
#
# "I want to assert it was called but don't care about timing"
#   → spy or instance_spy
#
# "The class doesn't exist yet / is external"
#   → double with explicit message expectations
#
# "I need verification that my stub matches the real interface"
#   → always use instance_double / class_double / instance_spy
```

---

## Answer 3: Writing Tests for Code That Mutates and Queries State

**Question:** "Show examples of the `change` matcher in complex scenarios including nested changes, compound matchers, and negative assertions."

**Answer:**

```ruby
RSpec.describe Account do
  let(:user)    { create(:user) }
  let(:account) { create(:account, user: user, balance: 1000) }

  describe "#withdraw" do
    it "decreases the balance by the withdrawal amount" do
      expect { account.withdraw(200) }
        .to change { account.reload.balance }.from(1000).to(800)
    end

    it "records the transaction" do
      expect { account.withdraw(200) }
        .to change { Transaction.count }.by(1)
    end

    it "does both at once (compound change)" do
      expect { account.withdraw(200) }
        .to change { account.reload.balance }.by(-200)
        .and change { Transaction.count }.by(1)
    end

    it "does not change balance on invalid withdrawal" do
      expect { account.withdraw(5000) }  # more than balance
        .not_to change { account.reload.balance }
    end

    it "changes status when balance hits zero" do
      expect { account.withdraw(1000) }
        .to change { account.reload.balance }.to(0)
        .and change { account.reload.status }.to("zero_balance")
    end
  end

  describe "#transfer_to" do
    let(:other_account) { create(:account, user: create(:user), balance: 500) }

    it "moves money between accounts" do
      expect { account.transfer_to(other_account, 300) }
        .to change { account.reload.balance }.by(-300)
        .and change { other_account.reload.balance }.by(300)
    end

    it "is atomic — either both change or neither does" do
      allow(other_account).to receive(:deposit!).and_raise("DB error")

      expect { account.transfer_to(other_account, 300) rescue nil }
        .not_to change { account.reload.balance }
    end
  end
end
```

---

## Answer 4: `shared_examples` and `shared_context` in a Real Rails App

**Question:** "Show me how you use shared examples and shared context to DRY up your test suite for a JSON API."

**Answer:**

```ruby
# spec/support/shared_contexts/authenticated.rb
RSpec.shared_context "as authenticated user" do
  let(:current_user) { create(:user) }
  let(:auth_headers) do
    token = JWT.encode({ sub: current_user.id }, Rails.application.secret_key_base)
    { "Authorization" => "Bearer #{token}" }
  end
end

RSpec.shared_context "as admin user" do
  include_context "as authenticated user"
  before { current_user.update!(role: :admin) }
end

# spec/support/shared_examples/json_api_endpoint.rb
RSpec.shared_examples "requires authentication" do
  context "without auth token" do
    it "returns 401" do
      subject  # the request
      expect(response).to have_http_status(:unauthorized)
    end
  end
end

RSpec.shared_examples "a paginated endpoint" do |factory, per_page: 10|
  before { create_list(factory, per_page + 2) }

  it "paginates results" do
    subject
    data = JSON.parse(response.body)
    expect(data["data"].length).to be <= per_page
  end

  it "includes pagination metadata" do
    subject
    meta = JSON.parse(response.body)["meta"]
    expect(meta.keys).to include("total", "page", "per_page")
  end
end

# Usage:
RSpec.describe "GET /api/v1/articles", type: :request do
  subject { get "/api/v1/articles", headers: auth_headers }

  include_context "as authenticated user"

  it_behaves_like "requires authentication" do
    subject { get "/api/v1/articles" }  # no headers
  end

  it_behaves_like "a paginated endpoint", :article

  it "returns only published articles" do
    create(:article, :published)
    create(:article, :draft)
    subject
    data = JSON.parse(response.body)["data"]
    expect(data.all? { |a| a["published"] }).to be true
  end
end

RSpec.describe "GET /api/v1/users", type: :request do
  subject { get "/api/v1/users", headers: auth_headers }

  include_context "as admin user"

  it_behaves_like "a paginated endpoint", :user
end
```

---

## Answer 5: Properly Testing ActiveRecord Associations and Scopes

**Question:** "How do you test model associations, scopes, and validations efficiently?"

**Answer:**

```ruby
RSpec.describe Article, type: :model do
  # Associations
  describe "associations" do
    it { is_expected.to belong_to(:author).class_name("User") }
    it { is_expected.to have_many(:comments).dependent(:destroy) }
    it { is_expected.to have_many(:tags).through(:article_tags) }
  end

  # Validations
  describe "validations" do
    subject { build(:article) }

    it { is_expected.to validate_presence_of(:title) }
    it { is_expected.to validate_length_of(:title).is_at_most(200) }
    it { is_expected.to validate_presence_of(:body) }
    it { is_expected.to validate_uniqueness_of(:slug) }

    context "custom validator" do
      it "rejects titles with profanity" do
        article = build(:article, title: "Bad word example")
        allow(ProfanityFilter).to receive(:clean?).with("Bad word example").and_return(false)
        expect(article).not_to be_valid
        expect(article.errors[:title]).to include("contains inappropriate language")
      end
    end
  end

  # Scopes
  describe "scopes" do
    let!(:published) { create_list(:article, 3, published_at: 1.day.ago, status: "published") }
    let!(:draft)     { create_list(:article, 2, published_at: nil, status: "draft") }
    let!(:old)       { create(:article, published_at: 1.year.ago, status: "published") }

    describe ".published" do
      it "returns only published articles" do
        expect(Article.published).to match_array(published + [old])
      end
    end

    describe ".recent" do
      it "returns articles from the last 30 days" do
        expect(Article.recent).to match_array(published)
        expect(Article.recent).not_to include(old)
      end
    end

    describe ".published.recent" do
      it "chains correctly" do
        results = Article.published.recent
        expect(results).to match_array(published)
      end
    end
  end

  # Custom methods
  describe "#reading_time" do
    it "estimates 200 words per minute" do
      article = build(:article, body: "word " * 400)  # 400 words
      expect(article.reading_time).to eq(2)  # 2 minutes
    end

    it "returns minimum of 1 minute" do
      article = build(:article, body: "short")
      expect(article.reading_time).to eq(1)
    end
  end
end
```

---

## Answer 6: Testing Error Handling and Exception Paths Thoroughly

**Question:** "How do you test error handling, retries, and circuit breakers?"

**Answer:**

```ruby
class ResilientApiClient
  MAX_RETRIES = 3

  def get(url)
    attempts = 0
    begin
      attempts += 1
      make_request(url)
    rescue Net::TimeoutError => e
      retry if attempts < MAX_RETRIES
      raise ServiceUnavailableError, "Service failed after #{MAX_RETRIES} attempts: #{e.message}"
    rescue Net::HTTPBadResponse => e
      raise InvalidResponseError, "Bad response: #{e.message}"
    end
  end

  private

  def make_request(url)
    # real HTTP call
  end
end

RSpec.describe ResilientApiClient do
  let(:client) { ResilientApiClient.new }

  describe "#get" do
    context "on timeout" do
      before do
        # Fail twice, succeed third time
        allow(client).to receive(:make_request)
          .and_raise(Net::TimeoutError, "timed out")
          .and_raise(Net::TimeoutError, "timed out")
          .and_return({ status: 200, body: "ok" })
      end

      it "retries up to MAX_RETRIES times" do
        result = client.get("https://example.com/api")
        expect(result[:status]).to eq(200)
        expect(client).to have_received(:make_request).exactly(3).times
      end
    end

    context "when all retries are exhausted" do
      before do
        allow(client).to receive(:make_request)
          .and_raise(Net::TimeoutError, "persistent timeout")
      end

      it "raises ServiceUnavailableError" do
        expect { client.get("https://example.com/api") }
          .to raise_error(ServiceUnavailableError, /failed after 3 attempts/)
      end

      it "attempts exactly MAX_RETRIES times" do
        client.get("https://example.com/api") rescue nil
        expect(client).to have_received(:make_request).exactly(3).times
      end
    end

    context "on bad response (no retry)" do
      before do
        allow(client).to receive(:make_request)
          .and_raise(Net::HTTPBadResponse, "malformed")
      end

      it "raises InvalidResponseError immediately without retrying" do
        expect { client.get("https://example.com/api") }
          .to raise_error(InvalidResponseError)
        expect(client).to have_received(:make_request).once  # no retry
      end
    end
  end
end
```

---

## Answer 7: How to Test Private Methods (and Why You Usually Shouldn't)

**Question:** "Should you test private methods? How do you do it when necessary?"

**Answer:**

```ruby
class CreditScoreCalculator
  def score(user)
    base = calculate_base_score(user)
    adjust_for_history(base, user)
  end

  private

  def calculate_base_score(user)
    # complex calculation
  end

  def adjust_for_history(base, user)
    # more complex logic
  end
end

# Philosophy: Private methods should be tested INDIRECTLY through public interface
RSpec.describe CreditScoreCalculator do
  let(:calc) { CreditScoreCalculator.new }

  # PREFERRED: test via public interface
  it "returns higher score for users with no missed payments" do
    good_user = build_stubbed(:user, missed_payments: 0)
    bad_user  = build_stubbed(:user, missed_payments: 5)
    expect(calc.score(good_user)).to be > calc.score(bad_user)
  end

  # WHEN you must test private methods (extracted logic, complex rules):
  # Option 1: send (bypasses visibility)
  it "base score is 700 for default user" do
    user = build_stubbed(:user)
    base = calc.send(:calculate_base_score, user)
    expect(base).to eq(700)
  end

  # Option 2: extract to a separate class/module
  # CreditScore::BaseScoreCalculator — now testable as public interface
  # (usually the right refactoring)

  # Option 3: use .instance_eval to access private context
  it "adjustment reduces score for late payments" do
    user = build_stubbed(:user, late_payments: 2)
    calc.instance_eval do
      base = 700
      adjusted = adjust_for_history(base, user)
      expect(adjusted).to be < base
    end
  end

  # Rule: if you're frequently needing to test private methods,
  # consider extracting them into a new class with a public interface
end
```

---

## Answer 8: Configuring RSpec for a Full Rails Project

**Question:** "Walk me through how you'd configure RSpec for a new Rails project with database cleaning, factory support, and shared helpers."

**Answer:**

```ruby
# spec/rails_helper.rb
require 'spec_helper'
require File.expand_path('../config/environment', __dir__)
require 'rspec/rails'
require 'database_cleaner/active_record'

Dir[Rails.root.join('spec', 'support', '**', '*.rb')].each { |f| require f }

RSpec.configure do |config|
  # Factory Bot
  config.include FactoryBot::Syntax::Methods

  # Helpers by type
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Rails.application.routes.url_helpers, type: :request

  # Database Cleaner
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, js: true) do
    DatabaseCleaner.strategy = :truncation
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end

  # Parallel specs — ensure correct DB cleaning
  config.before(:suite) do
    if ParallelTests.first_process?
      DatabaseCleaner.clean_with(:truncation)
    end
  end

  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!

  # Run focused specs with: rspec --tag focus
  config.filter_run_when_matching :focus

  # Randomize order to catch ordering dependencies
  config.order = :random
  Kernel.srand(config.seed)
end

# spec/spec_helper.rb
require 'simplecov'
SimpleCov.start 'rails' do
  minimum_coverage 85
  add_filter '/spec/'
end

RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true  # important for catching bad stubs on real objects
    mocks.verify_doubled_constant_names = true  # fail if instance_double class doesn't exist
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
end

# spec/support/shared_contexts/api_helpers.rb
RSpec.shared_context "json api helpers" do
  def json_response
    JSON.parse(response.body, symbolize_names: true)
  end

  def json_headers
    { "Content-Type" => "application/json", "Accept" => "application/json" }
  end
end

RSpec.configure do |config|
  config.include_context "json api helpers", type: :request
end
```

---

## Answer 9: Testing Concerns and Polymorphic Associations

**Question:** "How do you test a concern that's included in multiple models, and how do you handle polymorphic associations in tests?"

**Answer:**

```ruby
# spec/support/shared_examples/taggable.rb
RSpec.shared_examples "taggable" do
  let(:record) { create(described_class.name.underscore.to_sym) }

  describe "#add_tag" do
    it "adds a tag to the record" do
      record.add_tag("ruby")
      expect(record.tags.pluck(:name)).to include("ruby")
    end

    it "does not duplicate tags" do
      record.add_tag("ruby")
      record.add_tag("ruby")
      expect(record.tags.where(name: "ruby").count).to eq(1)
    end
  end

  describe "#remove_tag" do
    before { record.add_tag("ruby") }

    it "removes the tag" do
      record.remove_tag("ruby")
      expect(record.tags.pluck(:name)).not_to include("ruby")
    end
  end

  describe ".tagged_with" do
    before { record.add_tag("interview") }

    it "finds records with the specified tag" do
      expect(described_class.tagged_with("interview")).to include(record)
    end

    it "does not return records without the tag" do
      other = create(described_class.name.underscore.to_sym)
      expect(described_class.tagged_with("interview")).not_to include(other)
    end
  end
end

RSpec.describe Article, type: :model do
  it_behaves_like "taggable"
end

RSpec.describe Question, type: :model do
  it_behaves_like "taggable"
end

# Testing polymorphic association
RSpec.describe Comment, type: :model do
  it "belongs to commentable (polymorphic)" do
    article  = create(:article)
    question = create(:question)

    article_comment  = create(:comment, commentable: article)
    question_comment = create(:comment, commentable: question)

    expect(article_comment.commentable_type).to  eq("Article")
    expect(question_comment.commentable_type).to eq("Question")
    expect(article_comment.commentable).to  eq(article)
    expect(question_comment.commentable).to eq(question)
  end
end
```

---

## Answer 10: Testing Async/Background Job Integrations

**Question:** "How do you test code that enqueues jobs, and how do you test the jobs themselves?"

**Answer:**

```ruby
# Testing that a job is enqueued (from a service/controller)
RSpec.describe UserRegistrationService do
  include ActiveJob::TestHelper

  let(:service) { UserRegistrationService.new }

  it "enqueues a welcome email job" do
    expect {
      service.register(name: "Alice", email: "alice@example.com")
    }.to have_enqueued_job(WelcomeEmailJob)
  end

  it "enqueues with correct arguments" do
    user = service.register(name: "Alice", email: "alice@example.com")
    expect(WelcomeEmailJob).to have_been_enqueued.with(user.id)
  end

  it "schedules the job to run after 5 minutes" do
    service.register(name: "Alice", email: "alice@example.com")
    expect(WelcomeEmailJob).to have_been_enqueued
      .at(5.minutes.from_now)
  end
end

# Testing the job itself in isolation
RSpec.describe WelcomeEmailJob do
  include ActiveJob::TestHelper

  let(:user) { create(:user) }

  describe "#perform" do
    it "sends the welcome email" do
      expect { WelcomeEmailJob.perform_now(user.id) }
        .to have_enqueued_mail(UserMailer, :welcome).with(user)
    end

    it "updates the user's welcomed_at timestamp" do
      expect { WelcomeEmailJob.perform_now(user.id) }
        .to change { user.reload.welcomed_at }.from(nil)
    end

    it "handles missing user gracefully" do
      expect { WelcomeEmailJob.perform_now(999999) }
        .not_to raise_error
    end
  end
end

# Testing with perform_enqueued_jobs (integration style)
RSpec.describe "Registration flow", type: :request do
  include ActiveJob::TestHelper

  it "sends email after registration" do
    perform_enqueued_jobs do
      post "/api/v1/registrations", params: { name: "Alice", email: "alice@example.com" }
    end

    # All enqueued jobs have now run
    expect(ActionMailer::Base.deliveries.last.to).to include("alice@example.com")
  end
end
```
