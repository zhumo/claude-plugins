---
paths:
  - "spec/**/*.rb"
  - "test/**/*.rb"
---

# Writing RSpec Model Tests

## Critical Anti-Patterns to Avoid

### 1. BANNED: Mocks and Stubs

Mocks are banned in this codebase. They hide bugs and test the mock library instead of your code.

```ruby
# BANNED - never use mocks
allow(order).to receive(:calculate_total).and_return(100)
instance_double(User, full_name: "John Doe")
expect(service).to have_received(:call)

# GOOD - test real objects with real data
it "applies discount" do
  order = FactoryBot.create(:order, items: [FactoryBot.create(:item, price: 100)])
  order.apply_discount(10)
  expect(order.final_price).to eq(90)
end
```
### 3. Testing Implementation Instead of Behavior

Tests should verify WHAT the code does, not HOW it does it.

```ruby
# BAD - Coupled to internal structure
it "stores customer type as integer" do
  customer = FactoryBot.create(:customer, :premium)
  expect(customer.type_code).to eq(2)
end

# GOOD - Test observable behavior
it "identifies premium customers" do
  customer = FactoryBot.create(:customer, :premium)
  expect(customer).to be_premium
  expect(customer.discount_rate).to eq(0.15)
end
```

### 4. Happy Path Only

Test edge cases, error conditions, and boundaries—not just the golden path.

```ruby
# BAD - Only tests success
it "creates an order" do
  order = FactoryBot.create(:order, items: [item])
  expect(order).to be_valid
end

# GOOD - Test edge cases
describe "#place" do
  it "succeeds with valid items" do
    order = FactoryBot.build(:order, items: [FactoryBot.create(:item)])
    expect(order.place).to be true
  end

  it "fails when inventory is insufficient" do
    item = FactoryBot.create(:item, stock: 0)
    order = FactoryBot.build(:order, items: [item])
    expect(order.place).to be false
    expect(order.errors[:base]).to include("Insufficient stock")
  end

  it "fails with empty cart" do
    order = FactoryBot.build(:order, items: [])
    expect(order.place).to be false
  end

  it "handles nil items gracefully" do
    order = FactoryBot.build(:order, items: nil)
    expect(order.place).to be false
  end
end

describe "#full_name" do
  it "combines first and last name" do
    user = FactoryBot.create(:user, first_name: "Jane", last_name: "Doe")
    expect(user.full_name).to eq("Jane Doe")
  end

  it "handles nil first name" do
    user = FactoryBot.create(:user, first_name: nil, last_name: "Doe")
    expect(user.full_name).to eq("Doe")
  end

  it "handles nil last name" do
    user = FactoryBot.create(:user, first_name: "Jane", last_name: nil)
    expect(user.full_name).to eq("Jane")
  end

  it "handles both names nil" do
    user = FactoryBot.create(:user, first_name: nil, last_name: nil)
    expect(user.full_name).to eq("")
  end
end
```

## Good Patterns

### 1. Test Business Rules Through Behavior

```ruby
describe TokenUsage::Window do
  describe "#rate_limited??" do
    it "returns true when usage exceeds limit within window" do
      window = FactoryBot.create(:token_usage_window)
      FactoryBot.create(:token_usage_datum, window: window, input_tokens: 500_000)

      expect(window.rate_limited?).to be true
    end

    it "returns false for temporary throttles" do
      window = FactoryBot.create(:token_usage_window)
      FactoryBot.create(:token_usage_datum, window: window, input_tokens: 1000)

      expect(window.rate_limited?).to be false
    end
  end
end
```

### 3. Structure with describe/context and let/let!

```ruby
describe User do
  describe ".active" do  # Class method: use dot
    let!(:active_user) { FactoryBot.create(:user) }
    let!(:inactive_user) { FactoryBot.create(:user, :inactive) }

    it "returns users who logged in within 30 days" do
      expect(User.active).to include(active_user)
      expect(User.active).not_to include(inactive_user)
    end
  end

  describe "#active?" do
    context "when user has no subscription" do
      let(:user) { FactoryBot.create(:user) }

      it "is deactivated" do
        expect(user.subscription).to be_nil
        expect(user).to_not be_activated
      end
    end

    context "When user has subscription" do
      context "active subscription" do
        let(:user) { FactoryBot.create(:user, :with_subscription) }

        it "is activated" do
          expect(user).to be_activated
        end
      end

      context "inactive subscription" do
        let(:user) { FactoryBot.create(:user, :with_subscription, :deactivated) }

        it "is not activated" do
          expect(user).to_not be_activated
        end
      end
    end
  end

  describe "#deactivate!" do
    context "when user has active subscription" do
      let(:user) { FactoryBot.create(:user, :with_subscription) }

      it "cancels the subscription first" do
        expect { user.deactivate! }.to change { user.active? }
        expect(user.subscription).to be_deactivated
      end
    end

    context "when the user's subscription is already canceled" do
      let(:user) { FactoryBot.create(:user, :with_subscription, :deactivated) }

      it "does nothing" do
        expect { user.deactivate! }.to_not change { user.active? }
      end
    end
  end
end
```

### 4. One Behavior Per Test

