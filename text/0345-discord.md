---
Start Date: 07/11/2018
RFC PR: https://github.com/emberjs/rfcs/pull/345
Ember Issue: (leave this empty)

---

# RFC to move the Ember community chat to Discord

## Summary

Encourage the Ember community to adopt Discord for real-time chat (vs Slack or other options).

## Motivation

Real-time chat is essential to the function of online communities, particularly in open source. Chat fosters an informal, interactive style of communication that is important for building relationships, sharing community norms, coordinating on projects, and brainstorming ideas.

The Ember community predominantly uses [a Slack instance](https://ember-community-slackin.herokuapp.com/) as the gathering place of choice. While we have benefited enormously from Slack, there are significant downsides as well.

### Loss of History

Because we use Slack's free plan, the entire instance is limited to 10,000 messages in history at any time. Because of this hard cap, the amount of time messages persist continues to shrink as the community grows.

It's hard to quantify exactly how painful this limitation is, as it means that new community members can't search for the answer to a question that was likely answered in the past. We can never go back to reference how or when a decision was made, which can mean decision-making feels less transparent that it should be.

This limit applies not just to chat messages, but direct messages between community members as well. This leads to annoyance, as people have to ask for the same information over again if they forgot to save it, or data loss, as useful things like code snippets vanish into the ether.

### Performance

The architecture of the Slack native application relies on running a separate web application per Slack instance the user is signed into. For users who need to be in multiple Slack instances, this can add up to a significant tax on computer resources, particularly as the application starts up.

Once Slack is up and running, most people generally find the performance reasonable, but a solution that offers better startup and runtime performance would be ideal.

### Privacy Concerns

Slack is very clear that their target audience is companies, who often have strict compliance rules that they must follow. Unfortunately, those needs are often at odds with concerns about privacy in an open source community.

In particular, Slack recently added a feature called Corporate Export that theoretically allows administrators to export all messages, including private messages, without notifying users.

Now, the odds of this feature being abused are extremely low. It is only available on Slack's Plus plan, which means a malicious actor would need to be granted administrator priveleges, pony up at least $150,000 to upgrade our Slack plan for that month, apply for the the Corporate Export feature, have it granted by Slack, and then perform an export without anyone noticing.

Because the difficulty of exploiting this feature for evil is so remote, it's not a primary concern driving this change. But all things being equal, we prefer a solution that doesn't offer export of private messages at all, so it's never a concern at the back of someone's mind.

### Better Communication & Transparency

Out of frustration with Slack's disappearing messages, the Ember.js core team set up a Discord server to evaluate if it might be a better fit for open source communities.

While this was a public Discord server that anyone could sign up for, its existence was not widely publicized because we were unsure if Discord was the right solution.

Over time, the core team and many contributors gravitated towards the Discord, finding that it served our needs better. Because of how valuable the Slack instance is, no one wanted to propose a move to Discord until a plan (like, say, this RFC) could be put in place.

Unfortunately, this state of affairs has had several undesirable outcomes.

First, it has caused many of the most prolific contributors to be less active in Slack. This may give the appearance of stagnation or disinterest, when momentum on Ember has never been higher. It robs lurkers of the ability to become contributors if a good opportunity to help pops up. And it prevents some of the most experienced members of the community from being around to help answer questions they might have an off-hand answer to.

Second, and perhaps worst of all, it undermines the transparency and open governance that we have worked hard to create. Our bar is higher than just making it possible to contribute—we go out of our way to actively welcome and encourage everyone to participate, learn and contribute.

Finally, this is not intended to replace [the forum](https://discuss.emberjs.com/), and that should be made clear. The forum is still the preferred place for asyncronous, threaded conversations where in-depth discussion is desired. 

## Detailed design

### Transition Plan

We will need these things to transition the community smoothly:

- a period of time when we use both chat platforms during the transition, put the equivalent Discord channel information in the Slack channel topic
- a clear guide (with illustrations)
- once all of the setup is complete, the Discord server invites can be distributed. 

Note: the current Discord chat will be closed while this RFC is under consideration. If the RFC is accepted, then a detailed implementation plan (mostly role/channel/server setup) & invitation strategy will be carried out.  

#### Initial Setup

Because Discord has fine-grained controls, we will be able to implement categories for chats.

We intend to have the "welcome" channel as the initial channel for everyone who joins the Discord server. This channel will be read-only and will list the rules for the Discord server.

We also intend to have a "setup" channel. This channel will give you a complete guide of how to take advantage of the personalization, privacy and security, and notification controls in Discord. 

**Verification Level**
Initially, we will be implementing the "low" verification level, which means users will need to have a verified email on their Discord account. If this proves to be too easy of a target for spammers, we will implement a higher level of verification (levels include amount of time a user has to be a verified member of the server before they can post).

**Explicit Content Filter**
Since this is a public Discord server, we will be setting an explicit content filter- it will scan messages from all members without a role. Email-verified members will be given a community member role to start, and other roles may be added to users over time. 

**Categories and Channels**
Community members will then have the option of visiting the "setup" channel and learning more about fine-grained controls, such as:

- notifications
- muting a channel
- muting a category

Because our goal is transparency, all of the channels that exist will be visible in the channel list. A lock icon will display if the user does not have the role necessary to join that channel. (_FWIW, the alternative is to not display locked channels at all, which we felt would be less ideal- it is better to know that there are channels where private conversations are necessary and see what they are._)

The following proposed initial category and channel list was chosen based on the current channel needs and evaluation of the channels with the most members on Slack. _Additional channels may be requested in the Admin/community-feedback channel._

**Category/Channel List:**
- (No Category)
  - welcome (community guidelines are posted here) <--readonly & the server invite puts users in this channel first. 
  - setup-profile (how to setup your profile) <--readonly
- Admin
  - community-feedback (questions, comments, concerns, requests)
  - security
  - steering-committee 🔒 (locked to role “steering-committee”)
  - news & announcements
  - ember-jobs
  - bots
- Core Teams
  - ember-js 🔒 (locked to role “core-js”)
  - ember-data 🔒 (locked to role “core-data”)
  - ember-cli 🔒 (locked to role “core-cli”)
  - ember-learning 🔒 (locked to role “core-learning”)
- Working on Ember
  - ember-cli
  - ember-data
  - ember-engines
  - ember-js
  - glimmer-vm
  - triage
  - st-* (as needed)
- Using Ember
  - general-help
  - learning-ember
  - a11y
  - backend
  - internationalization
  - jsonapi
  - mobile
  - ember-js
  - ember-data
  - ember-cli
  - ember-engines
  - fastboot
  - ember-twiddle
  - e-*
- Supporting Ember
  - documentation
  - website
  - marketing-and-advocacy
  - infrastructure
- Event-Chat
  - EmberConf
  - EmberCamps
  - EmberFest
  - Talks
  - Other Conferences
  - Meetup organizers
- Social
  - Water-cooler (random)
  - Local-*
  - Media (livestreams, videos, podcasts)
  - Pets
  - Women in Ember 🔒
  
**Integrations**
Discord's integration game is strong. Discord has a [very detailed API](https://discordapp.com/developers/docs/intro) and many integrations already exist, and with no limitation (compared to free Slack instances, that have limited numbers of integrations).

## How do we teach this?

In addition to having a setup channel available upon login (with illustrated instructions), here are some links where community members can read more: 

- [Discord Loves Open Source](https://discordapp.com/open-source)
- [Discord Community Guidelines](https://discordapp.com/guidelines)
- [How to use Discord](http://www.businessinsider.com/how-to-use-discord-the-messaging-app-for-gamers-2018-5)

## Drawbacks

### Supporting Learning vs Supporting Development

There is some concern that there is already some confusion on Slack about where to get help learning/using Ember, and where to coordinate working on Ember. We need to have a clear delineation so that the folks who are spending their volunteer time to ship Ember features can continue to concentrate and do that.

### Losing Community Members 

There is some concern that we may lose some community members due to this move. This could happen for a variety of reasons- the nature of OSS work means that some are not always active on the chat community, or the user doesn't want a different chat app, etc. We believe that the former is probably more likely than the latter, since many of us are on at least 2-3 chat apps already. 

## Alternatives

The alternative to this would be to temporarily remain on Slack until we are able to evaluate and choose another viable option. However, we believe that staying on Slack is not desirable.

List of Slack alternatives:

- riot.io
- mattermost.org
- rocket.chat
- spectrum.chat


## Unresolved questions & FAQ

- When will there be conversation threads? We have been told that it is in the works, but there is no ETA.
- Disqus, Discord, Discuss? Which is which? For clarity, we will encourage the use of the terms **chat** (Discord), **the forums** (Discuss), and **blog comments** (Disqus)- mostly so no one has to try to remember.



