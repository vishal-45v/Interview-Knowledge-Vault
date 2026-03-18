# Chapter 09 — Testing Rails: Structured Answers

---

## Answer 1: Complete Request Spec for API Endpoint

**Question:** Write a comprehensive request spec for a `PostsController#create` API endpoint with authentication, validation, and side effects.

```ruby
# spec/requests/api/v1/posts_spec.rb
require 'rails_helper'

RSpec.describe "POST /api/v1/posts", type: :request do
  let(:user)    { create(:user) }
  let(:headers) { auth_headers(user) }  # Helper generates JWT token

  describe "POST /api/v1/posts" do
    let(:valid_params) do
      {
        post: {
          title: "Rails Testing Best Practices",
          body:  "Here is what I've learned...",
          tags:  ["rails", "testing"]
        }
      }
    end

    context "with valid params and authenticated user" do
      it "creates a post and returns 201" do
        expect {
          post api_v1_posts_path, params: valid_params, headers: headers, as: :json
        }.to change(Post, :count).by(1)

        expect(response).to have_http_status(:created)
      end

      it "returns the created post data" do
        post api_v1_posts_path, params: valid_params, headers: headers, as: :json

        json = JSON.parse(response.body)
        expect(json['data']['title']).to eq("Rails Testing Best Practices")
        expect(json['data']['author_id']).to eq(user.id)
      end

      it "enqueues the post indexing job" do
        expect {
          post api_v1_posts_path, params: valid_params, headers: headers, as: :json
        }.to have_enqueued_job(IndexPostJob)
      end

      it "enqueues the notification email" do
        user.followers.each { |f| create(:follow_notification_preference, user: f) }
        expect {
          post api_v1_posts_path, params: valid_params, headers: headers, as: :json
        }.to have_enqueued_mail(FollowerNotificationMailer, :new_post)
      end
    end

    context "with invalid params" do
      let(:invalid_params) { { post: { title: "", body: "Body without title" } } }

      it "returns 422 with error details" do
        post api_v1_posts_path, params: invalid_params, headers: headers, as: :json

        expect(response).to have_http_status(:unprocessable_entity)
        json = JSON.parse(response.body)
        expect(json['error']['code']).to eq('validation_failed')
        expect(json['error']['details']).to have_key('title')
      end

      it "does not create a post" do
        expect {
          post api_v1_posts_path, params: invalid_params, headers: headers, as: :json
        }.not_to change(Post, :count)
      end
    end

    context "without authentication" do
      it "returns 401" do
        post api_v1_posts_path, params: valid_params, as: :json
        expect(response).to have_http_status(:unauthorized)
      end
    end

    context "with expired token" do
      let(:expired_headers) { auth_headers(user, expires_in: -1.hour) }

      it "returns 401 with token_expired error code" do
        post api_v1_posts_path, params: valid_params, headers: expired_headers, as: :json

        expect(response).to have_http_status(:unauthorized)
        expect(JSON.parse(response.body)['error']['code']).to eq('token_expired')
      end
    end
  end
end

# spec/support/auth_helpers.rb
module AuthHelpers
  def auth_headers(user, expires_in: 15.minutes)
    token = JwtService.encode_access(user_id: user.id, exp: expires_in.from_now.to_i)
    { 'Authorization' => "Bearer #{token}" }
  end
end

RSpec.configure do |config|
  config.include AuthHelpers, type: :request
end
```

---

## Answer 2: Complete Pundit Policy Spec

**Question:** Write a comprehensive spec for a Pundit policy.

