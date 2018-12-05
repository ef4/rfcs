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

## Ember's Two Resolution Systems

Ember currently has two distinct resolution systems. The first, which I will call the "Module Resolver", is the one that governs how `import` in Javascript works today.  It's not exactly the same as the node_modules spec, but it's close:

 - it already incorporates key features like the `./index.js` convention.
 - it handles relative specifiers (starting with `..` or `.`) exactly the same
 - it handles bare package names in a way that is at least inspired by NPM packages, although the details differ.

This RFC is not about changing how Module Resolver works. (Aligning it more closely with the broader node_modules semantics is the subject of other ongoing work, and can be it's own RFC.) For this RFC, we take the existing semantics as a given.

The second resolution system in Ember apps is the one that locates components, helpers, routes, controllers, services, and other Ember-specifier object types. I will refer to this as the "Collection Resolver". The collection resolver's limitations are thoroughly described in [Module Unification, RFC 143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md). In short, it has no locality (there's no such thing as relative lookup, so many things end up globally-scoped) and it has no notion of packages (all components, helpers, etc from addon packages go into one big global namespace soup).

This RFC is not about deprecating or dropping the Collection Resolver. Doing so properly would require both a long period of transition, and alternative designs for doing some of the things that the Collection Resolver can do that the Module Resolver should not. For example, the Collection Resolver implements dependency injection, which is _not_ the same semantics as module imports. DI and modules serve different purposes in large scale systems.


## Module Resolving in Ember Templates

Specific proposed rules:

1. We add the ability to use Ember's existing `import` semantics (the Module Resolver) from templates. This is supported in any template, irrespective of any "classic vs pods vs MU" distinction. This new `import` takes precedence, but anything not explicitly imported is handled the classic way by the Collection Resolver.
2. Components and helpers inside MU code (meaning under `/src`) can _only_ be resolved by the Module Resolver, not the Collection Resolver. (See Compatibility Plan below for how addons can be compatible with both worlds.)
3. Importing components and helpers that are authored in non-MU code is not supported.

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

The frontmatter section (delimited by `---`) uses Javascript syntax, but it is limited to only allow import statements.

(This limitation could be lifted by a followup RFC, which would require clarifying the semantic boundary between names in the JS and names in the hbs. That leaves the door open to a single-file component design that could put component Javascript, helpers, etc directly into the frontmatter.)

## Auto-insertion of imports

You don't need to import built-in constructs that ship with Ember (like `if`) or Ember's built-in helpers (like `action`).

To maintain framework extensibility and prevent these built-ins from being so "magical" and privileged, we offer a clear path for apps to register similar global names:

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

 - When a component has a Javascript module, and you import the component from Javascript, you should get the values back you'd expect for a normal Javascript module. We don't want to rewrite things in an opaque way.
 - Consumers of a component should not need to know whether it is implemented as only a template, a template plus Javascript, or only Javascript.
 - A component that is reexported should continue to work unchanged (in contrast with today, in which component authors needs to take special care to make their templates come along).
 - We want to follow the _existing_ Module Resolver rules and not introduce new ones. Especially new ones that would diverge further from typical node_modules resolution.

Given these requirements, the proposed design is:

1. Imports in templates are resolved the same way they are resolved today in Javascript. Conceptually, we add a custom handler for `.hbs` files that allows them to be resolved the same as `.js` files. `.js` files take precedence over `.hbs` files, such that if both exist for a given ModuleSpecifier, the JS wins.

2. Component Javascript should be authored using default exports (as they already are):

    ```js
    export default class extends Component {
      yourStuffHere(){}
    };
    ```

    There is no magic that alters the exported value. When it's imported (whether in Javacript or HBS), the value you get is the actual class.

3. When both `.js` and `.hbs` exist for a component, the build system unobtrusively associates the template with the default value exported by the Javascript. The template must be co-located with the Javascript, having the same filename except for changing the extension to `.hbs`.

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

