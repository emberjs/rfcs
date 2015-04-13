- Start Date: 2015/04/12
- RFC PR: 
- ember-cli Issue: 

# Summary

Introduce a new Handlebars syntax that would denote beginning and end of a template block. 

# Motivation

In Ember 2.0, we're moving towards components becoming primary mechanisms for encapsulating functionality with templates being the mechanism that wires these components together. It would be helpful to allow developers the ability to customize certain portion of component's template from it's block template without having to extend a component or overwrite it's layout.

Currently, we have the ability to do this but it's limited to `{{else}}` helper.

For example, we can customize what `{{each}}` helper presents when the array passed to it is empty.

```
{{#each model as |item|}}
	{{item.name}}
{{else}}
  	No items are available.
{{/each}}
```

`{{if}}` helper has the same behaviour.

We are currently lacking the ability to specify a custom template block that we can consume inside of the component or a helper. This RFC proposes that we introduce the ability to specify custom named template blocks that can be used to customize the default behaviour of a component or a helper. 

A table component is a good example of a component that could benefit from this API. Let's consider Addepar's [Ember Table Component](https://github.com/Addepar/ember-table). *Ember Table* has 3 distinct areas that a user might want to cusomize while using the component, namely header, body & footer. 

Currently, *Ember Table*'s component layout looks like this

```
{{#if controller.hasHeader}}
  {{view Ember.Table.HeaderTableContainer}}
{{/if}}
{{view Ember.Table.BodyTableContainer}}
{{#if controller.hasFooter}}
  {{view Ember.Table.FooterTableContainer}}
{{/if}}
{{view Ember.Table.ScrollContainer}}
{{view Ember.Table.ColumnSortableIndicator}}
```

To customize any of these areas, the developer has to extend the component, change specify a new layout and make changes inside of this new layout. If we had the ability to specify custom named template blocks then we could allow the developer to customize one of these areas from the template where the component is being used. 

Here is what that might look like,

```
{{#ember-table}}
  {{^header as |name|}}
  	{{name}}
  {{^cell as |value|}}
  	{{value}}
  {{^}}
  Nothing to show.
{{/ember-table}}
```

Specifying a named block inside of the component's block template would override default implementation of that block inside of the component's layout.

# Detailed design

## Expand ^ syntax

Currently, Handlebars implements `^` which means `else`. Infact, `else` is aliased to `^`. We would expand this syntax to allow named portions of template. 

Here is an example of what a template with this syntax might look like.

```
{{#table-component items as |item|}}
	// default block
	{{^header}}
	// header content
	{{^footer}}
	// footer content
	{{^}}
	// empty view
{{/table-component}}
```

`{{^name}}` serve as dividers of the block template.

## Refactor `{{else}}` helper

One possible implementation would be to make `{{^}}` implementation and allow the name of the helper to be specified. 

Handlebars has several built in blocks that are availble on helper's `options` argument,  namely `options.fn` & `options.inverse`. These would be changed to `options.blocks.default` & `options.blocks.inverse` respectively. Every other named template block would be available on `options.blocks` hash. The above example would have `options.blocks.header` & `options.blocks.footer` in addition to it's default blocks.

The component hook will append named blocks onto the template as it does currently with default block.

## Block are available in the layout

The component must be able to determine programmatically if it should consume it's default block or use the passed in named block. If we consider the above example, then `table-component`'s layout might look something like this.

```
{{#if blocks.header}}
	{{yield-to 'header' headerContent}}
{{else}}
	<thead>
		{{#each headerContent as |name|}}
			<th>{{name}}</th>
		{{/each}}
	</thead>
{{/if}}
```

`blocks` keyword becomes a reserved keyword and will throw a warning when the component defines a *blocks* property. We already have a reserved `hasBlock` keyword in the template, so this will not be a far stretch.

## Named blocks have block params

Named blocks will have block params and will receive values with a new helper. `{{yield-to}}` will take name of a block as a first parameter and yield properties as `{{yield}}` does. This will allow the component to expose template friendly values to be used in the named blocks.

```{{yield-to 'header' headerContent}}``` will yield `headerContent` to `header` block.

## Named blocks must be portable

Under certain circomstances it might be necessary to be able to pass a named block into a nested component. This would make it easier to compose nested components.

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?
