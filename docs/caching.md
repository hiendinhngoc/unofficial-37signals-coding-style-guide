# Caching

> Read this when optimizing performance with HTTP caching (ETags) or fragment caching.

---

## Overview

Design caching early - it reveals architectural issues. Use ETags for HTTP caching, fragment caching for views.

## Key Patterns

### HTTP Caching with ETags

```ruby
def show
  @card = Current.user.cards.find(params[:id])

  # Returns 304 Not Modified if content unchanged
  fresh_when etag: @card
end

# Multiple objects
def index
  @tags = Current.account.tags
  @boards = Current.user.boards

  fresh_when etag: [@tags, @boards]
end
```

### Fragment Caching with Context

```erb
<%# Include everything that affects output %>
<% cache [card, Current.user, timezone_from_cookie] do %>
  <%= render "cards/preview", card: card %>
<% end %>

<%# Collection caching %>
<%= render partial: "cards/preview", collection: @cards, cached: true %>
```

### Touch Chains for Invalidation

```ruby
class Comment < ApplicationRecord
  belongs_to :card, touch: true  # Card cache busts when comment changes
end

class Card < ApplicationRecord
  belongs_to :board, touch: true  # Chain continues up
end
```

### Client-Side Personalization

```erb
<%# Cache the card, personalize with Stimulus %>
<% cache card do %>
  <div data-controller="ownership"
       data-ownership-creator-value="<%= card.creator_id %>"
       data-ownership-current-user-value="<%= Current.user.id %>">
    <button data-ownership-target="ownerOnly" hidden>Delete</button>
  </div>
<% end %>
```

```javascript
// Show/hide based on ownership client-side
export default class extends Controller {
  static targets = ["ownerOnly"]
  static values = { creator: Number, currentUser: Number }

  connect() {
    if (this.creatorValue === this.currentUserValue) {
      this.ownerOnlyTargets.forEach(el => el.hidden = false)
    }
  }
}
```

## Rules

- [ ] Use `fresh_when etag:` for HTTP caching
- [ ] Don't HTTP cache pages with forms (CSRF tokens get stale)
- [ ] Include rendering context in cache keys: `[card, Current.user, timezone]`
- [ ] Use `touch: true` chains for cache invalidation
- [ ] Extract dynamic content to Turbo Frames to preserve parent cache
- [ ] Use Stimulus to personalize cached fragments client-side
- [ ] Design caching early - it reveals architectural issues

### Lazy-Loaded Content

```erb
<%# Defer expensive queries to interaction time %>
<nav data-action="mouseenter->dialog#loadLazyFrames">
  <dialog>
    <%= turbo_frame_tag "menu",
          src: menu_path,
          loading: :lazy do %>
      <%= render "menu/skeleton" %>
    <% end %>
  </dialog>
</nav>
```

### Extract Dynamic Content

```erb
<%# Assignment changes often - extract to preserve card cache %>
<% cache [card, board] do %>
  <article>
    <h2><%= card.title %></h2>

    <%= turbo_frame_tag card, :assignment,
          src: card_assignment_path(card),
          loading: :lazy %>
  </article>
<% end %>
```
