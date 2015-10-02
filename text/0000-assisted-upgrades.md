- Start Date: 2015-10-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Manually updating Ember apps to remove depreciated syntax is time consuming and
error-prone. We should automate this process where possible and make it a first
class citizen in Ember.

# Motivation

Simplifying the Ember upgrade process is good for everyone. Application developers spend less
time upgrading their apps and have a reduced likelihood of introducing bugs. New Ember features 
can be more ambitious with syntax changes and possibly have shorter depreciation timeframes 
allowing for a leaner framework codebase.

# Detailed design

A large number of the [v1.x depreciations](http://emberjs.com/deprecations/v1.x/) could be
automated. Once simple example is the `bind-attr` deprecation:

`<div {{bind-attr title=post.popup}}></div>`

is now

`<div title="{{post.popup}}"></div>`

Depreciation detection and assisted upgrade capabilities could be added to ember-cli. The
process for introducing new syntax and APIs which depreciate old ones should include
consideration for automating the transformation.

# Drawbacks

Introducing a deprecation would take a little more effort.

# Alternatives

The [ember-watson](https://github.com/abuiles/ember-watson) addon provides some commands for
automatically migrating JavaScript syntax changes but not Handlebars ones.

# Unresolved questions

Can we leverage libraries such as [htmlbars](https://github.com/tildeio/htmlbars) and
[babel](https://github.com/babel/babel) to aid the transformation of syntax?
