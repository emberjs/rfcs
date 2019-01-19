- Start Date: 2015-05-16
- RFC PR: [#12](https://github.com/ember-cli/rfcs/pull/12)

# Summary

This has come up in [#3699](https://github.com/ember-cli/ember-cli/issues/3699) and [EmberTown/ember-hearth/#66](https://github.com/EmberTown/ember-hearth/issues/66).

In short, it would be nice for tools that depend on Ember-CLI to be able to read the help output as JSON (for example `ember g --help --json`).

# Motivation

In our specific use case in [Ember Hearth](https://github.com/EmberTown/ember-hearth/) we would like to be able to render a dynamic GUI for some tasks, like generating blueprints. This way we could also include any blueprints added by addons. This will also apply to any other tools interfacing with Ember-CLI.

# Detailed design
We should probably make the internal help-functions (like `printBasicHelp` and `printDetailedHelp`) use JSON internally, and parse to human readable before printing (unless `--json` is specified).

I'm imagining the json output would be something like this:

```json
{
  "name":"generate",
  "description":"Generates new code from blueprints.",
  "aliases":["g"],
  "flags":[
    {
      "flag":"--verbose",
      "aliases":["-v"],
      "description":"Verbose output"
    }, {…}],
  "commands":[
    {
      "command":"template",
      "description":"Generates a template.",
      "arguments":["name"]
    },
    {
      "command":"model",
      "description":"Generate an ember-data model.",
      "arguments":[
        "name",
        {
          "argument":"attr:type",
          "description":"Add attributes to the model, e.g. 'name:String age:Number'",
          "multiple":true
        }]
    }, {…}]
}
```

Note that this output contains a bit more info than the current --help, specifically in the attr:type argument for the model command. This is something I feel is currently missing (I did not understand the model generator command without consulting a colleague, for example), and would be nice to add while we're at it.

It should be pretty straight forward to generate a human readable output from this JSON. There are a few things missing: However: The generate help command specifically groups commands by addon. I'm not sure how this should be accomplished, and if this matches the other help outputs. Ideally, any tools reading the JSON should be able to rely on the format being the same for all commands. This would keep the internals cleaner as well, including the human readable parser.

# Drawbacks

* Requires rewrite of help methods, possibly also for some addons (unless we can provide backwards compatability)
* Increases codebase size

# Alternatives

* We could standardize help output enough that it can be safely regexed by other tools
* We could not do this, and require any tools to update whenever Ember-CLI changes any commands

# Unresolved questions

* Internal architecture specifics (rewrite printBasicHelp or create a new setup, etc)
* Specifying JSON format details
* List any dependencies, like docs, that will need to be updated with this change
