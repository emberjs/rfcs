- Start Date: 2015-07-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC adds in named yields into components to match the slots proposal for web components; It allows authors to add named areas to place content into their components.

# Motivation

Components are more than just dumb boxes, creating components that are more than just content is currently difficult.
So in the modal component case allowing the user of the component to be able to specify what is within the title of the modal. 

# Detailed design

templates/components/my-thing.md
```
<div>
  <div class="heading">{{yield thing}}</div>
  <div>{{yield}}</div>
</div>
```

templates/application.md
```
Look at my great thing:

<my-thing>
  Text
  <div slot="2">Some text 2</div>
  
  <h1 slot="thing">Some text 1</h1>
</my-thing>
```

The rendered DOM would look similar to this:
```
<my-thing>
  <div>
    <div class="heading"><h1>Some text 1</h1></div>
    <div>Text</div>
  </div>
</my-thing>
```

## Distribution

Should loosely match the definition here: https://github.com/w3c/webcomponents/blob/gh-pages/proposals/Slots-Proposal.md

- Is the node text or missing a slot attribute
  - Send to **default yield**
- If the slot attribute matches a named yield
  - Send node to **specified yield**
- Else
  - Discard node


# Drawbacks

- Slots is still a proposal

# Alternatives

The alternatives are letting all component users redefine structure in their templates repeatedly as currently is happening.

# Unresolved questions

The JavaScript distribution API is stalling slots however I feel this isn't required for Ember.

