- Start Date: 2020-01-10
- Relevant Team(s): Ember, Ember CLI, Ember Learn
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking:

# SSR by default

## Summary

This RFC proposes adding FastBoot into the default Application blueprint, turned off by default.

## Motivation

The past decade of web development has swung heavily towards client-side rendering. Although this
has sparked a rennaisance in component-based architecture, and introduced new programming paradigms
into the frontend stack, it has handicapped some fundamental features of the web, such as accessibility,
performance, SEO (and scraping in general), and sharing. Each of these are core to the web, and it
is the responsibility of frontend frameworks to find their way back to supporting them. Although
rendering Javascript apps on the server has its own drawbacks, in order for the technique to mature,
it must first be used widely. Adding a SSR experience into Ember would nudge developers in the right
direction. Further, it would:

- reduce the barrier to entry for new applications to turn on SSR
- clearly communicate that SSR is supported and encouraged
- reduce the pain points of introducing SSR "late in the game"
- make SSR a core concern and competency of Ember
- increase the pace of innovation in SSR and techniques around it (e.g. rehydration).
- help improve the developer experience by encouraging its use

## Detailed design

- Add `ember-cli-fastboot` and `ember-cli-fastboot-testing` into the package.json blueprint and
update the README.md in the blueprint to include a section on FastBoot.
- Update `ember-cli-fastboot` to be *off* by default, and turn on if a query param or the `FASTBOOT_ENABLED` environment variable is set.
variable exists.

## How we teach this

Because this RFC focuses on enabling *access* to SSR, rather than making SSR the default experience,
the teaching material focuses on setting expectations for new applications, vs teaching about SSR
as a technology for frontend frameworks.

First and foremost, including FastBoot in the default application should set expectations that
the developer experience when running FastBoot locally comes with a new set of challenges that
Ember developers are not accustomed to. To be fair, these challenges should not be completely
unfamiliar to those who work in full stack applications today (live reload, server errors, etc).

Secondly, rendering Ember templates on the server comes with some caveats, such as lack of DOM access
and browser APIs. These caveats are addressed partially by features such as Element Modifiers that
no-op in FastBoot, and are documented on ember-fastboot.com, but they would need to be a core part
of teaching Ember as well. The majority of these caveats relate to object initialization, and there
may be a clever way of teaching this in one place.

Third, Ember's current approach to testing would need to account for writing page rendering tests
through the same server that is built into `ember serve`. This means creating generators for
Node-style tests that make requests directly to the server, rather than using the `visit()`
API to go to a URL. There would need to be a blueprint for this and possibly a new set of test helpers.

Lastly, deploying an Ember Application with FastBoot will need to explain that Ember apps built
with FastBoot turned on will not behave differently unless served by a Node server. In order to
make this absolutely clear, the Deployment guide will need to cover this.

## Drawbacks

Adding FastBoot in every application adds complexity to application development
(testing, deployment, etc). These can generally increase Ember's surface area and make it harder
to learn. I believe this can be mitigated by leaving FastBoot and all its various
pieces turned off by default, and making it clear that the built-in experience will take time to
mature.

## Alternatives

Add Server Side rendering as a top level section to the Guides on emberjs.com. This could be an
effective signal, but I don't think it goes far enough to accomplish our goal of improving the web.

## Unresolved questions

- Should `ember-cli-fastboot` be integrated with `fastboot-app-server` before it is integrated into Ember as a default?
- What percentage of existing Ember apps would offer a worse experience to end users if they served through FastBoot (vs static file server)?
