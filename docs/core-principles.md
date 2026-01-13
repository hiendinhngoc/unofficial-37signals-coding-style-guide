# Core Principles

> Essential rules for any Rails project. These are non-negotiable fundamentals from 37signals' development philosophy.

---

## Philosophy

### The 37signals Way

- [ ] Rich domain models over service objects
- [ ] CRUD controllers over custom actions
- [ ] Concerns for horizontal code sharing
- [ ] Vanilla Rails is plenty
- [ ] Ship to learn - prototype quality is valid
- [ ] Fix root causes, not symptoms
- [ ] Explicit over clever
- [ ] Build it yourself before reaching for gems

### Ship, Validate, Refine

- [ ] Merge "prototype quality" code to validate design before cleanup
- [ ] Don't polish prematurely - real-world usage reveals what matters
- [ ] Features evolve through iterations (accept 2-3 attempts before settling)
- [ ] Real data reveals what local testing cannot

### When to Extract

- [ ] Start in controller, extract when it gets messy
- [ ] Rule of three: duplicate twice before abstracting
- [ ] Don't extract prematurely - wait for real pain
- [ ] Ask "Is this abstraction earning its keep?"

---

## Routing

### CRUD Everything

Every action maps to a CRUD verb. When something doesn't fit, create a new resource.

```ruby
# BAD: Custom actions
resources :cards do
  post :close
  post :archive
end

# GOOD: New resources for state changes
resources :cards do
  resource :closure      # POST to close, DELETE to reopen
  resource :pin          # POST to pin, DELETE to unpin
end
```

### Turn Verbs into Nouns

| Action | Resource |
|--------|----------|
| Close a card | `resource :closure` |
| Watch a board | `resource :watching` |
| Pin an item | `resource :pin` |
| Publish a board | `resource :publication` |
| Assign a user | `resources :assignments` |

### Routing Rules

- [ ] Every action maps to a CRUD verb
- [ ] Use `resource` (singular) for one-per-parent resources
- [ ] Use `shallow: true` to avoid deep nesting (3+ levels)
- [ ] No separate API namespace - just `respond_to`

---

## Controllers

### Core Principle

Controllers are thin orchestrators. Business logic lives in models.

```ruby
# GOOD: Controller just orchestrates
class Cards::ClosuresController < ApplicationController
  def create
    @card.close  # All logic in model
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end

# BAD: Business logic in controller
def create
  @card.transaction do
    @card.create_closure!(user: Current.user)
    @card.events.create!(action: :closed)
    NotificationMailer.card_closed(@card).deliver_later
  end
end
```

### Controller Rules

- [ ] Controllers are thin orchestrators
- [ ] Business logic lives in models
- [ ] Use bang methods (`create!`) - let it crash
- [ ] Use `respond_to` for multiple formats
- [ ] Return `head :forbidden` for authorization failures

### Use Concerns for Shared Behavior

```ruby
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find(params[:card_id])
    end
end
```

---

## Models

### Heavy Use of Concerns

```ruby
class Card < ApplicationRecord
  include Assignable, Closeable, Eventable, Pinnable, Taggable

  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

### State as Records, Not Booleans

```ruby
# BAD: Boolean column
class Card < ApplicationRecord
  scope :closed, -> { where(closed: true) }
end

# GOOD: Separate record
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user
  # created_at gives you WHEN, user gives you WHO
end

class Card < ApplicationRecord
  has_one :closure, dependent: :destroy
  scope :closed, -> { joins(:closure) }
  scope :open, -> { where.missing(:closure) }
end
```

### Model Rules

- [ ] Each concern should be 50-150 lines
- [ ] Concerns must be cohesive - related functionality together
- [ ] Name concerns for capability: `Closeable`, `Watchable`, `Assignable`
- [ ] State records over booleans (capture WHO and WHEN)
- [ ] Use `default: -> { }` lambdas for belongs_to defaults
- [ ] Default creator to `Current.user`
- [ ] Use `Current` for request context
- [ ] Minimal validations - prefer DB constraints
- [ ] Use bang methods (`create!`) - let it crash
- [ ] Use `touch: true` for cache invalidation chains

### Scope Naming

```ruby
# GOOD - business-focused
scope :active, -> { where.missing(:pop) }
scope :unassigned, -> { where.missing(:assignments) }

