# Accessibility

> Read this when building accessible UI components, keyboard navigation, or screen reader support.

---

## Overview

ARIA patterns, keyboard navigation, focus management, and screen reader considerations.

## Key Patterns

### ARIA for Decorative Elements

```erb
<%# Hide decorative images from screen readers %>
<%= image_tag "icon.svg", aria: { hidden: true } %>

<%# Icon-only buttons need labels %>
<button aria-label="Delete">
  <%= image_tag "trash.svg", aria: { hidden: true } %>
</button>
```

### Visually Hidden Text

```css
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  clip-path: inset(50%);
  overflow: hidden;
  white-space: nowrap;
}
```

```erb
<button>
  <%= icon_tag "edit" %>
  <span class="visually-hidden">Edit comment</span>
</button>
```

### Keyboard Navigation Controller

```javascript
export default class extends Controller {
  static targets = ["item"]

  navigate(event) {
    if (event.key === "ArrowDown") this.#selectNext()
    if (event.key === "ArrowUp") this.#selectPrevious()
    if (event.key === "Enter") this.#activateSelected()
  }

  #selectNext() {
    const items = this.#visibleItems
    const index = items.indexOf(this.currentItem)
    this.#select(items[(index + 1) % items.length])
  }

  #select(item) {
    this.itemTargets.forEach(i => i.removeAttribute("aria-selected"))
    item.setAttribute("aria-selected", "true")
    item.focus()
  }

  get #visibleItems() {
    return this.itemTargets.filter(i => i.checkVisibility())
  }
}
```

### Focus Ring Styles

```css
:root {
  --focus-ring-color: var(--color-link);
  --focus-ring-size: 2px;
  --focus-ring-offset: 1px;
}

:is(a, button, input):focus-visible {
  outline: var(--focus-ring-size) solid var(--focus-ring-color);
  outline-offset: var(--focus-ring-offset);
}
```

## Rules

- [ ] Use `aria-hidden="true"` for decorative elements
- [ ] Provide `aria-label` for icon-only buttons
- [ ] Use `.visually-hidden` for screen reader text
- [ ] Build navigable list controller for arrow key navigation
- [ ] Use `:focus-visible` instead of `:focus`
- [ ] Use `checkVisibility()` for accurate item detection
- [ ] Update `aria-expanded` when toggling content
- [ ] Add `aria-label` and `aria-description` to dialogs
- [ ] Adapt keyboard shortcuts by platform (Cmd vs Ctrl)
- [ ] Test with actual screen readers (VoiceOver, NVDA)

### Platform-Specific Shortcuts

```ruby
def hotkey_label(hotkey)
  hotkey.map do |key|
    if key == "ctrl" && platform.mac?
      "\u2318"  # Command symbol
    else
      key.capitalize
    end
  end.join("+")
end
```

### Dialog Focus Management

```javascript
open() {
  this.dialogTarget.showModal()
  this.dialogTarget.querySelector('input, button')?.focus()
}
```

### Form Label Associations

```erb
<%# Use form builder's field_id helper %>
<%= form.check_box :assignee_ids, { multiple: true }, user.id %>
<%= form.label :assignee_ids, user.name,
      for: form.field_id(:assignee_ids, user.id) %>
```

### Quick Wins

- [ ] Add `aria-hidden="true"` to decorative icons
- [ ] Add `.visually-hidden` labels to icon-only buttons
- [ ] Use `aria-label` on dialogs
- [ ] Fix form label `for` attributes
- [ ] Implement `:focus-visible` styles
- [ ] Add `event.preventDefault()` to keyboard shortcuts
- [ ] Use semantic HTML (`<h1>`, `<nav>`, etc.)
