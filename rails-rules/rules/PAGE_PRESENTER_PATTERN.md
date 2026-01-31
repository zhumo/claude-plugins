---
paths:
  - "app/pages/**/*.rb"
  - "app/presenters/**/*.rb"
  - "app/controllers/**/*.rb"
  - "app/views/**/*.erb"
---

# Page and Presenter Pattern for Rails

This document defines how to structure Rails controllers, views, and their supporting objects using the Page and Presenter pattern.

## Core Principles

1. **Controllers are thin** - They only handle HTTP concerns: params, authentication, redirects, and response format
2. **Pages marshal data** - A Page object gathers and organizes all data needed for a specific view
3. **Presenters format output** - Presenter objects wrap models with view-specific formatting and logic
4. **Views are declarative** - Views only call methods on Page and Presenter objects, never query models directly

## Directory Structure

```
app/
├── controllers/
│   └── dashboard_controller.rb
├── pages/
│   └── dashboard/
│       └── show_page.rb
├── presenters/
│   └── user_presenter.rb
└── views/
    └── dashboard/
        └── show.html.erb
```

## Page Objects

### Purpose

A Page object is responsible for:
- Fetching all data needed for a single view
- Instantiating presenters for models
- Providing a clean interface for the view to consume

### Naming Convention

- File: `app/pages/{controller_name}/{action}_page.rb`
- Class: `{ControllerName}::{Action}Page`

### Structure

```ruby
# app/pages/dashboard/show_page.rb
module Dashboard
  class ShowPage
    def initialize(current_user:)
      @current_user = current_user
    end

    def user
      @user ||= UserPresenter.new(@current_user)
    end

    def recent_orders
      @recent_orders ||= @current_user.orders
        .recent
        .limit(10)
        .map { |order| OrderPresenter.new(order) }
    end

    def stats
      @stats ||= {
        total_orders: @current_user.orders.count,
        total_spent: @current_user.orders.sum(:total)
      }
    end
  end
end
```

### Rules for Pages

1. Accept dependencies through the constructor (current_user, params, etc.)
2. Memoize expensive queries with `@var ||=`
3. Return presenters for models, not raw ActiveRecord objects
4. Keep methods focused - one data concern per method
5. No formatting logic - that belongs in presenters

## Presenter Objects

### Purpose

A Presenter wraps a single model instance to:
- Format values for display (dates, currency, truncation)
- Provide computed display values
- Encapsulate conditional display logic
- Delegate to the underlying model for raw data

### Naming Convention

- File: `app/presenters/{model_name}_presenter.rb`
- Class: `{ModelName}Presenter`

### Structure

```ruby
# app/presenters/order_presenter.rb
class OrderPresenter
  delegate :id, :status, :created_at, to: :@order

  def initialize(order)
    @order = order
  end

  def total_display
    helpers.number_to_currency(@order.total)
  end

  def created_at_display
    @order.created_at.strftime("%B %d, %Y")
  end

  def status_badge_class
    case @order.status
    when "completed" then "bg-green-100 text-green-800"
    when "pending" then "bg-yellow-100 text-yellow-800"
    when "cancelled" then "bg-red-100 text-red-800"
    else "bg-gray-100 text-gray-800"
    end
  end

  def status_display
    @order.status.titleize
  end

  private

  def helpers
    ApplicationController.helpers
  end
end
```

### Rules for Presenters

1. Accept the model in the constructor
2. Use `delegate` for passthrough attributes
3. Suffix formatted methods with `_display` for clarity
4. Access Rails helpers through `ApplicationController.helpers`
5. No database queries - work only with the wrapped model and its loaded associations
6. One presenter per model - if you need context-specific formatting, pass options to the constructor

## Controller Implementation

Controllers become minimal:

```ruby
# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  def show
    @page = Dashboard::ShowPage.new(current_user: current_user)
  end
end
```

### Rules for Controllers

1. Instantiate exactly one Page object per action
2. Assign it to `@page`
3. Pass in current_user, params, or other dependencies as keyword arguments
4. Handle authentication/authorization before creating the Page
5. Redirects and flash messages belong here, not in Pages

## View Implementation

Views consume the Page object:

```erb
<%# app/views/dashboard/show.html.erb %>
<h1>Welcome, <%= @page.user.name %></h1>

<div class="stats">
  <p>Total Orders: <%= @page.stats[:total_orders] %></p>
  <p>Total Spent: <%= number_to_currency(@page.stats[:total_spent]) %></p>
</div>

<h2>Recent Orders</h2>
<% @page.recent_orders.each do |order| %>
  <div class="order">
    <span class="<%= order.status_badge_class %>">
      <%= order.status_display %>
    </span>
    <span><%= order.total_display %></span>
    <span><%= order.created_at_display %></span>
  </div>
<% end %>
```

### Rules for Views

1. Only access data through `@page`
2. No model queries (e.g., `User.find`, `@user.orders.where(...)`)
3. No complex conditionals - push that logic to presenters
4. Partials can accept presenters as locals

## Handling Collections

For index pages with collections:

```ruby
# app/pages/orders/index_page.rb
module Orders
  class IndexPage
    def initialize(user:, params:)
      @user = user
      @params = params
    end

    def orders
      @orders ||= base_scope
        .page(@params[:page])
        .map { |order| OrderPresenter.new(order) }
    end

    def filter_options
      Order.statuses.keys.map(&:titleize)
    end

    private

    def base_scope
      scope = @user.orders.recent
      scope = scope.where(status: @params[:status]) if @params[:status].present?
      scope
    end
  end
end
```

## Testing
There is no need to test Presenters and Pages directly. Instead, use integration testing to check whether the data is properly displayed.

## When to Create a Presenter

Create a presenter when:
- A model appears in views and needs formatting
- You have view-specific computed values
- Display logic becomes complex or repeated

Skip presenters when:
- You're just displaying raw values with no formatting
- The model only appears in one simple view

## Common Patterns

### Nested Presenters

```ruby
class OrderPresenter
  def customer
    @customer_presenter ||= CustomerPresenter.new(@order.customer)
  end
end
```

### Presenter with Options

```ruby
class DatePresenter
  def initialize(date, format: :long)
    @date = date
    @format = format
  end

  def to_s
    case @format
    when :short then @date.strftime("%m/%d")
    when :long then @date.strftime("%B %d, %Y")
    end
  end
end
```

### Base Presenter Class

```ruby
# app/presenters/base_presenter.rb
class BasePresenter
  def self.wrap(collection)
    collection.map { |item| new(item) }
  end

  private

  def helpers
    ApplicationController.helpers
  end
end

# Usage in pages:
def orders
  @orders ||= OrderPresenter.wrap(base_scope)
end
```
