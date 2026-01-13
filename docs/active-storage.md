# Active Storage

> Read this when implementing file uploads, image processing, or avatar handling.

---

## Overview

Variant preprocessing, direct upload configuration, and performance optimizations.

## Key Patterns

### Variant Preprocessing

```ruby
# Use preprocessed: true to avoid on-the-fly transformations
# (Prevents failures on read replicas)
has_many_attached :embeds do |attachable|
  attachable.variant :small,
    resize_to_limit: [800, 600],
    preprocessed: true
end
```

### Direct Upload Expiry

```ruby
# Extend expiry for slow uploads (Cloudflare buffering)
# config/initializers/active_storage.rb
module ActiveStorage
  mattr_accessor :service_urls_for_direct_uploads_expire_in,
    default: 48.hours
end
```

### Large File Preview Limits

```ruby
module ActiveStorageBlobPreviewable
  MAX_PREVIEWABLE_SIZE = 16.megabytes

  def previewable?
    super && byte_size <= MAX_PREVIEWABLE_SIZE
  end
end
```

### Avatar Optimization

```ruby
def show
  if @user.avatar.attached?
    # Redirect to blob URL - don't stream through Rails
    redirect_to rails_blob_url(@user.avatar.variant(:thumb))
  else
    render_initials if stale?(@user)
  end
end
```

## Rules

- [ ] Use `preprocessed: true` for variants (prevents read replica failures)
- [ ] Extend direct upload expiry to 48 hours (Cloudflare buffering)
- [ ] Skip previews above size threshold (e.g., 16MB)
- [ ] Redirect to blob URL instead of streaming through Rails
- [ ] Use mirror service for redundancy without impacting reads
- [ ] Use :thumb variant for consistent avatar sizing

### Mirror Configuration

```yaml
# config/storage.yml
mirror:
  service: Mirror
  primary: local
  mirrors: [s3_backup]

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

s3_backup:
  service: S3
  bucket: myapp-backups
```

### Preview vs Variant

```ruby
# Images: use variant
blob.variant(resize_to_limit: [800, 600])

# PDFs, videos: use preview
blob.preview(resize_to_limit: [800, 600])

# Don't conflate them - different operations
```