```ruby
# BAD - Multiple behaviors in one test
it "processes the order" do
  order.process
  expect(order).to be_processed
  expect(order.processed_at).to be_present
  expect(order.user.orders_count).to eq(1)
  expect(OrderMailer.deliveries.count).to eq(1)
end

# GOOD - Separate concerns
describe "#process" do
  it "marks order as processed" do
    order.process
    expect(order).to be_processed
  end

  it "records processing timestamp" do
    freeze_time do
      order.process
      expect(order.processed_at).to eq(Time.current)
    end
  end

  it "increments user order count" do
    expect { order.process }.to change { order.user.orders_count }.by(1)
  end

  it "sends confirmation email" do
    expect { order.process }.to have_enqueued_mail(OrderMailer, :confirmation)
  end
end
```

### 5. Guidelines for `let` and `let!`

**`let` is lazy**: The block is not executed until you reference the variable. This is efficient because unused variables don't hit the database.

**`let!` is eager**: The block executes immediately before each test. Use this when database state must exist before the test runs, such as when testing scopes or queries.

```ruby
describe User do
  let(:user) { FactoryBot.create(:user, email: "test@example.com") }

  describe "#active?" do
    it "returns true for valid user" do
      expect(user.active?).to be true
    end

    context "when email is nil" do
      let(:user) { FactoryBot.create(:user, email: nil) }

      it "returns false" do
        expect(user.active?).to be false
      end
    end

    context "when account is suspended" do
      let(:user) { FactoryBot.create(:user, :suspended) }

      it "returns false" do
        expect(user.active?).to be false
      end
    end
  end
end
```

Use `let!` for scope tests where records must exist before the query runs:

```ruby
describe ".published" do
  let!(:published_posts) { FactoryBot.create_list(:post, 3, :published) }
  let!(:draft_post) { FactoryBot.create(:post, :draft) }

  it "returns only published posts" do
    expect(Post.published).to match_array(published_posts)
  end
end
```

Override `let` in nested `context` blocks to create variations without duplicating setup:

```ruby
describe "#valid?" do
  let(:user) { FactoryBot.create(:user, email: "test@example.com") }

  it "returns true for valid email" do
    expect(user.valid?).to be true
  end

  context "with invalid email" do
    let(:user) { FactoryBot.create(:user, email: "invalid") }

    it "returns false" do
      expect(user.valid?).to be false
    end
  end
end
```

### 6. Test Scopes with Real Data

```ruby
describe ".recent" do
  let!(:old_record) { FactoryBot.create(:article, created_at: 2.months.ago) }
  let!(:new_record) { FactoryBot.create(:article, created_at: 1.day.ago) }
  let!(:newest_record) { FactoryBot.create(:article, created_at: 1.hour.ago) }

  it "returns records from last 30 days ordered by recency" do
    expect(Article.recent).to eq([newest_record, new_record])
  end
end
```

## Validation Checklist

Before considering a test complete, verify:

1. **Delete the implementation** - Does the test fail?
2. **Introduce a bug** - Does the test catch it?
3. **Read the test alone** - Can you understand the business rule?
4. **No mocks** - Are you using real objects and factories?
5. **Review assertions** - Are you testing behavior, not implementation?

## Red Flags in Generated Tests

Watch for these signs of low-value tests:

- Any use of `allow`, `receive`, `double`, or `instance_double` (BANNED)
- Tests that only verify method calls happened (`expect(x).to have_received`)
- Testing private methods directly
- No edge cases or error conditions (nil, empty, boundaries)
- Descriptions like "it works" or "it returns correctly"
- Tests that pass when implementation is deleted

## Example: Complete Model Test

```ruby
RSpec.describe TokenUsageWindow do
  describe ".current" do
    it "returns the window containing the current time" do
      freeze_time do
        window = TokenUsageWindow.current

        expect(window.starts_at).to be <= Time.current
        expect(window.ends_at).to be > Time.current
      end
    end

    it "creates a new window if none exists" do
      expect { TokenUsageWindow.current }.to change(TokenUsageWindow, :count).by(1)
    end

    it "returns existing window if one covers current time" do
      existing = FactoryBot.create(:token_usage_window, :current)

      expect(TokenUsageWindow.current).to eq(existing)
      expect(TokenUsageWindow.count).to eq(1)
    end
  end

  describe "#rate_limited?" do
    let(:window) { FactoryBot.create(:token_usage_window) }

    context "when total tokens exceed threshold" do
      let!(:usage) { FactoryBot.create(:token_usage, token_usage_window: window, input_tokens: 400_000, output_tokens: 100_000) }

      it "returns true" do
        expect(window.rate_limited?).to be true
      end
    end

    context "when usage is within limits" do
      let!(:usage) { FactoryBot.create(:token_usage, token_usage_window: window, input_tokens: 1000, output_tokens: 500) }

      it "returns false" do
        expect(window.rate_limited?).to be false
      end
    end

    context "with no usage data" do
      it "returns false" do
        expect(window.rate_limited?).to be false
      end
    end
  end

  describe "#add_usage" do
    let(:window) { FactoryBot.create(:token_usage_window) }

    it "creates a usage record with provided tokens" do
      expect {
        window.add_usage(input_tokens: 100, output_tokens: 50)
      }.to change(window.token_usages, :count).by(1)

      usage = window.token_usages.last
      expect(usage.input_tokens).to eq(100)
      expect(usage.output_tokens).to eq(50)
    end

    it "aggregates multiple usage entries" do
      window.add_usage(input_tokens: 100, output_tokens: 50)
      window.add_usage(input_tokens: 200, output_tokens: 100)

      expect(window.total_input_tokens).to eq(300)
      expect(window.total_output_tokens).to eq(150)
    end
  end
end
```
