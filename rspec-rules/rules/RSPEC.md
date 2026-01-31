---
paths:
  - "spec/**/*.rb"
  - "test/**/*.rb"
---

# RSpec General Rules

## Stack
### RSpec

## Gemfile
```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end

group :test do
  gem "letter_opener"
  gem "capybara"
  gem "selenium-webdriver"
  gem "shoulda-matchers"
  gem "simplecov"
  gem "vcr"
  gem "webmock"
  gem "database_cleaner-active_record"
  gem "timecop"
end
```

## Integration Tests
Use Capybara for integration tests. See rules/CAPYBARA.md

## Controller tests

- If a controller endpoint is a view endpoint, there is no need to test it. Rely on integration tests.
- If a controller is an API endpoint, it should be tested.

## Unit Tests
see rules/UNIT_TESTS.md

### Shoulda Matchers
Use shoulda-matchers to verify validations and associations are configured correctly:

```ruby
# spec/rails_helper.rb
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

```ruby
# GOOD - shoulda-matchers for framework configuration
RSpec.describe User do
  it { is_expected.to validate_presence_of(:email) }
  it { is_expected.to have_many(:orders).dependent(:destroy) }
  it { is_expected.to belong_to(:organization) }
end

# GOOD - Test business logic separately
it "requires email for account activation" do
  user = FactoryBot.create(:user, email: nil)
  expect(user.can_activate?).to be false
end
```

## External HTTP APIs with VCR

Use VCR to record and replay HTTP interactions:

```ruby
# spec/support/vcr.rb
VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data("<API_KEY>") { ENV["API_KEY"] }
  config.default_cassette_options = { re_record_interval: 30.days }
end
```

```ruby
# Record once, replay forever
it "fetches user data", :vcr do
  response = ExternalApi.fetch_user(1)
  expect(response.name).to eq("John")
end

# Or explicitly name cassettes
it "handles API errors" do
  VCR.use_cassette("api_error_response") do
    expect { ExternalApi.fetch_user(999) }.to raise_error(ApiError)
  end
end
```

## Email Testing with letter_opener

Use letter_opener to capture emails in tests:

```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test

# In tests, use ActionMailer::Base.deliveries
it "sends welcome email" do
  expect {
    UserMailer.welcome(user).deliver_now
  }.to change { ActionMailer::Base.deliveries.count }.by(1)

  email = ActionMailer::Base.deliveries.last
  expect(email.to).to include(user.email)
  expect(email.subject).to eq("Welcome!")
end

# Or use have_enqueued_mail for async
it "queues confirmation email" do
  expect {
    user.request_confirmation
  }.to have_enqueued_mail(UserMailer, :confirmation)
end
```

## Time Management with Timecop

Use timecop for time-sensitive tests:

```ruby
# Freeze time
it "expires after 24 hours" do
  token = create(:token)
  Timecop.freeze(25.hours.from_now) do
    expect(token).to be_expired
  end
end

# Travel to specific time
it "calculates billing at month end" do
  Timecop.travel(Date.new(2024, 1, 31)) do
    expect(invoice.due_date).to eq(Date.new(2024, 2, 28))
  end
end

# Always return to present in after block
after { Timecop.return }
```

## Database Cleaner Setup

Use database_cleaner for consistent test state:

```ruby
# spec/support/database_cleaner.rb
RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
```

## Test Coverage
Use SimpleCov for test coverage:

```ruby
# spec/spec_helper.rb (at the very top, before any other requires)
require "simplecov"
SimpleCov.start "rails"
```
