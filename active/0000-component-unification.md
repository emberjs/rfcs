- Start Date: 2015-05-24
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes a unification of the way that components and their containing element are defined. It also unifies "element" components and "fragment" or "tagless" components.

# Motivation

The motivation of this RFC is to unify:

* a component's template with its outer element
* element-style components with fragment-style components

In particular, it is an attempt to eliminate the gap between templates and APIs like `attributeBindings`, `classNames`, `classNameBindings`, `tagName`, `elementId` and others on the component itself. It is also an attempt to allow fragment-style components ("tagless components") to specify one of its elements to use as the root element for event handling and attribute shadowing.

In brief, an invocation of an angle bracket component would be replaced with the rendered contents of its layout in all cases:

```handlebars
{{!-- application.hbs --}}
<my-button content="hello" />
```

```handlebars
{{!-- my-button.hbs --}}
<button>{{attrs.content}}</button>
```

This invocation would result in (roughly):

```handlebars
<button>hello</button>
```

This also works if you have multiple top-level elements in your component layout:

```handlebars
{{!-- my-button.hbs --}}

<p class="open"></p>
<p>{{attrs.content}}</p>
<p class="close"></p>
```

The same invocation as above would result in:

```handlebars
<p class="open"></p>
<p>hello</p>
<p class="close"></p>
```

At the basic level, this mental model makes it easy to use components for all kinds of different abstractions, including abstractions in special HTML contexts, like `<table>`s, `<select>`s and SVG.

# Detailed design

When a component is invoked using angle brackets, that invocation is replaced by the component's template.

> This is different from today's curly-brace semantics, which replace
> the invocation (`{{my-button}}`) with an element managed by Ember,
> unless the component specifies `tagName: ''`. If a curly-brace
> component is "tagless", it cannot handle events, and has no access
> to its DOM element.

The component's template (sometimes referred to as its "layout"), has access to the attributes passed in by the invocation as `attrs`, and the JavaScript implementation of the component can access the attributes as `this.attrs`.

## The Root Element

If there is only a single element at the top-level of a component's template, that element:

* handles events for the component via declarative event handlers
  (methods and `.on`)
* is available at `component.element` and `component.$`

If there are multiple elements at the root, or if there is a dynamic control flow at the root (`#if` / `else` or `#each`), an element can be marked as the root element:

```handlebars
<hr>
<div {{root}}>
  {{yield}}
</div>
```

The element marked as `root` will then become the target of declarative event handlers and be available via `component.element` and `component.$()`

## Propagating Attributes

If there is a single root element, it will also shadow any string attributes onto itself.

These templates:

```handlebars
{{!-- application.hbs --}}
<my-box aria-role="button" title="jump ahead">Jump</my-box>
```

```handlebars
{{!-- my-box.hbs --}}
<div class="box-outer"><div class="box-inner">{{yield}}</div></div>
```

Would produce:

```html
<div class="box-outer" aria-role="button" title="jump ahead">
  <div class="box-inner">Jump</div>
</div>
```

The benefit of this shadowing is that it allows the invoker of a component to install common attributes on the root element of the content that will replace it.

> In practice, we have observed that components quite often forget to shadow important attributes, and this leads to a barrage of pull requests as people request specific attributes to be added. In practice, allowing the invoker of a component some control over the root element seems like a much better default than asking every component author to remember to shadow attributes.

This shadowing only applies to attributes provided as strings in original invocation. This means that an invoker of a component can avoid shadowing any attribute by writing `name={{"Yehuda"}}` rather than `name="Yehuda"`.

The resulting rule is very simple, and gives total control over shadowing, by default, to the invoker:

* Any attribute *specified as a string* will be shadowed onto the root
  element, if one exists.

### Shadowing Attributes Elsewhere

A component author may wish to specify the exact element to shadow the outer attributes because there is no single root element:

```handlebars
{{!-- application.hbs --}}
<my-box aria-role="button" title="jump ahead">Jump</my-box>
```

```handlebars
{{!-- my-box.hbs --}}
<hr><div class="box" {{splat-attrs}}>{{yield}}</div><hr>
```

Would produce:

```html
<hr>
<div class="box" aria-role="button" title="jump ahead">Jump</div>
<hr>
```

### Disabling Attribute Shadowing

In some cases, a component might want to completely disable attribute shadowing. While it is always possible for a component invocation to avoid shadowing even string attributes (via `attr={{"value"}}`), it may sometimes be useful for a structural component built together with the rest of its app to disable shadowing.

