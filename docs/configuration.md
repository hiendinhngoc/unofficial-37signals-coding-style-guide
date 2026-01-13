# Configuration

> Read this when setting up Rails environments, credentials, or deployment with Kamal.

---

## Overview

YAML DRYness, environment inheritance, and Kamal deployment patterns.

## Key Patterns

### YAML Anchor References

```yaml
# Use anchors for DRY config
production: &production
  adapter: solid_cable
  polling_interval: 0.1.seconds

beta: *production
staging: *production
```

### Environment Inheritance

```ruby
# config/environments/beta.rb
require_relative "production"

Rails.application.configure do
  # Only override what's different
  config.action_mailer.default_url_options = {
    host: "beta.example.com"
  }
end
```

### ENV Over Config

```ruby
# ENV takes precedence
report_uri = ENV.fetch("CSP_REPORT_URI") {
  config.x.content_security_policy.report_uri
}

# Boolean ENV vars - check key presence first
report_only = if ENV.key?("CSP_REPORT_ONLY")
  ENV["CSP_REPORT_ONLY"] == "true"
else
  config.x.content_security_policy.report_only
end
```

### Test Environment Fallbacks

```ruby
# Don't require credentials in tests
http_basic_authenticate_with(
  name: Rails.env.test? ? "test" :
    Rails.application.credentials.http_auth.name,
  password: Rails.env.test? ? "test" :
    Rails.application.credentials.http_auth.password
)
```

## Rules

- [ ] Use YAML anchors: `beta: *production`
- [ ] Environments inherit from production: `require_relative "production"`
- [ ] ENV variables take precedence over config files
- [ ] Avoid requiring credentials in tests (use fallbacks)
- [ ] Use Kamal secrets with 1Password
- [ ] Support both bare and full-featured local development
- [ ] Make RAILS_ENV explicit in deploy files

### Kamal Secrets

```bash
# .kamal/secrets.production
SECRETS=$(kamal secrets fetch --adapter 1password \
  --from Production/RAILS_MASTER_KEY)

RAILS_MASTER_KEY=$(kamal secrets extract RAILS_MASTER_KEY $SECRETS)
```

### Deploy Configuration

```yaml
# config/deploy.yml
aliases:
  console: app exec -i --reuse "bin/rails console"
  ssh: app exec -i --reuse /bin/bash

# config/deploy.production.yml
env:
  clear:
    RAILS_ENV: production
```

### Smart Seed Data

```ruby
# db/seeds.rb
def create_tenant(name, bare: false)
  if bare
    # Minimal tenant without external dependencies
    Account.create(name: name, external_id: SecureRandom.hex)
  else
    # Full tenant with integrations
    Account.create_from_external_service(name)
  end
end

create_tenant "cleanslate", bare: true  # Works offline
create_tenant "full-featured"            # Needs services
```

### Configuration Principles

1. **Default to production** - Beta/staging inherit production settings
2. **ENV beats config** - Runtime configuration wins
3. **One secret per env** - `RAILS_MASTER_KEY` unlocks credentials
4. **Self-contained tests** - No external secrets needed
5. **Flexible development** - Support minimal and full setups
