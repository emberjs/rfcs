- Start Date: (2017-07-31)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember components currently only support the role attribute by default, with the ariaRole property. Better support for basic ARIA attributes would benefit developers who are required to support assistive technology as a legal requirement. 

# Motivation

The motivation for the RFC comes from my work on a large UI component project in an enterprise environment. Because of the legal requirements for WCAG 2.0 AA conformance in my industry, I am always building in these properties to all of my components. I would prefer that Ember.component supports them by default, as it already does with ariaRole. 

# Detailed design

In the same way that ariaRole is included in Ember.component, it would be useful to see three(3) other basic ARIA- attributes: 
* ariaLabel = aria-label
* ariaLabelledBy = aria-labelledby
* ariaDescribedBy = aria-describedby

# How We Teach This

It could be integrated into the API and presented in the same way that ariaRole is now. It gives a basic definition, links to the WAI-ARIA specification, and generally leaves it up to the developer to determine how to use it in a conformant way. 

## ariaLabel 
```aria-label``` defines a string value that labels the current element. Since only specific elements can successfully use aria-label, it is recommended to review the aria-label specification before using this property. 

See https://www.w3.org/TR/wai-aria/states_and_properties#aria-label and https://www.w3.org/TR/WCAG20-TECHS/ARIA6.html

## ariaLabelledBy
aria-labelledby identifies the element(s) that labels the current element. It provides the user with a (machine) recognizable name of the object. The value should be the ID(s) of the element(s) that provide a label text. 

See https://www.w3.org/TR/wai-aria/states_and_properties#aria-labelledby and https://www.w3.org/TR/WCAG20-TECHS/ARIA13.html 

## ariaDescribedBy: 
aria-describedby identifies the element(or elements) that describe the object. It is generally intended to provide more verbose information. The value should be the ID(s) of the element(s) that provide this descriptive text. 

See https://www.w3.org/TR/wai-aria/states_and_properties#aria-describedby and https://www.w3.org/TR/WCAG20-TECHS/ARIA1.html

While I would like to see some basic accessibility concepts included in the Ember guides, that is beyond the scope of this RFC. 

# Drawbacks

Possible drawbacks:
* Maybe there's some hyphen concern deep inside the Ember internals that would prevent this from happening. I can't say for sure since I have never had to dig that deep, yet. 
* Performance: maybe Ember is already at some sort of perf tipping point and these would put it over the edge. 

# Alternatives

The impact of not doing this is that developers who are trying to make conformant components will keep needing to add these same properties and using attributeBindings to attach them to their components. 


# Unresolved questions

