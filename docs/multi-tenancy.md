# Multi-Tenancy

> Read this when building a SaaS application with multiple accounts/organizations sharing the same codebase.

---

## Overview

Use path-based tenancy with middleware extraction and `Current.account` context.

## Key Patterns

### Path-Based Tenancy

```ruby
# URL: /1234567/boards/123
module AccountSlug
  class Extractor
    def call(env)
      request = ActionDispatch::Request.new(env)

      if request.path_info =~ /\A(\/\d{7,})/
        request.script_name = $1
        request.path_info = $'
        account = Account.find_by(external_id: decode($1))
        Current.with_account(account) { @app.call(env) }
      else
        @app.call(env)
      end
    end
  end
end
```

### Current Context

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :account, :user, :session

  def user=(user)
    super
    self.account = user&.account
  end
end
```

### Job Tenant Preservation

```ruby
module TenantedJob
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
  end

  def serialize
    super.merge("account" => Current.account&.to_gid)
  end

  def deserialize(job_data)
    super
    @account = GlobalID::Locator.locate(job_data["account"])
  end

  def perform_now
    if account.present?
      Current.with_account(account) { super }
    else
      super
    end
  end
end
```

## Rules

- [ ] Use path-based tenancy (`/1234567/boards`)
- [ ] Middleware extracts tenant and sets `Current.account`
- [ ] Use `Current.with_account(account) { }` blocks
- [ ] Serialize tenant in jobs via GlobalID
- [ ] Recurring jobs iterate all tenants
- [ ] Scope session cookie to account path
- [ ] Always scope controller lookups through tenant
- [ ] Defense in depth - don't rely solely on middleware

### Always Scope Lookups

```ruby
# BAD
@comment = Comment.find(params[:id])

# GOOD - scope through tenant
@comment = Current.account.comments.find(params[:id])

# BETTER - scope through user's accessible records
@card = Current.user.accessible_cards.find(params[:id])
```

### Scoped Uniqueness

```ruby
validates :number, uniqueness: { scope: :account_id }
```

### Session Cookie Scoping

```ruby
cookies.signed.permanent[:session_token] = {
  value: session.signed_id,
  path: "/#{account.external_id}"  # Scope to tenant path
}
```
