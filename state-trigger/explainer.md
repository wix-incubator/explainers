# [animation-triggers-1] Proposal: Toggle Custom States Using Animation Triggers (`state-trigger`)

## Summary

This proposal introduces a new CSS property, `state-trigger`, that leverages the [animation trigger](https://drafts.csswg.org/css-animations-2/#animation-triggers) infrastructure to toggle [custom states](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/:state) on elements declaratively. Building on the [proposal to expand `:state()` to all elements](https://github.com/whatwg/html/issues/11466), this would allow authors to add or remove custom states based on scroll position, visibility, or user-interaction events.

## Motivation

### The gap between triggers and styling

The animation trigger specification ([animation-triggers-1](https://drafts.csswg.org/css-animations-2/#animation-triggers)) defines a powerful model of [timeline triggers](https://drafts.csswg.org/css-animations-2/#timeline-triggers) and [event triggers](https://drafts.csswg.org/css-animations-2/#event-triggers) that can control animation playback based on scroll position, viewport entry, clicks, and other interactions.

However, there are many use cases where the *desired reaction* to a trigger is not an animation, but a *state change* — toggling a class-like flag on an element so that different CSS rules can apply. Today, this requires JavaScript event listeners or IntersectionObservers to add/remove classes or custom states.

### Custom states are the right primitive

CSS [custom states](https://html.spec.whatwg.org/multipage/semantics-other.html#selector-custom) (`:state()`) provide a mechanism for elements to expose named boolean states that can be matched by CSS selectors. They are already supported for custom elements via `ElementInternals.states` (`CustomStateSet`), and there is an [active proposal](https://github.com/whatwg/html/issues/11466) to expand `Element.prototype.states` to all elements.

By combining triggers with custom states, we can enable a broad class of declarative, CSS-only interactions that currently require JavaScript.

### Use cases

- **Scroll-triggered state**: Mark an element as `:state(--visible)` when it enters the viewport, enabling CSS-only reveal effects, lazy styling, or conditional layouts.  
     
- **Sticky header detection**: Set `:state(--stuck)` on a sticky element when it reaches its sticking position, enabling shadow/border changes.  
     
- **Interaction-driven toggles**: Toggle `:state(--active)` on click (and untoggle on a second click), enabling accordion, tab, or disclosure patterns.  
     
- **Reading progress markers**: Mark table-of-contents items as `:state(--read)` as corresponding sections scroll past, enabling progress indicators.  
     
- **Hover/focus persistence**: Set `:state(--was-hovered)` that persists after the pointer leaves, enabling "seen" indicators or progressive disclosure.

## Proposed Solution

### New property: `state-trigger`

A new shorthand property `state-trigger` connects a named [trigger](https://drafts.csswg.org/css-animations-2/#animation-triggers) to the element it is specified on, with dedicated actions that modify the element's custom state set. This new property is a coordinated list property, allowing to set multiple triggers.

```css
state-trigger: [ [ <dashed-ident> <state-action>+ ]+ ]#
```

Where:

- `<dashed-ident>` references a named trigger (defined via `timeline-trigger` or `event-trigger` on some element)  
- `<state-action>` specifies what state modifications to perform when the trigger activates

### The `<state-action>` type

```
<state-action> = add-state(<dashed-ident>) | remove-state(<dashed-ident>) | toggle-state(<dashed-ident>) | none
```

- **`add-state(<dashed-ident>)`** — Adds the specified custom state to the element's state set.  
- **`remove-state(<dashed-ident>)`** — Removes the specified custom state from the element's state set.  
- **`toggle-state(<dashed-ident>)`** — Adds the state if absent, removes it if present.

**Note:** another option is to use `custom-ident`s instead if there’s no risk of name clash and we can achieve a similar syntax to custom states currently used in JS.

Like `animation-trigger`, trigger types that support two actions (stateful event triggers and timeline triggers) accept one or two `<state-action>` values: the first for the entry/activation action, the second for the exit/deactivation action.

### Full property definition

```
Name: state-trigger
Value: [ none | [ <dashed-ident> <state-action>+ ]+ ]#
Initial: none
Applies to: all elements
Inherited: no
Animation type: not animatable
```

## Examples

### Viewport entry — element becomes `:state(--visible)`

```css
/* Define a timeline trigger on the element itself */
.reveal {
  timeline-trigger: --on-screen view();
  state-trigger: --on-screen add-state(--visible);
}

/* Style based on the custom state — no animation needed */
.reveal {
  opacity: 0;
  transition: opacity 0.5s;
}
.reveal:state(--visible) {
  opacity: 1;
}
```

### Viewport entry/exit — toggle a state

```css
.card {
  timeline-trigger: --in-view view() entry 25% entry 75%;
  state-trigger: --in-view add-state(--in-view) remove-state(--in-view);
}

.card:state(--in-view) {
  outline: 2px solid highlight;
}
```

### Click toggle — accordion pattern

```css
.accordion-header {
  event-trigger: --toggle click;
  state-trigger: --toggle toggle-state(--open);
}

.accordion-header + .accordion-body {
  display: none;
}
.accordion-header:state(--open) + .accordion-body {
  display: block;
}
```

### Hover persistence

```css
.tooltip-trigger {
  event-trigger: --seen pointerenter;
  state-trigger: --seen add-state(--was-hovered);
}

.tooltip-trigger:state(--was-hovered)::after {
  content: "✓ Seen";
}
```

## Interaction with existing specs

### Trigger resolution

`state-trigger` uses the same trigger name resolution mechanism as `animation-trigger` (see [trigger-scope](https://drafts.csswg.org/css-animations-2/#trigger-scope)). The `<dashed-ident>` in `state-trigger` references a trigger by name, following the same scoping rules.

### Trigger actions model

The trigger infrastructure in [animation-triggers-1](https://drafts.csswg.org/css-animations-2/#animation-triggers) is designed to be generic — as noted in the spec:

*"This design for triggers and trigger instances, and the way they're associated with triggered animations and `<animation-action>`s, is intentionally somewhat generic, intended to support using triggers for other purposes in the future."*

`state-trigger` is a natural second consumer of this infrastructure, alongside `animation-trigger`. Where `animation-trigger` maps trigger activations to animation playback actions (`play`, `pause`, `reset`, etc.), `state-trigger` maps them to state mutations (`add-state`, `remove-state`, `toggle-state`).

### Dependency on expanded `:state()`

This proposal depends on custom states being settable on all elements, not just custom elements. This aligns with [whatwg/html\#11466](https://github.com/whatwg/html/issues/11466), which proposes `Element.prototype.states`. Without this expansion, `state-trigger` would only be useful on custom elements — significantly limiting its value.

### Prior Art

[CSS Toggles](https://tabatkins.github.io/css-toggle/) was a proposal for declarative state management in CSS. `state-trigger` differs from the Toggles proposal by specifically connecting the trigger infrastructure to custom element states, reusing existing trigger and `:state()` primitives.

The CSS Toggles proposal had a more elaborate set of state management actions that could be borrowed if necessary. This proposal focuses on the simpler, more narrowly scoped integration with `:state()`.

## API extension (imperative)

Using the imperative API in `animation-triggers-1` for creating triggers can probably be reused here for toggling custom states, and this would allow creating state triggers programmatically, though the primary motivation for this proposal is the declarative CSS API.

## Open issues/questions

* All names are bikeshedable.  
* Whether to use `custom-ident` or `dashed-ident` in `<state-action>`.  
* Whether we want a dedicated imperative API for this feature.