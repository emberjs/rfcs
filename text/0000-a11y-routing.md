- Start Date: (YYYY-MM-DD, updated on 2019-05-13)
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

This RFC proposes two different solutions for us to consider. The solution that is determined to be the most appropriate for our use will be made into an officially supported addon, and be included in the Ember blueprints, so it will be part of every new application by default. 

### Solution #1 - Navigation Message

> "When you learn how to use a screen reader, the first thing you learn how to do, is stop it talking." - a screen reader user. 

One potential solution is that we don't overwhelm the screen reader user with the contents of the page, but rather inform them that they have successfully transitioned to their desired route. In doing so, we will also reset the focus for them. 

- A component is created that provides a screen-reader only message, informing the user that "Page transition is complete. You may now navigate as you wish." The text of this component could be internationalized. 
- Focus is reset to the top left of the page

The component template: 

```hbs
<div tabindex="-1" class="ember-sr-only" id="nav-message">
  Navigation to {{this.currentURL}} is complete. You may now navigate as you wish.
</div>
```

The component js:

```js
import Component from '@ember/component';
import layout from '../templates/components/navigation-narrator';
import { inject as service } from '@ember/service';
import { schedule } from '@ember/runloop';

export default Component.extend({
  layout,
  tagName: '',
  router: service(),

  init() {
    this._super();

    this.router.on('routeDidChange', () => {
      // we need to put this inside of something async so we can make sure it really happens **after everything else**
      schedule('afterRender', this, function() {
        document.body.querySelector('#ember-a11y-refocus-nav-message').focus();
      });
    })
  }
});
```

The addon style (popular sr-only technique): 

```css
#ember-a11y-refocus-nav-message {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

The addon would attempt to provide a sensible resolution for all involved:
- the performance gains from `pushState` remain in place
- users with assistive technology are informed that a page transition has occurred
- the focus is reset to the message itself, which also resets the focus of the page (as is desired for the screen-reader user)
- `aria-live` can remain available for things that genuinely need it (a page transition should not need it)

Some [experimentation of this approach can be seen online](https://navigator-message-test-app.netlify.com/). In this example, the message _is_ shown for demonstration purposes, but in the final product, the message would not be shown.  

What do users with screen-readers think? "It works better than I thought it might."

### Solution #2 - Content Focus

What about content-level focus? [Ember-self-focused](https://github.com/linkedin/self-focused/tree/master/packages/ember-self-focused) and [ember-a11y](https://github.com/ember-a11y/ember-a11y) are both examples of community-produced addons that attempt to provide a contextual focus, based on new page content. 

After some research, we discovered that we could provide content-level focus by using (currently private) API to place focus around the content of the application. 

For the template, we could take one of two approaches: 

1. wrap the `{{outlet}}` on application.hbs with an element with the tabindex and id attributes, like this:

```hbs
<main tabindex="-1" id="ember-primary-application-outlet">
  {{outlet}}
</main>
```

2. change the application.hbs outlet so it is different from the usual `{{outlet}}`, and have it already include what we need:

```hbs
{{application-outlet}}
```

In this way, if an application author did not want to use the `{{application-outlet}}`, for whatever reason (maybe they are nesting apps, or maybe they already have a focus solution) they could remove the `application-` and be left with a classic outlet. 

Either way, we'd depend on the private API `renderSettled` to allow us to focus on the application outlet after the route has changed (a little hand-wavy since it's just an idea): 

```js
export default Route.extend({
  router: service('router'),
  init() {
    this._super(...arguments);
    this.router.on('routeDidChange', transition => {

      if (transition.to !== null) {
        emberRequire.renderSettled().then(function() {
          document.body.querySelector('#ember-primary-application-outlet').focus();
        });        
      }
    });
  }
});
```


## How we teach this

It's possible that enterprise Ember users have already implemented their own solution, since governments in many countries require this by law. An out of the box solution would need to be well-advertised and documented to avoid any potential conflicts for users. 


## Alternatives
1. We could try to implement something like Apple’s first responder pattern: https://developer.apple.com/documentation/uikit/uiresponder/1621113-becomefirstresponder 
2. Wait for a SPA solution to be implemented in assistive technology. I would note that this option seems less than ideal, since this specific issue has been reported to the different assistive technologies available, and has been an on-going issue for many years now. 

## Unresolved Questions
- how would we handle loading states?

## Appendix


### Related Reading
- Accessibility APIs
  - Windows: [Microsoft Active Accessibility](https://docs.microsoft.com/en-us/windows/desktop/WinAuto/microsoft-active-accessibility) (MSAA), extended with another API called [IAccessible2](https://wiki.linuxfoundation.org/accessibility/iaccessible2/start) (IA2)
  - Windows: [UI Automation](https://docs.microsoft.com/en-us/windows/desktop/WinAuto/entry-uiauto-win32) (UIA), the Microsoft successor to MSAA. A browser on Windows can choose to support MSAA with IA2, UIA, or both.
  - MacOS: [NSAccessibility](https://developer.apple.com/documentation/appkit/nsaccessibility) (AXAPI)
  - Linux/Gnome: [Accessibility Toolkit](https://developer.gnome.org/atk/stable/) (ATK) and [Assistive Technology Service Provider Interface](https://developer.gnome.org/libatspi/stable/) (AT-SPI). This case is a little different in that there are actually two separate APIs: one through which browsers and other applications pass information along to (ATK) and one that ATs then call from (AT-SPI).
- [NVDA Developer Guide](https://www.nvaccess.org/files/nvda/documentation/developerGuide.html)
- [Structured Negotiation: A Winning Alternative to Lawsuits](https://www.lflegal.com/book/)

