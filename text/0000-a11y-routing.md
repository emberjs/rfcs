- Start Date: (2019-01-18)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Accessible Routing in Ember - Route Navigation Announcement

## Summary

When the user navigates to a new route within an Ember application, screen readers do not read out the new content or appropriately move focus. This makes Ember applications not natively accessible for users with assistive technology.

Ember needs to provide a machine-readable way for assistive technology (like screen readers) to know that the route transition has occurred.

## Motivation

In a world where accessible websites are increasingly the norm, it becomes good business to ensure that Ember applications are able to reach basic accessibility standards. The first initiative in this area is to ensure that Ember can provide the same level of accessibility as a static website- and the first task is to tackle basic navigation within a web app.

In a static website, checking `document.activeElement` in the console upon navigation to a new page should return `[object HTMLBodyElement]`. However, in an Ember app this is not the case. For example, if you navigate to one of the pages via the navbar, the focus remains on the element in the navbar. Or, if you navigate via link in the HTML `<footer>`, the focus stays there- at the bottom of the page in the footer. This is a problem for accessibility; the screen reader doesn’t know that a route transition has occurred, and the screen reader user doesn’t know that the link to the new page actually worked (they hear nothing). 

So we have a puzzle to figure out: why don't screen readers read out the new content or appropriately move focus when the user navigates to a new route within an Ember application. In researching the issue more thoroughly, we discovered that NVDA doesn't seem to have a way to handle the history API. (Which makes sense- screen readers were released before the history API was released.)

One of the challenges is that screen reader technology is all closed-source except for [NVDA](https://www.nvaccess.org/), the open source screen reader for Windows (and typically used with Firefox). The source code for NVDA is available on [GitHub](https://github.com/nvaccess/nvda/) and is written in Python. It seems useful, at least initially, to try to solve for SPAs with Firefox + NVDA, since both can be done in open source. (Related: [NVDA Issue #6606](https://github.com/nvaccess/nvda/issues/6606)) 

## Detailed design

This RFC proposes that we take the first step toward accessible routing by adding an announcement message when the URL has changed. While this will not address the second part of accessible routing, focus, it will move the framework closer to the goal. 

We will implement a live region in the page inside of the body element. The message will be updated when the user interacts with the page in such a way that a new URL is needed (i.e., navigating to a new page, or interacting with an element such as a button that submits a form and moves the user to a confirmation page or similar). We will listen for `routeDidChange` and present the navigation event message to the screen-reader user at that time. This message will inform the screen-reader user that they have navigated to a new URL. There will be a default message but developers will also be able to create a custom message (those with internationalized apps will likely prefer to do so). Users without a screen reader should not have any change in experience. 

This first step of adding an announcement region would allow us to keep the performance gains from `pushState` while informing users with assistive technology that a page transition has occurred.

Some important details: 
- This would be added to the default Ember blueprint and included with all new Ember apps. 
- The developer would need to opt out of this entirely to turn it off (but it will not be recommended to do so).
- This will not address focus being reset in the way that users with assistive technology expect. This still needs to be addressed and we will continue to work toward a reasonable solution for this in the future. 

## How we teach this

It's possible that enterprise Ember users have already implemented their own solution, since governments in many countries require this by law. An out of the box solution would need to be well-advertised and documented to avoid any potential conflicts for users.

## Alternatives
1. We could provide an internationalized default for the top five languages that our data suggests Ember apps are built to support. 
2. We could try to implement something like Apple’s first responder pattern: https://developer.apple.com/documentation/uikit/uiresponder/1621113-becomefirstresponder 
3. We could wait for a SPA solution to be implemented in assistive technology (this option seems less than ideal, since this specific issue has been reported to the different assistive technologies available, and has been an on-going issue for as many years as JS frameworks have existed). 

## Unresolved Questions
- how would we handle loading states?
- how do we make sure this happens _before_ any other things happen (as in things that developers have added to their apps)?

## Appendix


### Related Reading
- Accessibility APIs
  - Windows: [Microsoft Active Accessibility](https://docs.microsoft.com/en-us/windows/desktop/WinAuto/microsoft-active-accessibility) (MSAA), extended with another API called [IAccessible2](https://wiki.linuxfoundation.org/accessibility/iaccessible2/start) (IA2)
  - Windows: [UI Automation](https://docs.microsoft.com/en-us/windows/desktop/WinAuto/entry-uiauto-win32) (UIA), the Microsoft successor to MSAA. A browser on Windows can choose to support MSAA with IA2, UIA, or both.
  - MacOS: [NSAccessibility](https://developer.apple.com/documentation/appkit/nsaccessibility) (AXAPI)
  - Linux/Gnome: [Accessibility Toolkit](https://developer.gnome.org/atk/stable/) (ATK) and [Assistive Technology Service Provider Interface](https://developer.gnome.org/libatspi/stable/) (AT-SPI). This case is a little different in that there are actually two separate APIs: one through which browsers and other applications pass information along to (ATK) and one that ATs then call from (AT-SPI).
- [NVDA Developer Guide](https://www.nvaccess.org/files/nvda/documentation/developerGuide.html)
- [Structured Negotiation: A Winning Alternative to Lawsuits](https://www.lflegal.com/book/)
