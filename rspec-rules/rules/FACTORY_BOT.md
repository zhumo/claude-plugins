---
paths:
  - "spec/**/*.rb"
  - "spec/factories/**/*.rb"
---

# FactoryBot Best Practices

## Explicitly state FactoryBot

Always use explicit `FactoryBot.` prefix:

```ruby
# GOOD
FactoryBot.create(:user)
FactoryBot.build(:order)

# BAD - implicit syntax
create(:user)
build(:order)
```

## Pre-define attributes in factory, override selectively
Define various default traits then lean on them consistently. Only when you need to, override them in the test themselves.

```ruby
# spec/factories/user.rb
FactoryBot.define do
  factory :user do
    first_name: "Charlie"
  end
end

# spec/model/user_spec.rb
user1 = FactoryBot.create(:user)
user1.first_name
# => "Charlie"

user2 = FactoryBot.create(:user, first_name: "Joe")
user2.first_name
# => "Joe"
```

## Traits
If used repeatedly, use traits to override default attributes

```ruby
FactoryBot.define do
  factory :user do
    first_name { "John" }
    last_name { "Doe" }
    admin { false }

    trait :admin do
      admin { true }
    end
  end
end

user = FactoryBot.create(:user)
# user.admin => false
admin = FactoryBot.create(:user, :admin)
# user.admin => true
```

## Use FactoryBot's native association creation
```ruby
# If the factory name is the same as the association name, it's simple
factory :post do
  author
end

# You can also specify a different factory or override attributes
factory :post do
  # ...
  association :author
end

# Builds and saves a User and a Post
post = FactoryBot.create(:post)
post.new_record?        # => false
post.author.new_record? # => false

# This allows you to skip doing this:
let(:author) { FactoryBot.create(:author) }
let(:post) { FactoryBot.create(:post, author: author) }

# But you can do it if there is something particular about the autho we need to overwrite
let(:uathor) { FactoryBot.create(:author, :last_name: "Musk") }
let(:post) { FactoryBot.create(:post, author: author) }
```

FactoryBot does NOT automatically create has_many :through associations. You must explicitly create the join records:

```ruby
# Models
class User < ApplicationRecord
  has_many :memberships
  has_many :groups, through: :memberships
end

# Factory - use a trait with after(:create) callback
factory :user do
  trait :with_groups do
    after(:create) do |user|
      FactoryBot.create_list(:membership, 2, user: user)
    end
  end
end

# Usage
user = FactoryBot.create(:user, :with_groups)
user.groups.count # => 2
```

## Increment attributes with sequences
When we need to generate different attributes, use sequences to minimize the amount of randomness.
```ruby
factory :user do
  sequence(:name) { |n| "Jane Doe #{n}" }
end
```

## Using factories
Unless specifically asked otherwise, only ever use `FactoryBot.create` in tests.

```ruby
# DO THIS BY DEFAULT ALWAYS
# Returns a saved User instance
user = FactoryBot.create(:user)

# DO NOT DO THE BELOW, UNLESS ASKED
# Returns an User instance that's not saved
user = FactoryBot.build(:user)
# Returns a hash of attributes that can be used to build an User instance
attrs = FactoryBot.attributes_for(:user)
# Returns an object with all defined attributes stubbed out
stub = FactoryBot.build_stubbed(:user)

# Passing a block to any of the methods above will yield the return object
FactoryBot.create(:user) do |user|
  user.posts.create(attributes_for(:post))
end
```

## Aliases

Aliases allow associations to be more readable.

```ruby
factory :user, aliases: [:author, :commenter] do
  first_name { "John" }
  last_name { "Doe" }
  date_of_birth { 18.years.ago }
end

factory :post do
  # instead of association :author, factory: :user
  author
  title { "How to read a book effectively" }
  body { "There are five steps involved." }
end

factory :comment do
  # instead of association :commenter, factory: :user
  commenter
  body { "Great article!" }
end
```
