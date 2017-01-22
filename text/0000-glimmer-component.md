- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Glimmer components are the next iteration of Ember's Component API. They incorporate a number of important improvements based on the community's large body of practical experience.

# Motivation

The key benefits of Glimmer components are:

## One-way data flow

Community best-practice has evolved overwhelmingly in favor of "data down, actions up". Glimmer components make this pattern explicit by using one-way binding by default.

## Template-based control over a component's root element

A Glimmer component's own root element appears within its template. This greatly simplifies the learning experience for all HTML-competent learners who are starting with Ember. It reduces the overall API surface (no more `tagName`, `attributeBindings`, `classNameBindings`, etc).

## HTML-like syntax

A Glimmer component may be invoked like `<my-component />` instead of `{{my-component}}`. The new syntax

 - allows greater expressivity for data-dependent attributes and arguments (consider `{{my-component class=(concat "foo-" type) }}` vs `<my-component class="foo-{{type}}" />`
 - reduces ambiguity between component invocations and data bindings (is `{{your-age}}` a component invocation or a property in the local scope?).
 - is more familiar to HTML-competent learners

## Portability between traditional Ember apps and micro-apps that depend only on Glimmer

Glimmer components are designed to depend only on features of Glimmer, so they can be used in places where the full power of Ember is not needed and byte-size is at a premium.

# Detailed design

The design is divided into the following sections:

 - "Glossary" defines key concepts that will be used in the remaining sections of the design.
 - "Template Invocation Syntax" explains how to call a Glimmer component from another template.
 - "Template Definition Syntax" explains what goes inside a component's own template.
 - "Javascript Syntax" addresses concerns like "can my component's class be a plain old ES2015 `class`? What ECMA features can we use to define component's implementation?"
 - "Javascript API" explains the semantics of the code that you would put into a component's Javascript module.
 - "Ecosystem adoption" explains how both application authors and addon authors can gradually adopt Glimmer components without causing incompatibilities.

## Glossary

* **attribute**: An HTML attribute. Present in the DOM with a name and optional string value.
* **argument**: A value passed into a component by the component's caller. The
  "data" in "data down".
* **fragment**: A component that results in zero or more than one top-most DOM elements. This is analogous to an `Ember.Component` with `tagName: ""`.

## Template Invocation Syntax

The naming rules for Glimmer components follow the existing rules for component names: They must contain a dash or be namespaced via `::` as described in the [Module Unification RFC](https://github.com/emberjs/rfcs/blob/6caed4c882e7be94af1d1e9fb0d723db725c691c/text/0143-module-unification.md). These requirements prevent any component from colliding with a future HTML tag. Dashes are reserved for custom elements within the HTML specification, and `::` is not part of a valid HTML tag name.

Like HTML tags, Glimmer components have both an abbreviated self-closing form and an expanded form:

```hbs
<power-select/>
<power-select></power-select>
```

### Attributes

Attributes on Glimmer components are true HTML attributes. They will be reflected into the DOM. So:

```hbs
<power-select class="important" data-foo={{foo}} />
```

expresses the intention to set the `class` and `data-foo` HTML attributes on the top-most HTML element defined by the `power-select` component. The point of this feature is two-fold.

First, it avoids the need for every component author to explicitly list every possible HTML attribute they want to expose to callers. In Ember components the `attributeBindings` API is used for this end, and in practice developers find that API unwieldy and difficult to master.

Second, it harmonizes the behavior between the component invocation case and
plain element invocation. For example to set a `class` on a Glimmer component
the following syntax is used:

```hbs
<power-select class="important">
```

And for HTML elements the syntax is shared:

```hbs
<div class="important">
```

Fragments, Glimmer components that do not have a topmost element, cannot automatically reflect HTML attributes. It is not an error to pass attributes to a fragment, they just have no default behavior. For more details about how a component author can decide which element receives attributes, see "Template Definition Syntax".

In HTML, every attribute value is necessarily either a string or completely absent. For these values, Ember templates will match HTML semantics (for example `checked=""` sets an attribute `checked` with the value `""`). However in Ember templates, we can dynamically bind arbitrary value types to an attribute. For non-string types (booleans, numbers, functions) the framework must define semantics for those types.

* **boolean values**: These values express the addition or removal of the attribute. So given `isReady = true`, then
  ```hbs
  <my-checkbox checked={{isReady}} />
  ```
  will result in the HTML attribute `checked` being added, not `checked="true"`. When the value is `false`, the `checked` attribute will be absent or removed.
* **function values**: These values result in a property set on the HTML Element. So:
  ```hbs
  <some-component onclick={{myFunction}}
  ```
  Sets a property `element.onclick = myFunction`. Because DOM properties cannot
  be represented in HTML, functions set this way have no effect when using
  Fastboot.

TODO: I want the above attribute value semantics to be the same for glimmer components vs plain elements. But we are constrained in the semantics we can change on plain elements, since they are existing API. Are there inconsistencies between my proposal and the existing plain-element semantics?

Note that the use of quotation marks necessarily casts a value to string, so

```hbs
<some-component checked="{{isTrue}}" />
```

would result in `checked="true"` despite `isTrue` having a boolean value. This is not new, it's the existing semantics of string concatenation in Ember templates.

TODO: we should discuss attribute name normalization. It should match the HTML spec. I think Kris had insights on this.

### Arguments

You can pass arguments, data for rendering, to Glimmer components using the `@` syntax:

```hbs
<power-select @options={{availablePeople}} @selected={{currentPerson}} />
```

Attributes and arguments are not the same thing. Traditional Ember components blur this distinction -- they really only have arguments, but by convention often treat some of those arguments as attributes too. Glimmer components make the distinction crisp.

The semantics of arguments are defined by the component author. They don't have any HTML reflection. It is up to the component author to use them for something.

Argument binding is one-way. If the value in the caller changes, the callee will receive the changed value. The callee has no mechanism to set the value for the caller.

TODO: do we still want opt-in two-way binding, ala `mut`? What is our solution for the case where a child component truly has higher-fidelity state than its parent? How do we avoid echoes despite arbitrary asynchrony?

### Blocks

Glimmer components accept blocks and yield block parameters, just like traditional Ember components:

```hbs
<power-select as |choice|>
  {{choice}}
{{else}}
  No choices available.
</power-select>
```

TODO: this else-block syntax is a strawman -- what should it really be? `<else>` could become a future HTML tag. `{{else}}` seems weird because it doesn't match the surrounding style. `<power-select-else>` seems terribly verbose.

### Interoperability

Glimmer components can invoke traditional Ember components. Traditional Ember components can invoke Glimmer components.

Glimmer components can still be invoked using curly syntax! This is important for easy compatibility, and it may be desirable for some kinds of components to always be used via curlies (particularly components that are providing control flow behaviors like `liquid-if` or custom forms of `#each`).

When invoked in curly syntax, you get only arguments, not attributes (since we can't distinguish the two). Your bindings are still one-way. TODO: am I missing any edge cases here? Is there a reason to make component authors explicitly declare which invocation syntax they support?

## Template Definition Syntax

Just like traditional Ember components, Glimmer components may have a template, a Javascript module, or both. This section of the design discusses the contents of a Glimmer component's template.

Unlike traditional Ember components, Glimmer components have no implicit top-level element. What you see in the template is what you get. Given a `my-message` component with this template:

```hbs
<span style="color:red">
  {{yield}}
</span>
```

If you call it this like:

```hbs
<my-message class="important" >
  Hello world
</my-message>
```

It will result in DOM like this:

```hbs
<span style="color:red" class="important">
  Hello world
</span>
```

### Fragments

Components with zero elements (a template with out text or value rendeing) or
multiple root elements are called "fragments". Fragments disregard attributes
since the root elements is missing or ambiguous.

To allow passed attributes to attach to the DOM, a frament can use the
`@@attibutes` syntax to choose which element will reflect them. This syntax
means "pass all the attributes I was given onto this element". For example this
invocation:

```hbs
<some-component data-name="nicki">
```

And this definition template:

```hbs
<h1>Name</h1>
<div @@attributes>Coming soon!</div>
```

Would render the following HTML:

```html
<h1>Name</h1>
<div data-name="nicki">Coming soon!</div>
```

### Components and web components as root elements

In some cases the root element of a template may itself be another component or
web component. This is valid. For example this invocation:

```hbs
<article-wrapper class="spotlight">
  {{body}}
</article-wrapper>
```

And this definition template:

```hbs
<article-body>
  {{yield}}
</article-body>
```

Where `article-body` is either another Ember component or a web component. In
this case `@@attributes` is again useful for passing the attributes of an
invocation down. For example, again our invocation template:

```hbs
<article-wrapper class="spotlight">
  My article body
</article-wrapper>
```

And this definition template for `article-wrapper`:

```hbs
<article-body @@attributes>
  {{yield}}
</article-body>
```

And this definition template for `article-body`:

```hbs
<div>{{yield}}</div>
```

Would result in this HTML:

```html
<div class="spotlight">
  My article body
</div>
```

Note that `@@attributes` only passes down HTML attributes. Arguments are cannot
be passed down in a batch, and instead each `@arg=` must be passed down
explicitly.

### Attribute Merging

Since both the caller of a component and the author of a component may choose to set the same HTML attributes, we define the following attribute merging rules.

  1. `class` is concatenated with a space separator, so that both caller and author may independently control their own list of class names and there is no conflict. Any name that appears in either place will appear on the final HTML element.
  2. `style` values are merged on a property-by-property basis, with the caller taking precedence over the author.
  3. For any other attribute, the caller's value takes priority.

Here's an example that demonstrates all three rules:

```hbs
<div style="width:100%; color:red" aria-role="dialog" class="message-box"></div>
```

And the invocation:

```hbs
<the-component style="color:blue" aria-role="alert" class="warning" />
```

The resulting HTML would be:

```hbs
<div style="width:100%; color:blue" aria-role="alert" class="message-box warning"></div>
```

TODO: we need a more complete definition of the style merging rule that cover the whole spec for what is legal within `style` attributes. See https://www.w3.org/TR/css-syntax-3/, in particular the [parse a list of declarations](https://www.w3.org/TR/css-syntax-3/#parse-a-list-of-declarations0) parser entry point, which is the one relevant to HTML style attributes. Thankfully this only pulls in a subset of the entire CSS syntax.

### Recursive Invocation

It is possible to define a glimmer component with an HTML custom element for a root element. For example, if you want your `display-folder` component to actually result in the DOM custom element `<display-folder>`, you can say so within the component definition:

```hbs
<display-folder>
  {{yield}}
</display-folder>
```

However, it should also be possible for a component to recursively invoke itself. This would appear to conflict with the above example (we don't want display-folder to infinitely recurse -- within its own definition, `display-folder` is actually an HTML element name, not a component invocation).

Therefore, we make a rule that *the top-level element in a component's template is never treated as an invocation of that template itself.*

This allows both use cases to coexist: the top-level element defines the component's HTML tag, while any inner uses of its own name are recursive invocations:

```hbs
<display-folder>
 {{#each @folder.children as |child|}}
   {{#if child.isFolder}}
     <display-folder @folder={{child}} />
   {{else}}
     <div>{{child.name}}</div>
   {{/if}}
 {{/each}}
</display-folder>
```

TODO: should we support a dynamic tag name? Like

```hbs
<{{tagName}}>
</{{tagName}}>
```

### Fragments

Fragment components are declared by using the special top-level element `<fragment>`

```hbs
<fragment>
  I am a fragment
</fragment>
```

This is special because `<fragment>` is *not* output to the DOM. It is elided. Invoking the above component results in only an HTML text node, no elements at all.

An attractive alternative would be to allow fragments to have no element at all

```hbs
I am a fragment
```

Or many top-level elements

```hbs
<div></div>
<div></div>
```

And thus implicitly allow any component to become a fragment if it doesn't satisfy the "one top-level element" rule. However, this would have significant downsides:

* the addition of an extra element at the end of a template suddenly changes the whole meaning of your component from "normal component" to "fragment component". Since their semantics differ in important ways, this will cause surprising breakage, like missing HTML attributes. TODO: the attribute ambiguity problem could go away if we make it an error to pass attributes to a fragment unless the fragment explicitly uses `@@attributes`.
* fragments need to allow component invocation as their first element. If that first element is not contained within a `<fragment>` marker, it creates a treacherous difference from the rule defined in the Recursive Invocation section ("the top-level element in a component's template is never treated as a component invocation"). TODO: we could address this concern by replacing that rule with a different syntax that prevents invocation.

TODO: if we can statically address the downsides of implicit fragments, it would be really nice to not require `<fragment>`, especially to support people who are just breaking up existing piles of HTML into templates.

### Referring to Arguments

Within a component's template, its arguments are accessible via `@` syntax:

```hbs
<div>
  {{@message}}
</div>
```

This is intended to support the mental connection between invocation and definition. And it makes it clear to the reader (and the Glimmer runtime, for optimization purposes) which values came from outside the component.

## Javascript Syntax

Should the distinction between traditional components and Glimmer components have anything to do with the distinction between Ember's pre-ES2015 class system and ES2015 classes?

No, because ES2015 class syntax is defined as syntactic sugar for regular old ES5 prototypical inheritance. In other words, the choice of switching to Glimmer components and the choice of switching to ES2015 classes are necessarily independent of each other. It should totally be possible for current Ember.Components to use ES2015 class syntax, so it can't be a Glimmer-component-specific thing.

That said, we should strive for consistent coding style throughout community documentation and learning materials, and teaching Glimmer components in terms of ES2015 classes seems like a good idea.

## Javascript API

### Base Class

TODO. Constraints:

* For future compatibility, components need to at least be unambiguously tagged as components. So exporting a plain function or arbitrary class would not be sufficient.
* We could require only a class decorator
* Or we could have a base class
* Must ensure that the implementation can avoid de-opts when hooks are only sometimes present.
* Pretty clear that base will not be current Ember.Object. It will be a subset that we want to work within pure Glimmer.

### Lifecycle Hooks

TODO. Document the lifecycle hooks and rationalize their names.

* "attrs" in hook names should become "args" (`didReceiveAttrs` -> `didReceiveArgs`)
* `didInsertElement`/`willDestroyElement` are now weird names relative to the more modern hook naming pattern. Plus fragments don't have an element, and it would be nice to have hooks that make sense in that case too.

### Arguments

The arguments are available as properties on `this.args`. They are implemented as getters so that unconsumed arguments may never need to be computed. They are not writable (because they are one-way bound from the outside -- you can't change the arguments you were given).

TODO: if dependent keys are still a thing, should we allow both `args.foo` and `@foo` as dependent keys? We need to at least error intelligently, because the `@` seems like an obvious mistake to make.

### Attributes

Attributes that were passed into the component by its caller are available as read-only properties on `this.attributes`. This has a reasonable symmetry with the `@@attributes` template syntax.

Attributes set on the top-level element within a component's own template are not included in `this.attributes`. It is analogous to `this.args` -- it exists to tell you what your caller was asking for.

### Blocks

TODO: `hasBlock`?

### DOM access

`this.element` is your HTML element.

Accessing `this.element` from within a fragment should throw a helpful error.

`this.firstNode` and `this.lastNode` are the first and last HTML Nodes of this component. They are valid for both non-fragment and fragment components.

TODO: do we want `this.$()`, and if so should we limit how much of jQuery's API we are leaking into our own?

### Change Tracking and Derived State

TODO.

How much of current `computed` do we want to bake into Glimmer? Can we do more automatic tracking?

Plain getters are ideal from a developer ergonomics standpoint. But without some smart dependency tracking, they spread contagious volatility throughout the component hierarchy.

It would not be unreasonable to adopt a rule like Vue, which says that all automatically tracked properties need to be present at initialization (so that getters can be installed for them).

Can we drop `this.get`? Likely yes.

Can we drop `set`? Probably not?

### Non-exhaustive list of things not in Glimmer components

Observer support.

Event support. (This may be debatable -- if we keep event support, lets make sure it's an explicitly designed API and not sneaking in any existing semantics)

All APIs for managing the top-level element (tagName, attributeBindings, classNameBindings, ariaRole, isVisible).

actions hash (is this relevant for closure actions? if yes, we should replace with a decorator instead.)

elementId: not a thing. (We can do event dispatch with a WeakMap over Elements instead). People should do whatever they want with their Element id, it doesn't have special semantics or restrictions in Glimmer components.

event methods (`click`, `keyDown`, etc): not a thing. You can use element modifiers within the template instead. TODO: should we offer a decorator alternative too?

`get`. TODO: this is tentatively gone but should be part of the change tracking discussion.

### Interoperability

TODO: can we allow a base component to upgrade to Glimmer component while still being `extend`ed by traditional components? What about the reverse?

## Ecosystem Adoption

The upgrade path within an application should be straightforward: you can begin writing new components as Glimmer components right away, and change nothing else until/unless you choose to rewrite existing components.

TODO. How do you upgrade an addon to use glimmer components while maintaining the best possible cross-version support? Can we write a polyfill so that authors can just write glimmer components but still get components that are usable in apps that don't have the glimmer component feature yet?

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
