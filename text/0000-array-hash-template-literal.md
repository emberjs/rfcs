- Start Date: 2020-04-02
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Introduce syntax for array and object literals in templates

## Summary

This RFC proposes to introduce syntax to arrays and object literals in templates.

Using array literal syntax, here is how you would pass an array with elements "ember", "angular", and "vue" to a component:

```handlebars
<DisplayFrameworks @frameworks={{['ember' 'angular' 'vue']}}/>
```

Using object literal syntax, here is how you would pass an object with key-value pairs to a component:

```handlebars
<DisplayFrameworkTaglines @frameworks={{
    ember='A framework for ambitious web developers.'
    angular="One framework. Mobile & desktop."
    vue="The Progressive JavaScript Framework"}}
/>
```

## Motivation

The framework previously introduced the [`hash` helper](https://api.emberjs.com/ember/3.17/classes/Ember.Templates.helpers/methods/hash?anchor=hash) in the v2.3.0 release.

This proved so useful that there was demand for an array helper to go along with it. After a [a array helper proposal](https://emberjs.github.io/rfcs/0318-array-helper.html) went through the RFC process, the [`array` helper](https://api.emberjs.com/ember/3.17/classes/Ember.Templates.helpers/methods/array?anchor=array) was made available in the v3.8.0 release.

The widespread usage of these helpers motivates enhancing the template syntax with literal notations for declaring both arrays and objects.

Developers can already use the helpers in conjunction with others like the [`let` helper](https://api.emberjs.com/ember/3.17/classes/Ember.Templates.helpers/methods/let?anchor=let) to declaratively define data structures in their templates, but introducing literal syntax makes it easier to tell it apart visually.

For example, imagining we create an array of objects with name and tagline properties describing JavaScript frameworks,
this is what it would look like with the `hash` and `array` helpers:

```handlebars
{{#let (array
    (hash name="ember" tagline="A framework for ambitious web developers.")
    (hash name="angular" tagline="One framework. Mobile & desktop.")
    (hash name="vue" tagline="The Progressive JavaScript Framework")
) as |frameworks|}}
    <DisplayFrameworkTaglines @frameworks={{frameworks}}
/>
{{/let}}
```

And with literal syntax:

```handlebars
{{#let [
    (name="ember" tagline="A framework for ambitious web developers.")
    (name="angular" tagline="One framework. Mobile & desktop.")
    (name="vue" tagline="The Progressive JavaScript Framework")
] as |frameworks|}}
    <DisplayFrameworkTaglines @frameworks={{frameworks}}
/>
{{/let}}
```

## Detailed design

```handlebars
<ul>
    {{#each ["ember" "angular" "vue"] as |item|}}
        <li>{{item}}</li>
    {{/each}}
</ul>
```

### Hash literal

```handlebars
<ul>
    {{#each-in [
        ember="A framework for ambitious
web developers."
        angular="One framework. Mobile & desktop."
        vue="The Progressive JavaScript Framework"
    ] as |framework tagline|}}
        <li>{{framework}}: {{tagline}}</li>
    {{/each}}
</ul>
```


```handlebars
{{["this" "is" "an" "array"]}}
```

```handlebars
{{["this" "is" "an" "array" ["inside" "an" "array"]]}}
```

```handlebars
{{one="this" two="is" three="a" four="pojo"}}
```

```handlebars
{{
    one="this"
    two="is"
    three="a"
    four="pojo" 
    (one="inside" two="a" three="pojo")
}}
```

```handlebars
{{
    name="Katie Gengler"
    teams=["cli" "framework" "steering"]
}}
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

N/A

## Alternatives

N/A

## Unresolved questions

N/A
