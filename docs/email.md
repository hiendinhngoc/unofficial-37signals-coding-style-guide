# Email

> Read this when implementing transactional emails, notification digests, or email delivery infrastructure.

---

## Overview

Multi-tenant URL handling, timezone-aware delivery, and resilient SMTP error handling.

## Key Patterns

### Multi-Tenant URL Helpers

```ruby
class ApplicationMailer < ActionMailer::Base
  private
    def default_url_options
      if Current.account
        super.merge(script_name: Current.account.slug)
      else
        super
      end
    end
end
```

### Timezone-Aware Delivery

```ruby
def deliver
  user.in_time_zone do
    Current.with_account(user.account) do
      Notification::BundleMailer.notification(self).deliver
    end
  end
end

# In User model
def in_time_zone(&block)
  Time.use_zone(settings.timezone_name, &block)
end
```

### SMTP Error Handling

```ruby
module SmtpDeliveryErrorHandling
  extend ActiveSupport::Concern

  included do
    # Transient - retry with backoff
    retry_on Net::OpenTimeout, Net::ReadTimeout,
      wait: :polynomially_longer

    # Permanent - log and continue
    rescue_from Net::SMTPFatalError do |error|
      Rails.logger.info "SMTP error: #{error.message}"
    end
  end
end

# Apply globally
ActionMailer::MailDeliveryJob.include SmtpDeliveryErrorHandling
```

### One-Click Unsubscribe

```ruby
module Mailers::Unsubscribable
  extend ActiveSupport::Concern

  included do
    after_action :set_unsubscribe_headers
  end

  def set_unsubscribe_headers
    headers["List-Unsubscribe-Post"] = "List-Unsubscribe=One-Click"
    headers["List-Unsubscribe"] = "<#{unsubscribe_url(token: @token)}>"
  end
end
```

## Rules

- [ ] Override `default_url_options` for multi-tenant URLs
- [ ] Wrap delivery in recipient's timezone: `Time.use_zone`
- [ ] Replace SVG with HTML/CSS (most clients block SVG)
- [ ] Add SMTP error handling with `retry_on` and `rescue_from`
- [ ] Use `ActiveJob.perform_all_later` for batch delivery
- [ ] Add RFC 8058 `List-Unsubscribe` headers
- [ ] Use inline styles and table-based layouts
- [ ] Set tenant context in mailer previews
- [ ] Put magic link code in email subject (cross-device auth)

### Email-Safe Avatars

```ruby
def mail_avatar_tag(user, size: 48)
  if user.avatar.attached?
    image_tag user_avatar_url(user), size: size
  else
    tag.span user.initials,
      style: "background: #{avatar_color(user)}; border-radius: 50%;"
  end
end
```

### Batch Delivery

```ruby
def self.deliver_all
  due.in_batches do |batch|
    jobs = batch.collect { DeliverJob.new(_1) }
    ActiveJob.perform_all_later(jobs)
  end
end
```

### Email Layout Best Practices

```html
<style>
  /* Outlook-specific fixes */
  .avatar { mso-line-height-rule: exactly; }
  table, td { border-collapse: collapse; }
</style>

<table>
  <tr>
    <td><%= mail_avatar_tag(user) %></td>
    <td><%= yield %></td>
  </tr>
</table>
```
