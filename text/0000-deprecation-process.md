- Start Date: Aug 12, 2015
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

## Improved Process for Communicating About and Discussing Future Deprecation Warnings ##

# Summary

Deprecations shall go through an RFC process and will be documented before being enabled.

# Motivation

Deprecating features provide greater short term benefit to developers of the code base than to its users, and more costs to those users than to the developers.

Up until now, community members feel they have no say in whether or when features are deprecated. They also feel that they do not have adequate prior notice. While improvements are being made to help deal with deprecations later, it is necessary to communicate why a feature is being deprecated and to ensure documentation is provided before a deprecation is released.

In truth, most deprecations done so far are for very good reasons. The only issue is that those reasons have not been communicated and the community does not know if the cost of those deprecations to their productivity has been taken into account.

# Detailed design

A deprecation will be initially proposed through an RFC here instead of a pull request. Only once the deprecation RFC is accepted will a pull request be accepted. A pull request including a new deprecation warning will reference the RFC.

In the RFC, a brief justification of the deprecation warning will be presented. Also, the proposed text of the deprecation guide and the text accompanying the deprecation warning will be included.

A pull request introducing the deprecation will be merged after the community has had an opportunity to comment on the suggested deprecation and a pull request to `website` repository has been created with a deprecation guide.

# Drawbacks

Often, a deprecation is not controversial and this process will appear to be unnecessary, especially to the contributor(s). Also, a contributor often needs to work with others because he or she may not be a very good writer of English.

# Alternatives

Some alternatives include:

1. Requiring a feature flag to allow for some process to occur prior to enabling a deprecation warning. However, given that enabling a warning by itself does not require that much code, a feature flag just for this is probably unnecessary.

2. Asking the community upon creating a deprecation warning to create a blog post about the warning. We think this does not truly make the community feel like they have some input into whether a feature is deprecated.

# Unresolved questions

How should the core team decide whether or not to accept an RFC for a deprecation warning?

In what other ways can the core team get out the message that a feature will be deprecated in a future release?
