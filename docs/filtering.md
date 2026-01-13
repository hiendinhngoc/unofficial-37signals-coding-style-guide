# Filtering

> Read this when building search, filter, or faceted navigation interfaces.

---

## Overview

Filter objects, URL-based state, and Stimulus controllers for rich filtering UX.

## Key Patterns

### Filter Object (PORO)

```ruby
class CardFilter
  def initialize(user, params = {})
    @user = user
    @tag_ids = params[:tag_ids]
    @assignee_ids = params[:assignee_ids]
    @status = params[:status]
  end

  def cards
    @cards ||= begin
      result = user.accessible_cards.preloaded
      result = result.tagged_with(tags) if tags.present?
      result = result.assigned_to(assignees) if assignees.present?
      result = result.with_status(status) if status.present?
      result
    end
  end

  def tags
    @tags ||= account.tags.where(id: tag_ids) if tag_ids
  end

  def assignees
    @assignees ||= account.users.where(id: assignee_ids) if assignee_ids
  end
end
```

### URL-Based State

```ruby
module Filter::Params
  PERMITTED = [:status, :sorted_by, tag_ids: [], assignee_ids: []]

  def as_params
    {
      tag_ids: tags.ids,
      assignee_ids: assignees.ids,
      status: status
    }.compact_blank
  end

  def as_params_without(key, value)
    as_params.dup.tap do |params|
      if params[key].is_a?(Array)
        params[key] -= [value]
        params.delete(key) if params[key].empty?
      elsif params[key] == value
        params.delete(key)
      end
    end
  end
end
```

### Filter Chips as Links

```erb
<%# Use links, not forms - simpler and more accessible %>
<% filter.tags.each do |tag| %>
  <%= link_to cards_path(filter.as_params_without(:tag_ids, tag.id)),
        class: "chip" do %>
    <%= tag.name %>
    <%= image_tag "close.svg" %>
  <% end %>
<% end %>
```

### Stimulus Filter Controller

```javascript
export default class extends Controller {
  static targets = ["input", "item"]

  initialize() {
    this.filter = debounce(this.filter.bind(this), 100)
  }

  filter() {
    const query = this.inputTarget.value.toLowerCase()

    this.itemTargets.forEach(item => {
      item.hidden = !item.textContent.toLowerCase().includes(query)
    })

    this.dispatch("changed")
  }
}
```

## Rules

- [ ] Extract filtering logic to Filter POROs
- [ ] Build queries lazily with memoization
- [ ] Store filter state in URL params (stateless, shareable)
- [ ] Use links for filter chips, not forms
- [ ] Dual Stimulus controllers: filter + navigable-list
- [ ] Debounce filter input (100ms feels responsive)
- [ ] Normalize and digest params for filter deduplication
- [ ] Test filters as unit tests, not integration tests

### Controller Usage

```ruby
class CardsController < ApplicationController
  def index
    @filter = CardFilter.new(Current.user, filter_params)
    @cards = @filter.cards
  end

  private
    def filter_params
      params.permit(*CardFilter::PERMITTED)
    end
end
```

### Filter Persistence

```ruby
# Digest params for deduplication
def self.digest_params(params)
  normalized = params.compact_blank.sort.to_h
  Digest::MD5.hexdigest(normalized.to_json)
end

# Remember filter (find or create)
def self.remember(user, params)
  user.filters.find_or_create_by!(params_digest: digest_params(params)) do |f|
    f.assign_attributes(params)
  end
end
```

### View Integration

```erb
<div data-controller="filter navigable-list"
     data-action="keydown->navigable-list#navigate
                  filter:changed->navigable-list#reset">

  <%= text_field_tag :search,
        data: { filter_target: "input", action: "input->filter#filter" } %>

  <ul>
    <% @items.each do |item| %>
      <li data-filter-target="item" data-navigable-list-target="item">
        <%= link_to item.name, item %>
      </li>
    <% end %>
  </ul>
</div>
```
