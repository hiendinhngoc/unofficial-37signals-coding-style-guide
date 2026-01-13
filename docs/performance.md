# Performance

> Read this when optimizing database queries, preventing N+1s, or tuning Ruby/Puma.

---

## Overview

N+1 prevention, query optimization, and Puma tuning for production workloads.

## Key Patterns

### N+1 Prevention

```ruby
# Use prosopite gem for detection
# Gemfile: gem "prosopite"

# Create preloaded scopes
scope :preloaded, -> {
  includes(:column, :tags, board: [:entropy, :columns])
}

# Prefer in-memory checks over DB queries
# BAD - extra query
assignments.exists?(assignee: user)

# GOOD - in-memory
assignments.any? { |a| a.assignee_id == user.id }
```

### Batch SQL Over Loops

```ruby
# BAD - N queries
cards.find_each { |card| card.mentions.destroy_all }

# GOOD - single query with JOINs
user.mentions
  .joins("LEFT JOIN cards ON ...")
  .where("cards.collection_id = ?", id)
  .destroy_all
```

### Puma Tuning

```ruby
# config/puma.rb
workers Concurrent.physical_processor_count
threads 1, 1  # Single thread per worker for CPU-bound Rails

before_fork do
  Process.warmup  # GC, compact, malloc_trim for copy-on-write
end
```

## Rules

- [ ] Use `prosopite` gem for N+1 detection
- [ ] Create `preloaded` scopes with `includes`
- [ ] Use in-memory checks over DB queries when data is loaded
- [ ] Replace `find_each` loops with JOINs for bulk operations
- [ ] Accept "unmaintainable" SQL when performance requires it
- [ ] Use counter caches for counts (but callbacks are bypassed)
- [ ] Tune Puma: `workers = CPU cores`, `threads 1, 1`
- [ ] Use `Process.warmup` in `before_fork` for copy-on-write
- [ ] Debounce filter searches (100ms feels responsive)

### CSS Performance

```ruby
# Avoid complex :has() selectors - Safari freezes
# Prefer simpler selectors over clever CSS
```

### Pagination

- [ ] Start with reasonable page sizes (25-50)
- [ ] Reduce if initial render is slow
- [ ] Use "Load more" or IntersectionObserver
- [ ] Separate pagination per column/section

### Optimistic UI

```javascript
// Insert immediately, request async
#insertDraggedItem(container, item) {
  // Insert at correct position
  container.insertBefore(item, referenceNode)
}

// Then submit
await this.#submitDropRequest(item, container)
```