```ruby
# spec/policies/post_policy_spec.rb
require 'rails_helper'

RSpec.describe PostPolicy, type: :policy do
  subject { described_class }

  let(:user)    { create(:user) }
  let(:admin)   { create(:user, :admin) }
  let(:owner)   { create(:user) }
  let(:post)    { create(:post, user: owner, published: false) }
  let(:published_post) { create(:post, user: owner, published: true) }

  permissions :show? do
    context "with a draft post" do
      it "denies guest users" do
        expect(subject).not_to permit(nil, post)
      end

      it "denies other users" do
        expect(subject).not_to permit(user, post)
      end

      it "permits the owner" do
        expect(subject).to permit(owner, post)
      end

      it "permits admin users" do
        expect(subject).to permit(admin, post)
      end
    end

    context "with a published post" do
      it "permits any user" do
        expect(subject).to permit(user, published_post)
      end

      it "permits guests" do
        expect(subject).to permit(nil, published_post)
      end
    end
  end

  permissions :update? do
    it "denies guest users" do
      expect(subject).not_to permit(nil, post)
    end

    it "denies other users" do
      expect(subject).not_to permit(user, post)
    end

    it "permits the owner if they have editor role" do
      owner.roles << create(:role, name: 'editor')
      expect(subject).to permit(owner, post)
    end

    it "denies the owner without editor role" do
      expect(subject).not_to permit(owner, post)
    end

    it "permits admin users" do
      expect(subject).to permit(admin, post)
    end
  end

  permissions :destroy? do
    it "permits only the owner" do
      expect(subject).to permit(owner, post)
    end

    it "permits admin" do
      expect(subject).to permit(admin, post)
    end

    it "denies other users" do
      expect(subject).not_to permit(user, post)
    end
  end

  describe PostPolicy::Scope do
    let(:scope) { PostPolicy::Scope.new(current_user, Post.all) }

    context "as admin" do
      let(:current_user) { admin }

      it "returns all posts including drafts" do
        create(:post, published: false)
        create(:post, published: true)
        expect(scope.resolve.count).to eq(Post.count)
      end
    end

    context "as regular user" do
      let(:current_user) { user }

      it "returns only published posts and own drafts" do
        other_draft = create(:post, published: false, user: owner)
        own_draft   = create(:post, published: false, user: user)
        published   = create(:post, published: true)

        resolved = scope.resolve
        expect(resolved).to include(own_draft, published)
        expect(resolved).not_to include(other_draft)
      end
    end

    context "as guest (nil user)" do
      let(:current_user) { nil }

      it "returns only published posts" do
        draft     = create(:post, published: false)
        published = create(:post, published: true)

        expect(scope.resolve).to include(published)
        expect(scope.resolve).not_to include(draft)
      end
    end
  end
end
```

---

## Answer 3: Service Object Test with External API Mocking

**Question:** Write a comprehensive unit test for a service that calls an external API with various failure modes.

```ruby
# spec/services/github_stats_service_spec.rb
require 'rails_helper'

RSpec.describe GithubStatsService do
  subject(:service) { described_class.new(user) }
  let(:user) { create(:user, github_username: 'alice') }

  describe "#fetch_stats" do
    context "on successful API response" do
      before do
        stub_request(:get, "https://api.github.com/users/alice")
          .with(headers: { 'Authorization' => /Bearer .+/ })
          .to_return(
            status: 200,
            body:   { login: 'alice', public_repos: 42, followers: 100 }.to_json,
            headers: { 'Content-Type' => 'application/json' }
          )
      end

      it "returns parsed stats" do
        stats = service.fetch_stats
        expect(stats).to eq({ repos: 42, followers: 100, username: 'alice' })
      end

      it "caches the result" do
        service.fetch_stats
        service.fetch_stats  # Second call should use cache
        expect(WebMock).to have_requested(:get, /github\.com/).once
      end
    end

    context "when rate limited" do
      before do
        stub_request(:get, "https://api.github.com/users/alice")
          .to_return(
            status: 429,
            body:   { message: "API rate limit exceeded" }.to_json,
            headers: {
              'Content-Type' => 'application/json',
              'Retry-After' => '3600'
            }
          )
      end

      it "raises GithubRateLimitError" do
        expect { service.fetch_stats }.to raise_error(GithubStatsService::RateLimitError)
      end

      it "includes retry-after time in error" do
        begin
          service.fetch_stats
        rescue GithubStatsService::RateLimitError => e
          expect(e.retry_after).to eq(3600)
        end
      end
    end

    context "when user not found" do
      before do
        stub_request(:get, "https://api.github.com/users/alice")
          .to_return(status: 404, body: { message: "Not Found" }.to_json)
      end

      it "raises GithubUserNotFoundError" do
        expect { service.fetch_stats }.to raise_error(GithubStatsService::UserNotFoundError)
      end
    end

    context "on network timeout" do
      before do
        stub_request(:get, "https://api.github.com/users/alice")
          .to_timeout
      end

      it "raises GithubConnectionError" do
        expect { service.fetch_stats }.to raise_error(GithubStatsService::ConnectionError)
      end
    end

    context "when user has no GitHub username" do
      let(:user) { create(:user, github_username: nil) }

      it "raises ArgumentError" do
        expect { service.fetch_stats }.to raise_error(ArgumentError, /github_username/)
      end

      it "does not make any HTTP requests" do
        begin; service.fetch_stats; rescue ArgumentError; end
        expect(WebMock).not_to have_requested(:get, /github\.com/)
      end
    end
  end
end
```

---