# BAD - SQL-ish
scope :without_pop, -> { ... }
scope :no_assignments, -> { ... }
```

---

## Views

### View Rules

- [ ] Use Turbo Streams for partial page updates
- [ ] Standard partials over ViewComponents
- [ ] Prefer locals over instance variables in partials
- [ ] Use Rails `dom_id` helper for IDs

### Fragment Caching

```erb
<% cache card do %>
  <%= render "cards/preview", card: card %>
<% end %>

<%# Collection caching %>
<%= render partial: "cards/preview", collection: @cards, cached: true %>
```

---

## Hotwire

### Turbo Rules

- [ ] Use `method: :morph` for complex updates
- [ ] Use `data-turbo-permanent` for elements that shouldn't refresh
- [ ] Ensure unique IDs - duplicates break morphing
- [ ] Use Turbo Streams for partial page updates

### Stimulus Rules

- [ ] Single-purpose controllers - one job per controller
- [ ] Use Values API: `static values = { url: String }`
- [ ] Always clean up in `disconnect()` - timers, listeners, observers
- [ ] Dispatch events for controller communication
- [ ] Use targets over CSS selectors

---

## CSS

### Philosophy

Native CSS only - no Sass, PostCSS, or Tailwind.

### CSS Rules

- [ ] Use native CSS only
- [ ] Use `@layer` for specificity: reset, base, layout, components, utilities
- [ ] Use CSS custom properties for theming
- [ ] Use native CSS nesting
- [ ] ~60 utility classes total (not Tailwind-style hundreds)

---

## Database

### Database Rules

- [ ] Use UUIDs instead of auto-incrementing integers
- [ ] State records over booleans - capture WHO and WHEN
- [ ] Use Solid Queue/Cache/Cable (no Redis)
- [ ] Hard deletes over soft deletes
- [ ] Always index foreign keys
- [ ] Index columns you filter/sort by

---

## Background Jobs

### Job Rules

- [ ] Use `enqueue_after_transaction_commit = true`
- [ ] Retry transient errors with polynomial backoff
- [ ] Jobs just call model methods (shallow jobs)
- [ ] Stagger recurring job schedules

```ruby
# Jobs just call model methods
class NotifyRecipientsJob < ApplicationJob
  def perform(notifiable)
    notifiable.notify_recipients
  end
end
```

---

## Testing

### Testing Rules

- [ ] Use Minitest, not RSpec
- [ ] Use fixtures, not FactoryBot
- [ ] Use labels for relationships, not IDs
- [ ] Use `assert_difference` for counting changes
- [ ] Tests ship with features in the same commit

```ruby
test "closing a card creates an event" do
  card = cards(:urgent_bug)

  assert_difference "Event.count", 1 do
    card.close(by: users(:david))
  end
end
```

---

## Security

### Security Rules

- [ ] Always escape before `html_safe`
- [ ] Don't HTTP cache pages with forms (CSRF tokens get stale)
- [ ] Rate limit auth endpoints
- [ ] Always scope lookups through user/tenant

```ruby
# BAD
@comment = Comment.find(params[:id])

# GOOD - scope through user
@comment = Current.user.comments.find(params[:id])
```

---

## Code Quality

- [ ] Use positive names (`active` not `not_deleted`)
- [ ] Prefer database constraints over AR validations
- [ ] Use `params.expect` over `params.require.permit` (Rails 8+)
- [ ] Write-time operations over read-time computations

---

## What to Avoid

| Instead of | Use |
|------------|-----|
| Devise | Custom auth (~150 lines) |
| Pundit/CanCanCan | Model predicates |
| Service objects | Rich model methods |
| ViewComponent | ERB partials |
| GraphQL | REST + Turbo |
| Sidekiq | Solid Queue |
| React/Vue | Turbo + Stimulus |
| Tailwind | Native CSS |
| RSpec | Minitest |
| FactoryBot | Fixtures |

### The Philosophy

Before adding a dependency, ask:
1. Can vanilla Rails do this?
2. Is the complexity worth the benefit?
3. Will we need to maintain this dependency?
4. Does it make the codebase harder to understand?
