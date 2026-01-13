# ActionCable

> Read this when adding real-time WebSocket features like live updates, notifications, or collaborative editing.

---

## Overview

Use Solid Cable (database-backed) instead of Redis. Always scope broadcasts by account in multi-tenant apps.

## Key Patterns

### Connection Authentication

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      set_current_user || reject_unauthorized_connection
    end

    private
      def set_current_user
        if session = Session.find_signed(cookies.signed[:session_token])
          self.current_user = session.user
        end
      end
  end
end
```

### Force Disconnect Users

```ruby
def deactivate
  update!(active: false)
  ActionCable.server.remote_connections
    .where(current_user: self)
    .disconnect(reconnect: false)
end
```

### Dual Broadcasting

```ruby
module Board::Broadcastable
  extend ActiveSupport::Concern

  included do
    broadcasts_refreshes
    broadcasts_refreshes_to ->(board) { [board.account, :all_boards] }
  end
end
```

## Rules

- [ ] Use Solid Cable (database-backed) instead of Redis
- [ ] Set tenant context during WebSocket connection
- [ ] Use `remote_connections.disconnect` to force disconnect users
- [ ] Dual broadcast: specific resources AND catch-all streams
- [ ] **Always scope broadcasts by account** (prevents self-DoS)
- [ ] Use `broadcast_*_later` for async broadcasting
- [ ] Test with `assert_turbo_stream_broadcasts`

### Critical: Scope Broadcasts by Account

```ruby
# WRONG - broadcasts to ALL accounts!
<%= turbo_stream_from :all_boards %>

# CORRECT - scoped to current account
<%= turbo_stream_from [Current.account, :all_boards] %>
```

### Solid Cable Configuration

```yaml
# config/cable.yml
production:
  adapter: solid_cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

### Testing Broadcasts

```ruby
test "broadcasts on update" do
  assert_turbo_stream_broadcasts([card.user, :notifications]) do
    card.update!(title: "New Title")
  end
end
```

### Selective Subscriptions

```erb
<%# Subscribe only to visible data %>
<% if filter.boards.any? %>
  <% filter.boards.each do |board| %>
    <%= turbo_stream_from board %>
  <% end %>
<% else %>
  <%= turbo_stream_from [Current.account, :all_boards] %>
<% end %>
```
