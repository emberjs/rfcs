- Start Date: 10/22/2019
- Relevant Team(s): Steering, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/549
- Tracking:

# <RFC title>

## Summary

Ember is a web technology, but use with other technologies allows Ember to become more than just great at building web apps. We dont do a great job showcasing that, or even talking about it in our docs / website.

## Motivation

Using one framework to create web apps, mobile apps and desktop apps can be very beneficial to people and companies. 
I believe there is a reason that when you go to any other popular frameworks site i.e https://angular.io or https://reactjs.org there is a reason why the first (or one of the first) things you will see is them talking about using it on many different platforms.
Ive heard people say "We aren't sure if there is enough interest" while developing Glimmer Native. But that is to be expected since we don't advertise ourselves at all for being a cross platform solution. People don't know what we can do if we don't tell them what we have to offer.

## Detailed design

This is more of a mindset shift than anything but I think we need to start caring and giving support to developing for other platforms.
My ideal solution (which I realize probably isn't realistic)
1. Create a new core team (Platform Development)
2. Put them in charge of Mobile and Desktop
3. Add a section onto the homepage of the website showcasing that we offer solutions for things other than the web.

But it seems that the ember teams don't have time to pick up and give first class native or desktop support which is fine. So my backup proposal is

1. Add a section onto the homepage of the website showcasing that we offer solutions for things other than the web.
2. Have that section lead to a page that describes / lists those 3rd party solutions
3. While a stretch, it would be beneficial to have 1 member of the Core team to work with and oversee these projects since we are advertising them.

## Drawbacks

Obviously we don't have unlimted time, so the biggest issue with creating a new core team would be that we would probably have to borrow from other teams which would provide time constraints.
A drawback of keeping the libraries developed by 3rd parties is we are putting our (Embers) name on them by advertising them, so if they don't work right it looks bad on us.

## Alternatives
The alternative would be to continue doing nothing/ say nothing related to development for other platforms
