- Start Date: 2014-11-03
- RFC PR:
- Ember Issue: 

# Summary

Allow Handlebars/HTMLbars templates to be embedded within Ember JavaScript code.

# Motivation

To avoid the ugliness of reinventing / reimplementing ES6 imports into the
Handlebars language.

# Detailed design

Handlebars templates compile to JavaScript and perform property lookups
on a context object, or resolve helpers via a container, etc. The proposed
design would allow this compiled JavaScript template to close over values
within the JavaScript file that the Handlebars template is embedded within.
This should seem familiar to how JSX in React allows you to close over
imports/requires/variables, but there are some important differences
with what we are describing. Consider the following:

    import FunComponent from '../components/fun';
    function someHelper(value, options) {
      return value + "!!!";
    }
    export default Ember.Component.extend({
      layout: $HJS(
        <!-- use someHelper from above, use name from context -->
        <p>Hello {{someHelper name}}</p>
        
        <FunComponent>Woot</FunComponent>
        or {{#FunComponent}}{{/FunComponent}}
      )
    });
    
This is an example of embedding Handlebars within the component's
JavaScript file. Beyond just the simple combination of template and JS
within the same file, the most important detail is how variables / helpers /
components are looked up. The rule is this:

- Given `{{foo}}` within an embedded template, if `foo` exists in the local
  JavaScript scope in which the template is defined, `{{foo}}` will lookup
  `foo` within that scope rather than the typical Handlebars behavior of
  looking up `foo` on the template context.
- Any other `{{value}}` that is not declared within that local scope will 
  fall back to today's Handlebars lookup logic and perform the lookup on the
  template context.
  
The result of this is that:

- You can import components/helpers/constants with ES6 imports (rather than
  reimplementing with `import` Handlebars helper or whatever)
- We have an exit strategy from today's everything-is-global-in-a-template
- A migration strategy is preserved by the fact that all non-local lookups
  happen on the template context just like classic Handlebars.
  
Note that by "local scope" we generally mean everything declared within
the JavaScript file; if you wanted to expose a property stashed on `window`
to the embedded template, you would need to do `var foo = window.foo;`.

## Implementation

To opt-in to this behaviour you will use a new extension `.hjs` which will signal 
ember-cli to use the HJS transpiler. The job of the transpiler is to find the
embedded templates in the hjs file and compile them with HTMLBars. We also
statically analyze the JavaScript scope that is accessible to the template and
pass a list of all the identifiers to the HTMLBars compiler.

    var foo = "Hello";
    export default Ember.Component.extend({
      bar: "world";
      layout: $HJS(
        {{foo}}, {{bar}}!
      )
    });

The template compiler will be invoked as

    HTMLbars.compile("{{foo}}, {{bar}}!", { identifiers: ["foo"] });

and will generate hydration code like

    hooks.content(morph0, foo, context, [], {escaped:true}, env);
    hooks.content(morph0, "bar", context, [], {escaped:true}, env);

The important thing to note is that foo is passed as a reference instead of as a
string.

# Drawbacks

Currently {{x-foo}} will use the resolver to separately resolve the x-foo
component's class and template using the string "x-foo". This would no longer
be possible when importing the component class directly. Instead, every
subcomponent we need to embed its template. This means migrating 
templates/components to this new format will have to start at the leaves.

# Alternatives

We have considered introducing a handlebars 
`{{import [FunComponent] from "../components/fun"}}` which would reimplement
parts of the ES6 module import syntax to allow referencing components 
in a template without having them be global accessible in _every_ template.

## Prior Art

- React components use a similar approach. "It's just JavaScript".

# Unresolved questions

- Will Tomhuda concede?

