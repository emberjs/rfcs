- Start Date: 04/03/2015
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Route Driven Pod Structure

Have pods become the default file structure that reflect the hierarchy of the application.

# Motivation

Pods are a great way of splitting applications apart into functional sets, however in their current form they have several problems.  The largest of those problems is findability of files.

### Findability

Files in pods are named after their type and not the name of the type e.g. `my-component/component.js` vs. `components/my-component.js`. Having tens of files called `component.js` is extremely difficult to search for and is also cumbersome when working with multiple `component.js` files. For instance below is an example of having 4  different components open at the same time.

![](http://i.imgur.com/wgwKUJa.png)

A developer has to preform so much context switching just to ensure they are working in the correct file.

Searching for files has the same problem of scanability of the file names.

![](http://i.imgur.com/m2wv5Fs.png)

### Alignment With Routable Components

Looking at the existing pod structure it does not align well the mental model of routable components.

# Detailed design

To address the existing shortcomings of pods, a new default structure needs to be introduced. For the most part, this new structure is more semantic and is in the same of vein as "if your url is nested, your ui is nested".  See the below sample structure.


```
app
├── routes
│   ├── profile
│   │   ├── member-photo
│   │   │   ├── member-photo.hbs
│   │   │   ├── member-photo.js
│   │   │   └── member-photo.scss
│   │   ├── member-connections
│   │   │   ├── member-connections.hbs
│   │   │   ├── member-connections.js
│   │   │   └── member-connections.scss
│   │   ├── edit
│   │   │   ├── edit-name
│   │   │   │   ├── edit-name.hbs
│   │   │   │   ├── edit-name.js
│   │   │   │   └── edit-name.scss
│   │   │   └── profile-edit-route.js
│   │   └── profile-route.js
│   └── admin
│       ├── member-permissions
│       │   ├── member-permissions.hbs
│       │   ├── member-permissions.js
│       │   └── member-permissions.scss
│       └── admin-route.js
├── helpers
│   └── format-date.js
├── initializers
│   └── tracking.js
├── services
│   └── tracking.js
├── utils
│   └── fib.js
├── shared
│   ├── dropdown-menu
│   │   ├── dropdown-menu.js
|   |   ├── dropdown-menu.scss
│   └── └── dropdown-menu.hbs
├── app.js
└── router.js

```

The largest change here is that `routes` is where majority of your application lives.  In Ember applications, everything is organized and flows from the routes. Because of this it's fitting that routes map back to the project structure.  When your project is organized around the routes several things fall out of this.

- A happy path to coarse bundling of the application
   - This can be thought as an intermediate step towards engines.
- Automatic namespacing of components
- Discrete workspaces for larger teams

## Resolution

The resolution of components and nested-routes is a tiered/fallback approach much like the current pattern in JJ Abrams Resolver.  Routes take precedence over over components.


## Coarse Bundling

With a route centric structure, projects can very easily be bundled into coarse bundles. For instance the sample structure described above can very easily be built into the following files.

```
dist
├── routes
│   ├── profile
│   │   ├── profile.js
│   │   └── profile.css
│   └── admin
│       ├── admin.css
│       └── admin.js
└── app.js

```

In this case `app.js` is everything that is top level sans the `routes` directory.  The reason behind this is that these constructs are application agnostic and no nothing about the overall structure of the application.

In an engines world the directories in the `routes` folder can be looked as mountable entry points to the hosting application.

Since you can have nested components sitting the same level, routes once again take precedence over components.


## Automatic Namespacing

As of Ember 1.10.0, components can be nested we have the ability to namespace modules based on the route in which the component is contained in. For example, using the `member-photo` component would look like the following.

```
<profile@member-photo src={{member.imgUrl}} position={{member.position}}"></profile@member-photo>
``` 

For nested routes you would simply extend the namespace to incorporate the nested route.

```
<profile.edit@edit-name name={{member.name}}></profile.edit@edit-name>
```

The convention for components that need to span across more than 1 route is to place them within the `shared` directory.  The shared directory is bundled into `app.js` and is not namespaced as it can appear anywhere in the application.

```
<dropdown-menu></dropdown-menu>
```

## Custom Resolver

To enable this type of structure we will need to create a custom resolver and introduce the concept of `@` symbol based namespaces.


# Drawbacks
This will be introducing another directory structure and would have to come up with a migration plan if this becomes the default structure.

Because this structure follows the "if your url is nested, your ui is nested" convention, it means that you potentially have a folder pyramid of hell.

We need to have generators be aware of this new structures.

# Alternatives

Robert Jackson has a [proposal](https://github.com/rwjblue/container-resolver-namespaces) that talks about namespaces, so he may have other ideas.

# Unresolved questions

- `@` symbol namespace resolution
- Where do fragments fit in?
