- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Accessible Routing in Ember

## Summary

When the user navigates to a new route within an Ember application, screen readers do not read out the new content or appropriately move focus. This makes Ember applications not natively accessible for users with assistive technology.

Ember needs to provide a machine-readable way for assistive technology (like screen readers) to know that the route transition has occurred.

## Motivation

In a world where accessible websites are increasingly the norm, it becomes good business to ensure that Ember applications are able to reach basic accessibility standards. The first initiative in this area is to ensure that Ember can provide the same level of accessibility as a static website- and the first task is to tackle basic navigation within a web app.

In a static website, checking `document.activeElement` in the console upon navigation to a new page should return `[object HTMLBodyElement]`. However, in an Ember app this is not the case. For example, if you navigate to one of the pages via the navbar, the focus remains on the element in the navbar. This is a problem for accessibility; the screen reader doesn’t know that a route transition has occurred, and the screen reader user doesn’t know that the link to the new page actually worked (they hear nothing). 

So we have a puzzle to figure out: why don't screen readers read out the new content or appropriately move focus when the user navigates to a new route within an Ember application. In researching the issue more thoroughly, we discovered that NVDA doesn't seem to have a way to handle the history API. (Which makes sense- screen readers were released before the history API was released.)

One of the challenges is that screen reader technology is all closed-source except for [NVDA](https://www.nvaccess.org/), the open source screen reader for Windows (and typically used with Firefox). The source code for NVDA is available on [GitHub](https://github.com/nvaccess/nvda/) and is written in Python. It seems useful, at least initially, to try to solve for SPAs + NVDA, since both can be done in open source. (Related: [NVDA Issue #6606](https://github.com/nvaccess/nvda/issues/6606)) 

## Detailed design

After reviewing possible approaches to this problem, it seems like the very first step should be an implementation that, at the very _least_, returns Ember back to the starting line. We could then build on that by providing a more machine-intelligent iteration for an upscaled experience. This iterative approach would give us a better way to tackle the more difficult edge-case scenarios.

This RFC proposes that we could either implement a two-phase approach (outlined below) OR decide to move directly to the approach outlined in phase two. 

### Phase One - Return to the starting line

This solution proposes the simplest possible solution- implement in Ember what already exists in other traditional websites:
1. When a browser loads a website, the screen reader starts to read the content of that page
2. When the user navigates to a new page in that website, the screen reader starts to read the content of that page
3. (repeat)

Perhaps we could achieve this by adding a function to set focus on the body element when a route transition has occurred. 

In order to not break similar solutions that may already exists in other apps or addons, I propose that the solution be implemented as an optional feature/with a flag. The default could be set to on, and if folks already have a solution in their addons/apps, they could then opt-out of this feature. 

### Phase Two - Intelligent content focus

It would be even better to have a more intelligent solution, since there are several ways new page content could be introduced in an Ember application.

[Ember-self-focused](https://github.com/linkedin/self-focused/tree/master/packages/ember-self-focused) and [ember-a11y](https://github.com/ember-a11y/ember-a11y) are both examples of community-produced addons that attempt to provide a contextual focus, based on new page content. 

From the [Ember-self-focused](https://github.com/linkedin/self-focused/tree/master/packages/ember-self-focused) readme: 
> When UI transitions happen in a SPA (or in any dynamic web application), there is visual feedback; however, for users of screen reading software, there is no spoken feedback by default. Traditionally, screen reading software automatically reads out the content of a web page during page load or page refresh. In single page applications, the UI transitions happen by rewriting the current page, rather than loading entirely new pages from a server; this makes it difficult for screen reading software users to be aware of UI changes (e.g., clicking on the navigation bar to load a new route).

> If the corresponding HTML node of the dynamic content can be focused programmatically, screen reading software will start speaking the textual content of that node. Focusing the corresponding HTML node of the dynamic content can be considered guided focus management. Not only will it facilitate the announcement of the textual content of the focused HTML node, but it will also serve as a guided context switch. Any subsequent “tab” key press will focus the next focusable element within/after this context. 

The tedious, error-prone task of keeping track of HTML nodes that can/could/would/should have focus is something that Ember could do for you. 

A comparable from another JS Ecosystem: [Reach Router](https://reach.tech/router) is available for React apps. 

## How we teach this

For phase one, the guides should include information about skip links, since the application author will need to add those in themselves (just as they would for a static site). For phase two, we would explain that skip links are no longer needed because we have integrated that functionality in a more machine intelligent way. 

### Skip Links
A skip link is an internal link at the beginning of a hypertext document that permits users to skip navigational material and quickly access the document's main content. Skip links are particularly useful for users who access a document with screen readers and users who rely on keyboards.

Directly inside the `body` element, place a skip link like this: 

    <a href="#maincontent">Skip to main content</a> 

This allows the user with assistive technology to skip directly to the main content and is useful for not needing to repeatedly navigate through a navbar, for example.  


It's possible that this solution is a [spherical cow](https://en.wikipedia.org/wiki/Spherical_cow), and the complexity of route transitions in a framework deserves an accessibility solution that is is equally complex. 

It's possible that enterprise Ember users have already implemented their own solution, since governments in many countries require this by law. An out of the box solution would need to be well-advertised and documented to avoid any potential conflicts for users. 

## Alternatives
1. We could try to implement something like Apple’s first responder pattern: https://developer.apple.com/documentation/uikit/uiresponder/1621113-becomefirstresponder 
2. Wait for a SPA solution to be implemented in assistive technology. I would note that this option seems less than ideal, since this specific issue has been reported to the different assistive technologies available, and has been an on-going issue for many years now. 

## Unresolved Questions
- how would we handle loading states?
- how does/could/would this fit in with the router helpers work that Chad did?

## Appendix

### Related RFCs
- [Route Helpers](https://emberjs.github.io/rfcs/0391-router-helpers.html) by Chad Hietala

### Related Reading

- [NVDA Developer Guide](https://www.nvaccess.org/files/nvda/documentation/developerGuide.html)
- [Microsoft Active Accessibility (MSAA)](https://docs.microsoft.com/en-us/windows/desktop/winauto/microsoft-active-accessibility) MSAA is an Application Programming Interface (API) for user interface accessibility. MSAA was introduced as a platform add-on to Microsoft Windows 95 in 1997. MSAA is designed to help Assistive Technology (AT) products interact with standard and custom user interface (UI) elements of an application (or the operating system), as well as to access, identify, and manipulate an application's UI elements. The current and latest specification of MSAA is found in part of [Microsoft UI Automation](https://docs.microsoft.com/en-us/windows/desktop/WinAuto/uiauto-specandcommunitypromise) Community Promise Specification.
- [Structured Negotiation: A Winning Alternative to Lawsuits](https://www.lflegal.com/book/)

