- Start Date: 2019-02-20
- Relevant Team(s): Ember Data
- RFC PR: [452](https://github.com/emberjs/rfcs/pull/452)
- Tracking: [Issue](https://github.com/emberjs/rfc-tracking/issues/50)

# **Summary**

This RFC sets the plan for Ember Data's medium term development, on the path towards modularizing the library. We expect the following immediate design and implementation work to take between one and two quarters. The main goals are:

- Increase approachability of the library for both the users and contributors
- Increase iteration speed and stability
- Enable shedding of legacy layers
- Increase flexibility and lay the foundation for future improvements

# Background and Motivation

Ember Data needs to continue to evolve with Ember and the wider Javascript ecosystem and provide a modern first class data handling support. Moreover, we want to reach the future in which using ED is a much more modular and composable experience. In order to do this, we need to create a foundation for future development while making sure we are honoring our stability guarantees and helping the ecosystem evolve together with us. The two main motivations for this RFC are:

- Enabling future improvements
- Increasing velocity and stability


## Enabling future improvements

In order to provide Ember users with a future proof data stack, ED has to mold and adapt. A sample of requirements for a modern data library for Ember include:

- Not depending on Ember's object model
- Support for complex persistence strategies
- Flexible query expressions and mutations, allowing optional GraphQL support
- Out of the box high level of performance without compromises

Ember Data should incrementally evolve to support all of the above with full backwards compatibility. However, evolving a complex codebase over time while maintaining compatibility has always been a tricky endeavor abound with tradeoffs. In the 3.x cycle there have been several angles of attack towards exploring the future of ED.

---

- Orbit.js has been driving experimentation with a multi source transformation driven approach
- An experimental spike of Ember Data frontend backed by Orbit.js (https://github.com/igorT/data-orbit)
- Single model class Ember Data Addon with dynamic schemas and no normalization (https://github.com/hjdivad/ember-m3)
- Public interface in Ember Data that allows for above efforts to be stable and exposes the internals for more experimentation (https://github.com/emberjs/rfcs/pull/293)

While some of these efforts are experimental spikes and others have seen production use, we want to bring their ideas and improvements back into ED proper. The first motivation for this RFC is to design our internal architecture so that we can iteratively move the library in that direction. We have already done the first part of this work, refactoring the bulk of in memory data storage into a pluggable RecordData interface and now need to spread that approach to the rest of the library.

However, we will not succeed in delivering this future vision in a timely manner and bringing it's benefits to our existing body of users if we do not improve our velocity and stability.

# Increasing velocity and stability

Ember Data has continuously evolved over seven years and over time acquired several layers of cruft along with a intimately connected internal architecture. The current state of the library is not particularly hospitable to new contributors while at the same time being hard to iterate on in a stable manner. Lots of Ember Data's public apis have been designed with stability and extensibility in mind, but the internal layers of the library have not benefited from the same amount of design work. The unspecified internal layers together with a host of implicit dependencies make it hard to expose parts of Ember Data to addon developers as well as prevent ED contributors from being able to easily do experimental iterative work. 

The three main problems this internal structure causes us is: 

- It is hard for developers to grok the codebase and become prolific contributors due to the complexity
- Testing ideas in addons is usually a great way to move faster and derisk ideas. However due to the internal structure of ED, such addons use a large surface area of internal apis, negating most of the benefits
- Due to the structure of the code base and the inherent complex nature of data retrieval and storage, it is hard to make changes to ED with confidence. While we have an extensive acceptance level test suite, testing of the internal apis and data structures is lacking at best

We have already begun some efforts to address these problems: incremental conversion to Typescript and the RecordData refactor stand out as biggest value adds.

A great example of how these issues impact us are the problems one runs into when trying to remove ED's dependency on DS.Model and Ember's object model. In a perfect world, if we wanted to ship ED that used ES6 classes instead of DS.Models, we would adopt an incremental approach:

- A small refactor of ED to make base record class pluggable
- Develop an addon that uses ES6 classes as a base record class and experiment with it
- After a period of experimentation, merge the addon into ED core with a compatibility layer for legacy users

However, this is currently very hard to do in the ED codebase, because the whole system is littered with both explicit and implicit assumptions about record's intimate apis and it's class structure. The implicit assumptions throughout the codebase of how a record behaves are especially challenging to reason about and test.

# Goals

Success criteria for this plan is shipping an Ember data release which:

- Does not break SemVer and has no user facing changes
- Consists of several independently tested packages with clearly defined boundaries and interfaces
- (mostly) In Typescript
- Enables a path for users to not have model classes dependent on the old Ember object model and DS.Model
- And allows addons like ModelFragments and ember-m3 to not use any private apis

# Non Goals

While the internal apis and interfaces will be designed with an eye towards the future public and addon use, in this phase of the work we will not be exposing all of them to apps and addons as fully stable public apis, or we will do so for addon author with an explicit plan to deprecate and replace.

Moreover, as part of the RFC we will not be deciding on, or baking in the future app level public apis. 

# High Level plan

The high level plan is to refactor the ED internals so that:

- DS.Model and code relying on it is isolated from the rest of the system allowing us to swap it out in the future
- InternalModel is removed in favor of moving the functionality to specced out APIs
- We introduce a concept of RecordIdentity, a string or POJO which can be used to uniquely identify a record throughout the system
- Boundaries between parts of the system are hardened, specced out and implemented in typescript

In a typical Ember app, Ember Data sits as a layer between the application code which issues queries for data, reads and manipulates the results and the browser fetch or websocket interface.  You could model a typical use of ED at a high level as :

App Query —>   In memory cache —>  Request sent → Data normalized → Result object returned

However Ember Data's architecture currently is quite a bit more complex:

![](https://i.imgur.com/98oiXCx.png)

While the overall shape of the architecture looks as you might expect, our current implementation suffers from several critical problems:

- A large part of critical model behavior is handled by the internal model abstraction
- Entire library implicitly relies on DS.Model classes, Ember's object model
- The division of responsibilities between internalModel, RecordData, Store are not clear

This medium size ball of mud should be refactored to the following high level pieces: (add diagram here): 

![](https://i.imgur.com/C3tNcH2.png)

*this is a high level design, each of these would correspond to an RFC*

- User facing record objects, and a corresponding interface which the rest of the system uses to interact with records. This currently corresponds to DS.Model, DS.RecordArray, ObjectProxy and ArrayProxy. These classes will be swappable, and give us a path to change ED's object model
- RecordIdentity - a system wide way to uniquely identify records
- In memory store, RecordData++, taking what RecordData does today(in memory storage of record and relationship information) and adding record state management, error handling and capability to have a singleton Record Data instead of having to have a Record Data instance per Record
- Schema Service - a service for providing attribute and relationship info to the rest of the system, the initial implementation would just provide a wrapper for DS.Model, but allows us a path forward to dropping DS.Models
- Identity Map, which is keyed of record identity objects instead of holding references to internal models
- User facing Store - class that most resembles todays Store which exposes finder methods to the user
- Query builder - a service which takes user queries such as `findRecord` and turns them into Orbit.js like query objects that can be passed around
- Fetching service - layer that takes user queries, relationship queries and save requests and decides whether they need fulfillment, and if so delegates to adapter/serialzier. Currently done by a mix of store and internal model.

Adapter, Serializer and Snapshots would not have major changes at this time, other than those needed to make use of the schema service and public apis for accessing records. 

It is important to note that none of the existing public user apis would change, and that the bulk of this work is taking existing code structures and making the specified and isolated.

# Tactical plan

- Continue the Typescript conversion
- Implement packages RFC
- Cleanup Record Data

Write RFCs for:

- RecordIdentity
- Custom Record Classes
- Record Data incremental improvements for:
    - Internal Error storage
    - Record State RFC
- Schema Service
- Query builder

# Open questions

What level of future support should we commit ourselves to with the new apis?

# Downsides

- Exposing more intimate apis
