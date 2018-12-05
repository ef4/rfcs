- Start Date: 2018-12-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes allowing complete ECMA `import` semantics from inside Ember templates.

# Motivation

[Module Unification, RFC 143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md) designed a new standard layout for Ember apps. Unfortunately, before it shipped the requirements shifted beneath us, as scoped NPM packages became increasingly popular, which interfered with our syntactic design. While "how to access a component from an addon package" may seem like only one small piece of MU, it's a critical piece that is central to the whole model and the goal of eliminating global namespace collisions.

[Module Unification Packages, RFC 367](https://github.com/emberjs/rfcs/pull/367) describes this problem in detail and proposes a solution to it in the form of `use`. The name `use` was chosen over `import` because it has different semantics from Javascript's `import`.

Community feedback on that RFC, and increasingly clear adoption trends in the wider Javascript ecosystem have led us to question again: "but why _not_ make the full JS `import` work in our templates?".

This design explains how we could choose to do that, and what it means for amending both of those earlier RFCs. Overall, I think it results in a clearer design.

# Detailed design

## Primitives vs Guidelines

First, let's delinate two separate layers of concern:

 - **Core Primitives**: the underlying rules that govern how the system works. These need to be clear and robust, and preferrably standards-driven.
 - **Best-practice Guidelines**: the defaults, nudges, and lints that we use to help Ember developers be productive and avoid every team wasting time bike-shedding their own ways of doing things.

Sometimes, these two layers can be one-in-the-same. If you're designing a new system in isolation, you can make "the conventional way" be "the only way". But we don't live in isolation, we are part of a wider Javascript language and tooling ecosystem. There are benefits to making our system a subset of the broader system:

 - IDEs, web-based development tools, libraries, and code analysis tools will work better with Ember apps if we stay aligned with the most broadly-adopted rules.
 - Javascript developers need to acquire less Ember-specific knowledge if the invariants they've already learned elsewhere also remain true in Ember apps.

By "subset", I mean that we can make our **Core Primitives** follow broadly-compatible Javascript conventions and layer our own **Best-practice Guidelines** on top of that. For example: Javascript lets you `import` a module from anywhere that you'd like to put it, and we shouldn't actively break that. But we can provide blueprints and lint rules that help developers choose a good _conventional_ place to put their module.

## ECMA Modules with node_modules Resolution

We we were early adopters of the ECMA module specification, and since then the rest of the JS ecosystem has also thoroughly embraced it. But the module spec by itself doesn't say anything about how a particular _ModuleSpecifier_ get resolved into an actual source code file.

```
import Component from "@ember/component";
//                    |----------------|
//                      ModuleSpecifier
```

Meanwhile, the rapid adoption of NPM for browser-Javascript package management and Node as a build toolchain for browser-Javascript has caused Node's `node_modules` resolution algorithm to become the _de facto_ standard resolver. Most popular editors and Javascript tools understand this convention.

Even in cases where people are proposing alternative resolvers (such as Yarn Plug and Play), there is a clear expectation that code that delegates resolution to Node will continue to work (Yarn Plug and Play patches node's resolver to achieve compatibility).

Node's resolution implies that relative specifiers (like "./some/thing") work the way you would expect, and absolute specifiers (like "@ember/component") refer to an NPM package that can be found following the `node_modules` filesystem rules (the particular rules are not super relvant to our discussion -- it's enough to decide to delegate resolving to this _de facto_ standard implementation).

It's also important to note that tools using node_modules resolution widely support the ability to plug in handlers for alternative file extensions. `.ts`, `.css`, `.jsx`, and `.json` are all commonly resolved using the same resolution system. How each is intereted as a Javascript module is a pluggable decision that we are free to use to our advantage.

But it's also important to note what is _not_ as easily pluggable without doing idiosyncratic, tool-specific work: changing the core way that `ModuleSpecifiers` are mapped into files.

For example, making `import Something from "./some/thing"` resolve to `./some/thing.hbs` is a pretty widely-supported pattern. Whereas making it refer to `./some/thing/template.hbs` is not as widely-supported, and is likely to require a lot of tool-specific customizations.

## Resolution in Ember's Javascript

This RFC is not about changing the way we do resolution between Javscript modules. Closing the remaining gaps between our existing semantics and the system just described is an active area of work that is making rapid progress and deserves its own RFC(s).

Our main focus is on resolution in templates.

## Resolution in Ember's Templates

Today, any component or helper used in a template is located via the Ember resolver. Its limitations are thoroughly described in [Module Unification, RFC 143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md). Specifically, it has no locality (there's no such thing as relative lookup, so many things end up globally-scoped) and it has no notion of packages (all components and helpers from addon packages go into one big global namespace soup).

The first MU design attempted to fix this with strict filesystem layout rules and some additional syntax for packages. This makes the filesystem layout extremely high stakes, as it becomes "the only way to do things", so it took a lot of work to get concensus, and there were necessarily compromises (like the dash-prefixed collection directories). [Module Unification Packages, RFC 367](https://github.com/emberjs/rfcs/pull/367) makes a "small" extension to that plan by introducing `use` to fix the package syntax problem.

This RFC proposes an alternative solution: once you accept the need for a `use`-like construct, there's little reason not to go all the way to using `import` and make it fully compatible with "ECMA modules with node_modules resolution" as described above. And once you choose that, you now have a flexible **Core Primitive** that takes some pressure off the filesystem layout design -- many design affordances become a question of **Best-practice guidelines** rather than hard implementation.

Concretely: templates under a /src directory must use `import` to locate other components and helpers.

## Frontmatter syntax

Imports appear within a new frontmatter section at the beginning of a template:

```hbs
---
import Button from './button';
import { Select } from 'ember-power-select';
---
<Select />
<Button />
```

The frontmatter section (delimited by `---`) uses Javascript syntax, but is limited to only allow import statements.

This limitation could be lifted by a followup RFC, which would require clarifying the semantic boundary between names in the JS and names in the hbs.

## Auto-insertion of imports

You don't need to import built-in constructs that ship with Ember (like `if`) or Ember's built-in helpers (like `action`).

To maintain framework extensibility and prevent these built-in from being so "magical", we offer a clear path for apps to register similar global names:

```js
//config/template-globals.js
export default {
  t: "import { t } from 'ember-intl'"
};
```

This says "whenever the template compiler encounters the undefined name `t`, it should automatically insert `"import { t } from 'ember-intl'"` at the start of the template. This only applies to otherwise _undefined_ names: if a template creates its own binding for `t`, that takes precedence.

Addons are deliberately _not_ allowed to register themselves into the template-globals. They can offer a generator that helps app-authors edit `template-globals.js`, but control over globals remains _explicit in the app_, so that there's only one place where you need to look to find where a global name is coming from.

## Rationalizing Components as Modules

A key piece of this design is explaining how to interpret Ember components as Javascript modules. There are some important requirements:

 - When you import a component from Javascript, you should get the values back you'd expect for a normal Javascript module. We don't want to rewrite things in an opaque way.
 - Consumers of a component should not need to know whether it is implemented as only a template, a template plus Javascript, or only Javascript.
 - A component that is reexported should continue to work unchanged (its template had better follow along automaticaly).
 - We should conform with "ECMA modules with node_modules resolution" as described above, which means we are not free to choose how ModuleSpecifiers map into actual files, but we _are_ free to choose how non-JS files are interpreted as JS modules.

Given these requirements, the proposed design is:

1. Imports are resolved following node's resolution strategy. We register a custom handler for `.hbs` files. `.js` files take precedence over `.hbs` files, such that if both exist for a given ModuleSpecifier, the JS wins.

2. Component Javascript should be authored using default exports (as they already are):

    ```js
    export default class extends Component {
      yourStuffHere(){}
    };
    ```

    When that exported value is imported from a template, the value you get is the actual class (no magic here).

3. When both `.js` and `.hbs` exist for a component, the build unobtrusively associates the template with the default value exported by the Javascript. The template must be co-located with the Javascript, having the same filename except for changing the extension to `.hbs`.

    For illustrative purposes, here's a toy implementation of how we can do "unobtrusive" association of the template with the Javascript value:

    ```js
    // the previous example code would get rewritten to something like this:
    import { registerTemplate } from '@ember/hypothetical-private-api';
    export default registerTemplate(class extends Component {
      yourStuffHere()
    }, compiledTemplateAndMetadataHere);
    ```

    where the implementation of `registerTemplate` effectively does:

    ```js
    const templates = new WeakMap();
    function registerTemplate(value, templateDetails) {
      templates.set(value, templateDetails);
      // notice that the value is unchanged. That's why I mean by "unobtrustive".
      // You can still import this value into both templates and javascript and
      // get what you expected to get.
      return value;
    }
    ```

    Please note this is only an example to show that unobtrusive association is possible. The exact details would be private API.

4. When only a template exists, it gets resolved following the normal node_modules resolution system as described in step 1. Our handler for `.hbs` files causes them to be interpreted as Javasript modules like:

    ```js
    // for illustrative purposes only
    import { registerTemplate, useDefaultComponentManager } from '@ember/hypothetical-private-api';
    export default registerTemplate(useDefaultComponentManager, compiledTemplateAndMetadataHere);
    ```

    The point of this is that you can import and invoke the template-only component in the same way you would a template-and-javascript component. In both cases, from glimmer's perspective the actual value that you import is used to lookup the component's metadata & template in a shared, behind-the-scenes WeakMap.

5. Helpers are resolved just like component Javascript. They are simpler because there is no template association to worry about.

## Implications

Given the system described above, several patterns _necessarily_ become possible. Specifically, this component invocation:

```
---
import Button from './path/to/button';
---
<Button />
```

Should work correctly for all of the following situations:

1. `./path/to/button.hbs` is a template-only component.
2. `./path/to/button/index.hbs` is a template-only component.
3. `./path/to/button.js` exports a component class and `./path/to/button.hbs` is its template.
4. `./path/to/button/index.js` exports a component class and `./path/to/button/index.hbs` is its template.

Now, for anti-bikeshedding and aesthetic reasons we may want to lint against some of these patterns. But they are all necessarily supported by virtue of us not wanting to break the node_modules resolution rules. The filesystem layout becomes a **Best-practice guidance** concern, not a **Core primitives** concern.

Another thing that necessarily must work is reexporting components as named exports. Given this module:

```js
export { default as Select } from './select';
export { default as MultiSelect } from './multi-select';
```

You can use the named exported components:

```hbs
---
import { Select, MultiSelect } from 'that-module';
---

<Select />
<MultiSelect />
```

We require component Javascript to be _authored_ using default exports (because we associate templates with default exports), but the resulting components can still be grouped and shared via named exports.

Another nice feature that necessarily works in this model is that helpers can be named exports grouped into one file:

```js
import { helper } from '@ember/component/helper';

const gt = helper(function([left, right])  {
  return left > right;
}

const lt = helper(function([left, right]) {
  return left < right;
}

export { gt, lt };
```

```hbs
---
import { gt, lt } from 'the-helpers-module';
---
{{#if (gt this.thing 10)}}
  more than 10
{{/if}}

{{#if (lt this.thing 1)}}
  less than 1
{{/if}}
```

In fact, helpers can colocate with components in one file if you want them to:

```js
// my-component/index.js
import Component from '@ember/component';
import { helper } from '@ember/component/helper';

export default class extends Component({

});

export const customHelper = helper(function(){
  return 'hello world';
})
```

```hbs
{{! my-component/index.hbs }}
---
import { customHelper } from '.';
---
<div class={{customHelper}} />
```




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

## Why "foo/index.js" over "foo/component.js"?

Because `index.js` is well-understood by Javacript ecosystem tooling to be a file that will be found when you try to resolve `./foo`, whereas `foo/component.js` is not.

We get less to customize and less to teach if we stick with the better supported word. And we loose very little by picking a different constant word.

## Why "foo/index.hbs" over "foo/component.hbs"?

This is _almost_ the same argument as the previous point, but with one caveat.

It's true that when a Javascript file for the component exists, we are free to choose our own convention for which template to associate. So we _could_ make this work:

 - foo/index.js
 - foo/template.hbs

However, this creates a refactoring hazard if the developer later deletes `foo/index.js`. This is _not_ resolvable by itself using standard node_modules rules:

 - foo/template.hbs

Whereas `foo/index.hbs` is resolvable, as long as we've registered a handler for `.hbs` files.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
