- Start Date: 2015-08-18
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Summary

Adds command line completion *(tab completion)* to ember-cli that fills in partially typed commands by the user and suggests available sub commands or options. *(from now on i will refer to "strings in the command line" as "cli-commands" and "generate" and "new" as unprefixed "commands")*

# Motivation

With all the already existing commands and especially all blueprints, plus the fact that any addon can add even more blueprints to your tool kit, users can get overwhelmed. Currently, when you want to execute a specific task and you don't quite know the correct cli-command you have to invoke `ember help` which is noisy and slow *( especially when you just want to know the spelling of a specific thing)*. This feature will enable the user to choose from all existing cli-commands by pressing __[tab]__ or just to let ember-cli fill partially typed cli-commands for speed.

# Detailed design

The two main components of that feature are a __completion function__ that is responsible for the actual tab-completion, and a __generation function__ that will write the cli-command hierarchy together with metadata into a JSON file for fast processing.

## completion function

To enable this feature, a __completion function__ will run at the main entry point of the process *(making use of  [omelette](https://github.com/f/omelettev) for shell completion)*. On every __[tab]__ it will parse the command line and either completes a partially typed, unambiguous cli-command, or suggests possible cli-commands for the current context.

The user interface will work as you would expect it from a shell completion:

- it suggests all commands if none are typed yet

  ```
    $ ember <tab>
    > addon     destroy   help      install   serve     version
      build     generate  init      new       test
  ```
- it completes partially typed commands

  ```
    $ ember gen<tab>
    $ ember generate
  ```
- it completes to suggests commands based on user input (note how it __should__ understand aliases)

  ```
    $ ember g ad<tab>
    > adapter       adapter-test  addon
  ```
- it, by default, will not suggest options

  ```
    $ ember g resource <tab>
  ```
- it will suggest options on demand (note how it __should__ know when an option needs a value)

  ```
    $ ember g resource post --<tab>
    --dry-run         --in-repo-addon=  --verbose
    --dummy           --pod
  ```

## generation function

For a good user experience we don't want to figure out those suggestions on runtime or the completion feature would not be substantially faster then the `ember help` command. So there will be a __generation function__ that generates a JSON file once after ember-cli is installed and then during every `ember install some-addon` command to ensure that blueprints added by new addons are recognized aswell.

Here an example snippet of a cli-command with one cli-subcommand:
```
  ...
  {
    "name": "command-name",
    "aliases": [
      "cn",
      "c"
    ],
    "options": [
    ],
    "commands": [
      {
        "name": "some-subcommand",
        "aliases": [
        ],
        "options": [
          {
            "name": "pods",
            "type": "boolean"
          }
        ],
        "commands": [
        ]
      }
    ]
  }
  ...
```
The __generation function__ will, as a first step, iterate over all commands and reads the following properties:
- __name:__ a string, the autocompletion function will suggest
- __aliases:__ this is what the autocompletion function will accept in the cli-command chain
- __availableOptions:__ an array of options that need to have a `name` and a `type` property those will be accumulated for every cli-command in the cli-command chain and suggested on `some-command --<tab>`
- __cliCommands:__ can either be an array or a function that returns an array of objects that themselfs will be parsed for the properties in this list those cli-commands will be accepted as subcommands of the current cli-command.
- __skipHelp:__ whenever this property is set to true, the cli-command will also not be suggested by the autocompletion

Whenever an object does not have one of the above properties, a reasonable default is chosen (except for `name`. If it has no name, it will not be shown at all). This way it is easy to extend that feature in the future to handle arbitrary nested cli-commands.

# Alternatives

- Currently the __completion function__ will just expect certain properties to be on a cli-command, this way most commands and blueprints work out of the box but maybe some architectual pattern, like a cli-command mixin or the like would be more robust and obvious.
- Someone with more experience with ember-cli could have an idea of how to generate all cli-commands fast enough at runtime. So that we would not need to store the data in a JSON file.

# Unresolved questions

Currently the autocompletion will figure out your default shell and configures it to allow tab-completion for ember. However on __first-time usage__ you would need to resource your config file (or close and open your terminal) and I haven't figured out how to do this programmatically.
