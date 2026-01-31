---
paths:
  - "app/models/**/*.rb"
  - "app/services/**/*.rb"
  - "app/forms/**/*.rb"
---

# Rails Model Structure

## Section Order

Organize model code in this order:

```ruby
class User < ApplicationRecord
  # 1. Includes/Extends
  include Searchable
  extend FriendlyId

  # 2. Constants
  ROLES = %w[admin member guest].freeze
  MAX_LOGIN_ATTEMPTS = 5

  # 3. Attribute declarations
  attribute :remember_me, :boolean, default: false
  store_accessor :settings, :theme, :notifications

  # 4. Validations
  validates :email, presence: true, uniqueness: true
  validates :name, length: { maximum: 100 }

  # 5. Associations
  belongs_to :organization
  has_many :posts, dependent: :destroy
  has_many :comments
  accepts_nested_attributes_for :profile

  # 6. Delegations
  delegate :name, to: :organization, prefix: true
  delegate :city, :country, to: :address

  # 7. Callbacks (before/after filters)
  before_validation :normalize_email
  after_create :send_welcome_email

  # 8. Scopes
  scope :active, -> { where(active: true) }
  scope :recent, -> { order(created_at: :desc) }

  # 9. Class methods
  def self.find_by_credentials(email, password)
    find_by(email: email)&.authenticate(password)
  end

  # 10. Public instance methods
  def full_name
    "#{first_name} #{last_name}".strip
  end

  def admin?
    role == "admin"
  end

  # 11. Private instance methods
  private

  def normalize_email
    self.email = email&.downcase&.strip
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end
end
```

## Naming Conventions

- **Predicate methods** end with `?` - return boolean, no side effects
  ```ruby
  def active?
  def can_publish?
  ```

- **Bang methods** end with `!` - dangerous or raises on failure
  ```ruby
  def activate!    # raises if fails
  def publish!     # mutates state, may raise
  ```

- **Regular methods** - for everything else
  ```ruby
  def activate     # returns true/false
  def publish      # safe version
  ```

## Service Objects

Use service objects when a method orchestrates multiple models, calls external APIs, or has complex business logic with side effects.

**Problem: Model doing too much**

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def register(params)
    self.attributes = params
    return false unless save

    organization = Organization.create!(name: "#{name}'s Org", owner: self)
    Subscription.create!(user: self, plan: Plan.free, organization: organization)
    UserMailer.welcome(self).deliver_later
    AdminMailer.new_signup(self).deliver_later
    Analytics.track("user_registered", user_id: id)
    true
  end
end

# Controller
def create
  @user = User.new
  if @user.register(user_params)
    redirect_to dashboard_path
  else
    render :new
  end
end
```

**Solution: Extract to service object**

```ruby
# app/services/user_registration.rb
class UserRegistration
  def initialize(params)
    @params = params
  end

  def call
    ActiveRecord::Base.transaction do
      create_user
      create_organization
      create_subscription
    end
    send_notifications
    track_analytics
    @user
  rescue ActiveRecord::RecordInvalid
    nil
  end

  def user
    @user
  end

  private

  def create_user
    @user = User.create!(@params)
  end

  def create_organization
    @organization = Organization.create!(name: "#{@user.name}'s Org", owner: @user)
  end

  def create_subscription
    Subscription.create!(user: @user, plan: Plan.free, organization: @organization)
  end

  def send_notifications
    UserMailer.welcome(@user).deliver_later
    AdminMailer.new_signup(@user).deliver_later
  end

  def track_analytics
    Analytics.track("user_registered", user_id: @user.id)
  end
end

# app/models/user.rb - now simple
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
  validates :name, presence: true
end

# Controller
def create
  registration = UserRegistration.new(user_params)
  if @user = registration.call
    redirect_to dashboard_path
  else
    @user = registration.user
    render :new
  end
end
```

## Form Objects

Use form objects when a form spans multiple models or has validations that don't belong on any single model.

**Problem: Controller handling multi-model form**

```ruby
# app/controllers/checkouts_controller.rb
def create
  @user = User.new(checkout_params[:user])
  @address = Address.new(checkout_params[:address])
  @order = Order.new(checkout_params[:order])

  if checkout_params[:same_as_billing]
    @address.attributes = checkout_params[:billing_address]
  end

  ActiveRecord::Base.transaction do
    @user.save!
    @address.user = @user
    @address.save!
    @order.user = @user
    @order.shipping_address = @address
    @order.save!
  end
  redirect_to confirmation_path(@order)
rescue ActiveRecord::RecordInvalid
  render :new
end
```

**Solution: Extract to form object**

```ruby
# app/forms/checkout_form.rb
class CheckoutForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :email, :string
  attribute :name, :string
  attribute :street, :string
  attribute :city, :string
  attribute :zip, :string
  attribute :same_as_billing, :boolean, default: false
  attribute :billing_street, :string
  attribute :billing_city, :string
  attribute :billing_zip, :string
  attribute :product_id, :integer
  attribute :quantity, :integer

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true
  validates :street, :city, :zip, presence: true
  validates :product_id, :quantity, presence: true
  validate :product_in_stock

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      create_user
      create_address
      create_order
    end
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    false
  end

  def order
    @order
  end

  private

  def create_user
    @user = User.create!(email: email, name: name)
  end

  def create_address
    addr = same_as_billing ? billing_address_attrs : shipping_address_attrs
    @address = Address.create!(addr.merge(user: @user))
  end

  def create_order
    @order = Order.create!(
      user: @user,
      shipping_address: @address,
      product_id: product_id,
      quantity: quantity
    )
  end

  def shipping_address_attrs
    { street: street, city: city, zip: zip }
  end

  def billing_address_attrs
    { street: billing_street, city: billing_city, zip: billing_zip }
  end

  def product_in_stock
    product = Product.find_by(id: product_id)
    return errors.add(:product_id, "not found") unless product
    errors.add(:quantity, "exceeds stock") if product.stock < quantity.to_i
  end
end

# app/controllers/checkouts_controller.rb - now simple
def create
  @checkout = CheckoutForm.new(checkout_params)
  if @checkout.save
    redirect_to confirmation_path(@checkout.order)
  else
    render :new
  end
end
```
