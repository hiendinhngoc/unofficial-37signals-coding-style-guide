# Notifications

> Read this when implementing in-app notifications, email digests, or activity feeds.

---

## Overview

Time-window bundling, user preferences, real-time updates with Turbo Streams.

## Key Patterns

### Timestamps Over Booleans

```ruby
class Notification < ApplicationRecord
  scope :unread, -> { where(read_at: nil) }
  scope :read, -> { where.not(read_at: nil) }

  def read?
    read_at.present?
  end

  def read
    update!(read_at: Time.current)
    broadcast_remove_to user, :notifications
  end
end
```

### Time-Window Bundling

```ruby
class Notification::Bundle < ApplicationRecord
  belongs_to :user

  enum :status, %i[pending processing delivered]

  scope :due, -> { pending.where("ends_at <= ?", Time.current) }

  # Query notifications in window - no foreign key needed!
  def notifications
    user.notifications.where(created_at: starts_at..ends_at).unread
  end
end
```

### User Settings Model

```ruby
class User::Settings < ApplicationRecord
  belongs_to :user

  enum :bundle_email_frequency, %i[never every_few_hours daily weekly],
    default: :every_few_hours

  def bundle_aggregation_period
    case bundle_email_frequency
    when "every_few_hours" then 4.hours
    when "daily" then 1.day
    when "weekly" then 1.week
    end
  end
end
```

### Real-Time Updates

```ruby
class Notification < ApplicationRecord
  after_create_commit :broadcast_unread

  def read
    update!(read_at: Time.current)
    broadcast_remove_to user, :notifications
  end

  private
    def broadcast_unread
      broadcast_prepend_to user, :notifications, target: "notifications"
    end
end
```

## Rules

- [ ] Use `read_at` timestamps instead of boolean `read` flags
- [ ] Bundle with time windows (no foreign key needed)
- [ ] Extract settings to `User::Settings` model
- [ ] Auto-bundle via `after_create` callback
- [ ] Broadcast changes for real-time sync across tabs
- [ ] Use IntersectionObserver for infinite scroll
- [ ] Group notifications client-side with Stimulus
- [ ] Use signed tokens for email unsubscribe links

### Infinite Scroll with IntersectionObserver

```javascript
export default class extends Controller {
  static values = { url: String }

  connect() {
    new IntersectionObserver((entries) => {
      if (entries.find(e => e.isIntersecting)) {
        this.#fetch()
      }
    }).observe(this.element)
  }

  #fetch() {
    get(this.urlValue, { responseKind: "turbo-stream" })
  }
}
```

### RESTful Controllers

```ruby
# Model "reading" as a resource
resources :notifications do
  resource :reading, only: [:create]
end

class Notifications::ReadingsController < ApplicationController
  def create
    @notification = Current.user.notifications.find(params[:notification_id])
    @notification.read
  end
end
```

### Email Unsubscribe with Signed Tokens

```ruby
# User model
generates_token_for :unsubscribe, expires_in: 1.month

# Mailer
@unsubscribe_token = user.generate_token_for(:unsubscribe)

# Controller
def create
  user = User.find_by_token_for(:unsubscribe, params[:token])
  user&.settings&.bundle_email_never!
end
```