4. When only a template exists, it gets resolved following the normal Module Resolver system as described in step 1. Our handler for `.hbs` files causes them to be interpreted as Javasript modules like:

    ```js
    // for illustrative purposes only
    import { registerTemplate, useDefaultComponentManager } from '@ember/hypothetical-private-api';
    export default registerTemplate(useDefaultComponentManager, compiledTemplateAndMetadataHere);
    ```

    The point of this is that you can import and invoke the template-only component in the same way you would a template-and-javascript component. In both cases, from glimmer's perspective the actual value that you import is used to lookup the component's metadata & template in a shared, behind-the-scenes WeakMap.

5. Helpers are resolved just like component Javascript. They are simpler because there is no template association to worry about.

6. Anything that was not resolved by this new system continues to be resolved the classic way.

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

## App structure implications

Most of the MU structure remains unchanged. Specifically, you still need things like `src/data/models`, `src/init`, `/src/services`, and `/src/ui/routes/` because all these things are still handled by the Collection Resolver.

But _within_ component collections (meaning `src/ui/components` plus any `-components` directories nested under routes) we don't need to impose any particular structure. You can nest components where they're used, and access them via relative imports.

Within component collections, there is no longer any need for special `-components` or `-utils` private collections. And the names `component.js` and `template.hbs` lose their special meaning. There are no special rules for looking up components that differ from looking up any other modules.

## Addon structure implications

An addon authored in MU is relatively free to organize its components and helpers as it wishes within the `/src/ui/components` directory. Addon authors should provide a top-level entrypoint that re-exports public components & helpers so they're accessible without deep specifiers, and so it's clear what consitutes the addon's public API.

The prior MU RFCs were vague on top-level entrypoints, but they were consistent in insisting that import specifiers should equal on-disk paths. In concordance with that, we clarify that the top-level entrypoint of an MU addon (one that contains a `/src` dir) is its true `index.js` at the addon's root. The ember-cli-specific code that often appears there instead can be placed elsewhere thanks to the `ember-addon.main` option in package.json. (This is a preexisting feature, not a new feature in this RFC.)

For example, an addon containing one component might have `src/ui/components/the-component.js` and `src/ui/componetns/the-component.hbs`. These would technically be resolvable via `import TheComponent from "the-addon/src/ui/components/the-component"`, but that is an ugly deep-import and it's better for addon authors to provide clear reexports at the addon root:

```
// the addon's `index.js` file
export { default } from "./src/ui/components/the-component";
```

So that consumers can say `import TheComponent from "the-addon";`

The build-time hooks needed by ember-cli that are today placed by default in `index.js` can move to `build.js`, and that already works today if you set `ember-addon.main` to point at `build.js`.


## Compatibility Plan

Apps can unilterally adopt MU because they're still allowed to fall back to classic resolving for addons that are not authored in MU.

We can provide a shim utility to addon authors that allows them to reexport their MU components & helpers as a classic app tree, when the consuming app is not MU-aware. This lets addons upgrade unilterally, while clearly segregating the compatibility code within the shim utility, where it can be easily turned off in apps that don't need it.

# How We Teach This

This is intended to simplify the teaching story because

 - users already need to learn how `import` works
 - we are being careful to make sure it means exactly the same thing in templates


# Drawbacks

Explicit imports require more typing. A large template that uses a large number of components and helpers could end up with a big stack of imports. This is not a unique problem to our system, however, and it's already something developers need to learn to manage wisely in Javascript.

This proposal does not require any breaking changes to Handlebars syntax (a Handlebars parser that doesn't know anything about it will treat the front matter as "just content", which is harmless). But if we want nice syntax highlighting and completion inside the frontmatter, we may need to update tools to be aware of it. On the plus side, once this work is done it leaves us in excellent position to expand the frontmatter to full single-file components.

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

It would probably be good to synthesize all the remaining-valid parts of the two earlier RFCs into this document, so there is one authoritative resource.

This design leaves routes and their templates unchanged from the prior MU RFC. That seems to be an uncanny gap. They're still named `template.hbs`, which is maybe OK, but then if you want to nest more components under them you still need to reserve a place under `-components` to avoid ambiguity with child routes, and that means your imports in the route template need to say `import thing from "./-components/thing"` which seems bad.