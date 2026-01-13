# Observability

> Read this when setting up logging, metrics, or monitoring infrastructure.

---

## Overview

Structured JSON logging, Yabeda metrics, and console auditing for compliance.

## Key Patterns

### Structured JSON Logging

```ruby
# config/environments/production.rb
config.log_level = :fatal  # Suppress unstructured logs
config.structured_logging.logger = ActiveSupport::Logger.new(STDOUT)

# Gemfile: gem "rails_structured_logging"
```

### Multi-Tenant Context

```ruby
# Inject tenant into every log
before_action do
  logger.struct tenant: Current.account&.id
end
```

### User Context

```ruby
def set_current_session(session)
  logger.struct "Authorized User##{session.user.id}",
    authentication: { user: { id: session.user.id } }
end
```

### Yabeda Metrics Stack

```ruby
# Gemfile
gem "yabeda"
gem "yabeda-rails"
gem "yabeda-puma-plugin"
gem "yabeda-prometheus-mmap"
gem "yabeda-activejob"
gem "yabeda-gc"

# config/puma.rb
plugin :yabeda
plugin :yabeda_prometheus

on_worker_boot do
  Yabeda::ActiveRecord.start_timed_metric_collection_task
end
```

### Console Auditing

```ruby
# For compliance - audit all console access
# Gemfile: gem "console1984", gem "audits1984"

config.console1984.protected_environments = %i[production staging]
config.audits1984.base_controller_class = "AdminController"
```

## Rules

- [ ] Use `rails_structured_logging` for JSON logs
- [ ] Inject tenant into every log entry
- [ ] Log authenticated user context
- [ ] Use Yabeda stack for metrics
- [ ] Silence health checks: `config.silence_healthcheck_path = "/up"`
- [ ] Separate logger for jobs: `config.solid_queue.logger = ...`
- [ ] Use `console1984` for console auditing in production
- [ ] Deploy OpenTelemetry collector as sidecar for container metrics

### Log Configuration

```ruby
# Silence health checks
config.silence_healthcheck_path = "/up"

# Separate logger for background jobs
config.solid_queue.logger = ActiveSupport::Logger.new(STDOUT, level: :info)
```

### ActionCable Metrics

```ruby
# Gemfile: gem "yabeda-actioncable"

# config/recurring.yml
yabeda_actioncable:
  command: "Yabeda::ActionCable.measure"
  schedule: every 60 seconds
```