## Answer 4: System Test for Multi-Step Wizard

**Question:** Write a system test for a 3-step onboarding wizard.

```ruby
# spec/system/onboarding_spec.rb
require 'rails_helper'

RSpec.describe "Onboarding Wizard", type: :system do
  let(:user) { create(:user, onboarding_completed: false) }

  before do
    sign_in user  # Capybara sign_in helper
    visit onboarding_path
  end

  describe "completing the full onboarding flow" do
    it "saves profile, workspace, and preferences and redirects to dashboard" do
      # Step 1: Profile
      expect(page).to have_css('[data-step="profile"]')
      expect(page).to have_text("Tell us about yourself")

      fill_in "Full name",   with: "Alice Smith"
      fill_in "Job title",   with: "Software Engineer"
      select  "Engineering", from: "Department"
      click_button "Next"

      # Step 2: Workspace
      expect(page).to have_css('[data-step="workspace"]')
      expect(page).to have_text("Set up your workspace")

      fill_in "Workspace name", with: "Alice's Projects"
      choose  "Team"  # Radio button: Personal or Team
      click_button "Next"

      # Step 3: Preferences
      expect(page).to have_css('[data-step="preferences"]')
      check "Email digest"
      uncheck "Marketing emails"
      select "Pacific Time", from: "Timezone"
      click_button "Finish Setup"

      # Final state
      expect(page).to have_current_path(dashboard_path)
      expect(page).to have_text("Welcome, Alice Smith!")

      user.reload
      expect(user.full_name).to eq("Alice Smith")
      expect(user.onboarding_completed).to be true
      expect(user.workspace.name).to eq("Alice's Projects")
      expect(user.preferences.email_digest).to be true
      expect(user.preferences.timezone).to eq("Pacific Time")
    end

    it "preserves data when going back to a previous step" do
      fill_in "Full name", with: "Alice Smith"
      click_button "Next"

      fill_in "Workspace name", with: "Alice's Projects"
      click_button "Back"

      # Back to step 1 — data preserved
      expect(find_field("Full name").value).to eq("Alice Smith")
    end
  end

  describe "validation errors" do
    it "shows error when name is blank" do
      click_button "Next"  # Without filling in name

      expect(page).to have_css('[data-step="profile"]')  # Still on step 1
      expect(page).to have_text("Name can't be blank")
    end
  end

  describe "with JavaScript interactions", js: true do
    it "shows character count for bio field" do
      fill_in "Bio", with: "I am a software engineer"
      expect(page).to have_text("23 / 200 characters")
    end
  end
end
```

---

## Answer 5: FactoryBot with Traits and Sequences

**Question:** Design factories for a complex domain model with minimal database hits.

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    sequence(:username) { |n| "user_#{n}" }
    first_name { Faker::Name.first_name }
    last_name  { Faker::Name.last_name }
    password   { "password123" }
    confirmed_at { Time.current }

    # Traits for roles
    trait :admin do
      after(:create) do |user|
        user.roles << FactoryBot.create(:role, name: 'admin')
      end
    end

    trait :editor do
      after(:create) do |user|
        user.roles << FactoryBot.create(:role, name: 'editor')
      end
    end

    # State traits
    trait :unconfirmed do
      confirmed_at { nil }
    end

    trait :locked do
      locked_at           { 1.hour.ago }
      failed_attempts     { 10 }
    end

    trait :with_avatar do
      after(:build) do |user|
        user.avatar.attach(
          io:       File.open(Rails.root.join('spec', 'fixtures', 'avatar.jpg')),
          filename: 'avatar.jpg',
          content_type: 'image/jpeg'
        )
      end
    end

    # For pure unit tests — no DB needed
    factory :stubbed_user, class: 'User' do
      skip_create
      initialize_with { new(attributes) }
    end
  end
end

# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    sequence(:title) { |n| "Post Title #{n}" }
    body             { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    published        { false }
    association :user

    trait :published do
      published    { true }
      published_at { 1.day.ago }
    end

    trait :with_comments do
      transient do
        comment_count { 3 }
      end

      after(:create) do |post, evaluator|
        create_list(:comment, evaluator.comment_count, post: post)
      end
    end

    trait :featured do
      published
      featured     { true }
      featured_at  { 1.hour.ago }
    end
  end
end

# Usage examples:
# create(:user) — basic user
# create(:user, :admin) — admin user
# create(:user, :admin, :locked) — admin AND locked
# create(:post, :published, :with_comments, comment_count: 5, user: admin_user)
# build_stubbed(:user) — no DB, for unit tests
```

---
