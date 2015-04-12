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
  {{^}}
  {{^cell as |value|}}
  	{{value}}
  {{^}}
{{/ember-table}}
```
Specifying a named block inside of the component's block template would override default implementation of that block inside of the component's layout.

# Detailed design

## New Handlebars syntax

To enable this functionality we would need to introduce a new syntax that would deferentiate a  named template block from a regular helper with a block. This would be done with the introduction of `^` syntax that indicates that this is a named template block. 

Here is an example of what a template with this syntax might look like.

```
{{#table-component}}
	{{^header}}
		Header Content goes here
	{{^}}
	{{^footer}}
		Footer goes here
	{{^}}
{{/table-component}}
```

## Refactor `{{else}}` helper

One possible implementation would be to refactor `{{else}}` implementation into a more generic implementation that allows the name of the helper to be modified. 

Handlebars has several built in blocks that are availble on helper's `options` argument,  namely `options.fn` & `options.inverse`. These would be changed to `options.blocks.default` & `options.blocks.inverse` respectively. Every other named template block would be available on `options.blocks` hash. The above example would have `options.blocks.header` & `options.blocks.footer` in addition to it's default blocks.

## Blocks must be available in the template

The component must be able to determine programmatically if it should consume it's default block or use the passed in named block. If we consider the above example, then `table-component`'s layout might look something like this.

```
{{#if template.blocks.header}}
	{{template.blocks.header.yield headerContent}}
{{else}}
	<thead>
		{{#each headerContent as |name|}}
			<th>{{name}}</th>
		{{/each}}
	</thead>
{{/if}}
```

Similar pattern can be followed to specify the ability to customize body and footer.

## Named blocks can yield

Named template blocks need to be able to yield values into the scope in the same way as regular template blocks can. This would allow the component to expose template friendly values to be used in the template block.

## Named blocks must be portable

Under certain circomstances it might be necessary to be able to pass a named block into a nested component. This would make it easier to compose nested components.

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?
`<table-header></table-header>`

# Unresolved questions

What parts of the design are still TBD?
