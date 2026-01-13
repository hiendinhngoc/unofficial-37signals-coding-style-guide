# Webhooks

> Read this when implementing outbound webhooks for third-party integrations.

---

## Overview

Webhooks require careful security (SSRF protection), reliability (state tracking), and user experience (delinquency handling).

## Key Patterns

### SSRF Protection

```ruby
module SsrfProtection
  DISALLOWED_IP_RANGES = [
    IPAddr.new("0.0.0.0/8"),
    IPAddr.new("10.0.0.0/8"),
    IPAddr.new("172.16.0.0/12"),
    IPAddr.new("192.168.0.0/16"),
    IPAddr.new("169.254.0.0/16"),  # Link-local (AWS IMDS)
  ].freeze

  def resolve_public_ip(hostname)
    ips = Resolv::DNS.new.getaddresses(hostname)
    public_ips = ips.reject { |ip| private_address?(ip) }
    public_ips.first&.to_s
  end

  def private_address?(ip)
    ip = IPAddr.new(ip.to_s)
    ip.private? || ip.loopback? || ip.link_local? ||
      DISALLOWED_IP_RANGES.any? { |range| range.include?(ip) }
  end
end

# Pin resolved IP to prevent DNS rebinding
def http
  Net::HTTP.new(uri.host, uri.port).tap do |http|
    http.ipaddr = resolved_ip  # Pin to resolved IP!
    http.open_timeout = 7.seconds
    http.read_timeout = 7.seconds
  end
end
```

### Delivery State Machine

```ruby
class Webhook::Delivery < ApplicationRecord
  enum :state, %i[pending in_progress completed errored], default: :pending

  store :request, coder: JSON
  store :response, coder: JSON

  after_create_commit :deliver_later

  def deliver
    in_progress!
    self.response = perform_request
    completed!
    webhook.delinquency_tracker.record_delivery_of(self)
  rescue => e
    errored!
    raise
  end

  def succeeded?
    completed? && response[:code]&.between?(200, 299)
  end
end
```

### Delinquency Tracking

```ruby
class Webhook::DelinquencyTracker < ApplicationRecord
  THRESHOLD = 10
  DURATION = 1.hour

  def record_delivery_of(delivery)
    if delivery.succeeded?
      reset
    else
      increment!(:consecutive_failures_count)
      webhook.deactivate if delinquent?
    end
  end

  def delinquent?
    consecutive_failures_count >= THRESHOLD &&
      first_failure_at&.before?(DURATION.ago)
  end
end
```

### HMAC Signature

```ruby
def headers
  {
    "Content-Type" => "application/json",
    "X-Webhook-Signature" => signature,
    "X-Webhook-Timestamp" => Time.current.utc.iso8601
  }
end

def signature
  OpenSSL::HMAC.hexdigest("SHA256", webhook.signing_secret, payload)
end
```

## Rules

- [ ] SSRF protection: resolve DNS upfront, pin IP, block private ranges
- [ ] State machine: pending → in_progress → completed/errored
- [ ] Delinquency tracking: auto-disable after 10 failures over 1 hour
- [ ] Sign payloads with HMAC-SHA256
- [ ] Two-stage jobs: dispatch job → delivery job
- [ ] Auto-detect format from URL (Slack, Campfire, etc.)
- [ ] Auto-cleanup old deliveries (7 days)
- [ ] Limit response size to prevent memory exhaustion
- [ ] Set reasonable timeouts (7 seconds)

### Two-Stage Job Pattern

```ruby
# Stage 1: Dispatch
class Event::WebhookDispatchJob < ApplicationJob
  def perform(event)
    Webhook.active.triggered_by(event).find_each do |webhook|
      webhook.trigger(event)  # Creates delivery, enqueues stage 2
    end
  end
end

# Stage 2: Deliver
class Webhook::DeliveryJob < ApplicationJob
  def perform(delivery)
    delivery.deliver
  end
end
```

### Automatic Cleanup

```ruby
class Webhook::Delivery < ApplicationRecord
  STALE_THRESHOLD = 7.days
  scope :stale, -> { where(created_at: ...STALE_THRESHOLD.ago) }

  def self.cleanup
    stale.delete_all
  end
end

# config/recurring.yml
cleanup_webhook_deliveries:
  command: "Webhook::Delivery.cleanup"
  schedule: every 4 hours
```
