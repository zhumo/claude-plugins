---
paths:
  - "spec/system/**/*.rb"
  - "spec/features/**/*.rb"
---

# Capybara Integration Testing Guide

## When to Write System Tests

Think of integration tests as automated QA. Any visual user experience should be tested with system tests:

- All user-facing features and interactions
- Negative cases ("user types too many characters" → error displayed)
- Strange states ("user has no email but has twitter handle" → correct fallback)
- Edge cases in the UI (empty states, loading states, error states)

## Setup

```ruby
# spec/system/example_spec.rb
require "rails_helper"

RSpec.describe "Feature Name", type: :system do
  before do
    driven_by :selenium, using: :headless_chrome
  end
end
```

## Model Integration Test Example

```ruby
describe "creating a post" do
  let(:user) { FactoryBot.create(:user) }

  before do
    sign_in user
  end

  it "creates a new post" do
    visit new_post_path

    within ".post-form" do
      fill_in "Title", with: "My Post"
      click_on "Create"
    end

    expect(page).to have_content("Post created")
  end
end
```

## Best Practices
### Selectors

Use user-visible text combined with `within` blocks for specificity:

```ruby
within ".user-profile" do
  click_on "Edit"
  fill_in "Email", with: "user@example.com"
end

within ".shopping-cart" do
  expect(page).to have_content("3 items")
  click_on "Checkout"
end

within ".search-results" do
  expect(page).to have_content("Found 10 matches")
end
```

### Use Methods That Wait
When dealing with javascript interactions, these methods wait for the website to change.

```ruby
expect(page).to have_content("Loaded")
expect(page).to have_button("Submit")
expect(page).to have_link("More info")
find(".element")
click_on "Button"
```

BANNED: Do not use these methods which don't wait

```ruby
all(".items")           # BANNED - returns immediately, may be empty
page.first(".item")     # BANNED - doesn't wait
sleep 2                 # BANNED - arbitrary waits
have_selector(".foo")   # BANNED - not testing from user perspective
```

### Always Use a Wait Method After Actions

```ruby
# GOOD - wait for confirmation
click_on "Submit"
expect(page).to have_content("Saved!")
visit other_path

# BAD - race condition
# If you do not use the expect method, then Capybara will move on, regardless of whether the form submission worked or not.
click_on "Submit"
visit other_path
```

### Test implementation, not behavior
```ruby
# GOOD - testing what user sees
expect(page).to have_button("Submit", disabled: false)

# BAD - testing CSS classes
expect(page).to have_css(".btn-primary.active")
```

### Handling JavaScript

```ruby
# Alerts
accept_alert { click_on "Delete" }
```
