# Authentication

> Read this when implementing passwordless authentication or magic link sign-in.

---

## Overview

Use magic links over password authentication. Custom code (~150 lines) beats Devise for passwordless flows.

## Key Patterns

### Separate Identity from User

```ruby
class Identity < ApplicationRecord
  has_many :sessions, dependent: :destroy
  has_many :magic_links, dependent: :destroy
  has_many :users, dependent: :nullify  # One person, many accounts

  normalizes :email_address, with: ->(v) { v.strip.downcase.presence }
end

class User < ApplicationRecord
  belongs_to :identity
  belongs_to :account
end
```

### Magic Link Model

```ruby
class MagicLink < ApplicationRecord
  CODE_LENGTH = 6
  EXPIRATION_TIME = 15.minutes

  belongs_to :identity

  scope :active, -> { where(expires_at: Time.current...) }
  scope :stale, -> { where(expires_at: ..Time.current) }

  before_validation :generate_code, :set_expiration, on: :create

  def self.consume(code)
    active.find_by(code: code.strip.upcase)&.tap(&:destroy)
  end

  private
    def generate_code
      self.code ||= SecureRandom.alphanumeric(CODE_LENGTH).upcase
    end

    def set_expiration
      self.expires_at ||= EXPIRATION_TIME.from_now
    end
end
```

### Authentication Concern DSL

```ruby
module Authentication
  extend ActiveSupport::Concern

  class_methods do
    def require_unauthenticated_access(**options)
      allow_unauthenticated_access(**options)
      before_action :redirect_authenticated_user, **options
    end

    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def require_authentication
      resume_session || request_authentication
    end

    def start_new_session_for(identity)
      identity.sessions.create!(
        user_agent: request.user_agent,
        ip_address: request.remote_ip
      ).tap { |session| set_current_session(session) }
    end

    def set_current_session(session)
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.signed_id,
        httponly: true,
        same_site: :lax
      }
    end
end
```

## Rules

- [ ] Use magic links over passwords (~150 lines of custom code)
- [ ] Separate `Identity` from `User` for multi-account support
- [ ] Use `MagicLink` model with expiration and cleanup
- [ ] Rate limit aggressively: 10 requests per 3-15 minutes
- [ ] Verify email matches before consuming magic link
- [ ] Put magic link code in email subject (cross-device auth)
- [ ] Use class-level DSL: `require_unauthenticated_access`, `allow_unauthenticated_access`
- [ ] Store session in signed cookie with `httponly: true`

## Security

```ruby
# Rate limit in controller
rate_limit to: 10, within: 15.minutes, only: :create,
  with: -> { redirect_to root_path, alert: "Try again later." }

# Verify email matches the magic link's identity
def authenticate_with(magic_link)
  if session[:pending_email] == magic_link.identity.email_address
    start_new_session_for magic_link.identity
  else
    redirect_to new_session_path, alert: "Authentication failed."
  end
end
```
