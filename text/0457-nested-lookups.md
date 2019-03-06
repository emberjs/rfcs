- Start Date: 2019-03-05
- Relevant Team(s): Ember.js
- RFC PR: [#457](https://github.com/emberjs/rfcs/pull/457)
- Tracking: (leave this empty)

# Nested Invocations in Angle Bracket Syntax

## Summary

Create a syntax for invoking components nested inside of directories in angle-bracket syntax. The invocation `<AppIcons::Warning>` refers to the component `app/components/app-icons/warning.js` when using the default resolver, and looks up `component:app-icons/warning` in the container.

## Motivation

> Note: This RFC uses curly syntax without blocks for illustrative purposes. In all cases, adding a block doesn't change the normalization or lookup rules described here.

When using curly syntax, component invocation looks like this:

```hbs
{{some-component some=parameters}}
```

If `some-component` is not a local variable in scope, the semantics of this invocation are:

1. look up `component:some-component` in the container
2. look up `template:components/some-component` in the container
3. invoke the component with its template (using the relevant component manager semantics)

When using angle-bracket syntax, the equivalent invocation looks like this:

```hbs
<SomeComponent @some={{parameters}}>
```

The syntax means the same thing, with one additional step:

1. **normalize `SomeComponent` into `some-component` using the dasherization rule specified in [RFC #311][angle-bracket-dasherize]**
2. look up `component:some-component` in the container
3. look up `template:components/some-component` in the container
4. invoke the component with its template (using the relevant component manager semantics)

[angle-bracket-dasherize]: https://emberjs.github.io/rfcs/0311-angle-bracket-invocation.html#tag-name

Curly bracket syntax also allows the user to invoke a component nested inside of a directory. This syntax:

```hbs
{{app-icons/warning color="yellow"}}
```

has these semantics:

1. look up `component:app-icons/warning` in the container
2. look up `template:components/app-icons/warning` in the container
3. invoke the component with its template (using the relevant component manager semantics)

However, RFC #311 didn't specify an angle-bracket equivalent syntax for the same semantics.

This RFC proposes that the `::` separator serve the same purpose, with the same semantics, in angle-bracket notation:

```hbs
<AppIcons::Warning @color="yellow" />
```

## Detailed design

This RFC is an extension to the **normalization** rules that already occur in angle bracket notation.

RFC #311 specified the normalization rules as:

> The tag name will be normalized using the dasherize function, which is the same rules used by existing use cases, such as service injections

This RFC amends the normalization rule by first replacing any occurrences of `::` in the tag name with a `/`, but otherwise doesn't change the semantics of RFC #311.

[RFC #143][module-unification], Module Unification, uses `::` as a package separator, but because of problems with scoped packages in npm, we no longer intend to use the `::` syntax in that way. Instead, we intend to allow templates to import components from other packages using import syntax, and the `::` syntax is therefore available for this purpose.

[module-unification]: https://emberjs.github.io/rfcs/0143-module-unification.html

## How we teach this

RFC #311 introduced a normalization rule for angle bracket invocation, so we need to teach that `<NavBar>` invokes a component that appears in the file system as `nav-bar`.

After this RFC, we will also need to teach that directories in the file system should be referenced in the template using `::`.

## Drawbacks and Alternatives

This RFC introduces another normalization rule--Ember developers will need to understand that `::` refers to nested directories in the file system. It also introduces another divergence from the `{{` syntax, which already uses `/` for the same purpose.

Alternatively, we could use `/` as the separator for the same purpose. This would have the benefit of matching the idiomatic way that people describe file system nesting as well as matching the existing `{{` syntax.

On the other hand, it has poor syntax highlighting in virtually all existing highlighters:

```hbs
<AppIcons::Warning></AppIcons::Warning>
<AppIcons/Warning></AppIcons/Warning>
```

Additionally, some autocomplete systems assume that `<AppIcons/` is the beginning of a self-closing tag.

Another drawback of this proposal is that it uses `::` syntax for a temporary concern. It's possible that we would want to use this syntax, which might be considered valuable, in another context in the future, even after we implement template imports. That said, there is no specific proposal for what we might want to use this syntax for, and we could compatibly reclaim it in the context of template imports, at the cost of some mental churn.

Another alternative is to avoid introducing a solution to this problem, and wait for the expected longer-term solution, template imports. While this is indeed expected to serve our longer-term goals, it would mean that Ember users couldn't use angle-bracket invocation, with all the benefits they bring, with nested components, or choose not to nest components at all.

On balance, it seems better to introduce an interim syntax that restores feature parity with curly invocation.

## Unresolved questions

Which of the alternatives should we choose?
