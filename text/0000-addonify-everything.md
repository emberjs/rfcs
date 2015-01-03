- Start Date: 2015-01-03
- RFC PR: 
- ember-cli Issue: 

# Summary

Extract nearly all behavior out of ember-cli core into addons.

# Motivation

I believe that if we can reduce the surface layer API enough for ember-cli it can potentially be a common-purpose build tool for other 
JavaScript framework. The ember-cli proejct could be a collection of addons that add build-specific tooling for Ember projects.
The absolute core library would be one that could allow addons to add functionality to the core.

If we can get other frameworks to buy into using certain shared libraries of ember-cli then Ember itself
benefits.

Another benefit would be opening up alternative implemtation to existing core functionality. While much of 
what ember-cli does is very good, one could imagine a talented developer not involved with the project creating 
a competing addon that has significantly better functionality. ember-cli could reliably adopt this upstream if/when it wanted.

# Detailed design

ember-cli is growing in complexity and in some areas getting hairy. Extracting out the behavior into addons will allow
for more focused development. There could be subject matter expert teams specifically assigned to a given addon. For example
all of the build pipeline could be an addon (between the app and addons). The behavior in the build pipeline
has no context relation to say the blueprint system. 

# Drawbacks

It won't be easy. Many of the addon functionality is pretty inter-woven.

# Alternatives

Addons are currently supplementing some built-in behavior but I there isn't much beyond that.

# Unresolved questions

Unknown