> The reason to make shadowing the default and not vice versa is that **the risks of the wrong default are not symmetrical**. If we chose to make shadowing off-by-default and an addon didn't explicitly enable it, the only solution for apps would be to submit a pull request to the original addon. (This adds additional, potentially fatal friction to attributes that are used for accessibility.) In contrast, by making it on-by-default, an app can avoid shadowing by using a non-string attribute. While annoying, it can be done without changing the code of the component, and the resulting code is reasonably terse and understandable.

To explicitly disable shadowing attributes:

```handlebars
{{!-- my-box.hbs --}}
<div class="box" {{splat-attrs false}}>{{yield}}</div>
```

This should be done with care, especially in addons, as it prevents the invoker of a component from including useful attributes on the root element. **This is especially problematic for accessibility, because it makes it difficult to install ARIA attributes, `alt`, `title` and others that are critical for assistive technologies.**

## Pedagogy

In order for this to work, it is important to teach the component model matter-of-factly. "When you use a component it is replaced with its contents." "If it has a single root element that element is used for event handling and automatically shadows attributes."

We should design educational materials alongside the implementation work of this RFC to see whether the differences between single-element components and multi-element components overwhelm the similarities for the purposes of explanation.

If they do, it may be worth front-loaded the differences and go with the `<element>` alternative, but if the similarities dominate (as this proposal expects), the pedadogy benefit will be strong.

It will be important to attempt to teach the multi-element case as being the same as the single-element case, but without the benefit of the default root (which is a somewhat obvious limitation).

# Drawbacks

The first major drawback is the change from Ember 1.0 curly-brace components, which had an Ember-managed element. However, one major drawback of that system is that it was necessary to manage the attributes, classes, tag name, and ID through additional JavaScript APIs, despite the fact that other APIs existed in the template.

As the templating layer has gotten better (most notably with the removal of `bind-attr`), the gap between the templating layer and the medley of APIs to do the same thing in JavaScript has grown. This proposal re-unifies them.

Second, this proposal attempts to provide good defaults for root element handling and attribute splatting. While the risks of the wrong default are not symmetrical (as described above), it is possible that the default behavior is too aggressive.

Finally, this proposal makes Ember components inconsistent with web components, which manage an outer element. However, the fact that this makes web components unsuitable for abstracting content in special contexts (tables, select boxes, and SVG) and poorly suited for abstracting multiple elements in general is an argument in favor of the "replacement" strategy.

This is the strategy used by React.js, and it works quite well. The mental model of "when you invoke this component, it is replaced with its content" is intuitive, quite easy to teach, and works well with the CSS we have today.

# Alternatives

One major alternative would be to make the attribute splatting opt-in rather than opt-out. It is reasonably easy to teach people how to splat attributes, and adding additional implicit behavior is often frustrating.

The reason this proposal chose to make it opt-out is described in "Disabling Attribute Shadowing": the risks associated with forgetting to opt-in are much higher than the risks associated with forgetting to opt out.

That said, since there is an easy way to opt into splatting arguments (`{{splat-attrs}}`), it might be enough to encourage the authors of reusable attributes to include a splat in their definitions. On the flip side, if everyone has to remember to write it, maybe it's a good default!

Another alternative would be to stick with the original design from Ember 1.x and have components manage their own element. However, the drawbacks of this system are acute, and this solution is almost strictly better than the `attributeBindings`, `classNameBindings`, etc. world of Ember 1.x.

Yet another alternative would be to wrap templates with `<element>`, which would mark the part of the template owned by the component. The `<element>` tag would be able to specify attributes like any other tag, which would eliminate the need for the Ember 1.x parallel API in JS, and make it easy to see the entire structure of the component in the template.

However, it's not clear why this:

```handlebars
<element tagName="content" class="box">{{yield}}</element>
```

Is better than this:

```handlebars
<content class="box">{{yield}}</content>
```

One of the goals of this proposal is to make the basic mental model simple, and "the component invocation is replaced with its template" is simple and easy to understand.

When precisely to handle events, and whether to splat arguments are worth exploring. In the opinion of this proposal, those questions are easily amenable to the usual analysis of defaults: what is the common case, and what are the relative costs of each decision.

With all of that said, opt-out can sometimes be more frustrating to people who did not expect the default behavior, even when they understand the rationale for the tradeoff.

# Unresolved questions

* Should we go with `<element tag="foo">` as an explicit marker of
  the root element and an opt-in for auto-splat?
* Should we make splatting opt-in instead of opt-out?