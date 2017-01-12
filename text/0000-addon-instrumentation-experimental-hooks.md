- Start Date: Wednesday, 14 December 2016
- RFC PR: 
- Ember CLI Issue: 

# Summary

Add an instrumentation hook that is available to addons.  This enables users to
write addons that do things like summarize and report build performance
information.

- see https://github.com/ember-cli/ember-cli/issues/6349 for additional context.
- see https://github.com/ember-cli/ember-cli/pull/6606 for an experimental
  implementation.

# Motivation

Build performance is important to users.  We want to enable users to:

1. Easily discover which portions of their build are costly;
2. Be able to summarize and report build information in an addon;
3. Be able to write addons that analyze build performance instrumentation so
   that they can more easily help diagnose build performance issues in projects
   to which they do not have direct access.  This is of particular interest to
   @ember-cli/core &c.

In order to provide these hooks to enable iteration and experimentation prior to
making firm commitments to format, this rfc propose to initially expose them as
experiments (see the experiments section below).

# Detailed design

## Experiments

Experiments live in `lib/experiments/index.js`.  Unlike feature flags, there is
no need to strip them from production.  Experiments allow us to provide power
user features that are not fully stable without their resorting to private API
usage.

Experiments are available only in canary builds.  This is achieved by only
including `lib/experiements/index.js` in canary, and making it the entry point
for all experiments.

## Instrumentation Hook

We have already a build instrumentation hook as an
experiment in https://github.com/ember-cli/ember-cli/pull/6546

A more encompassing instrumentation hook is implemented in
https://github.com/ember-cli/ember-cli/pull/6606


The goal of this RFC is:

  1. To make the concept of experiments supported and explicit
  2. To promote this particular experiment to public API

### Enabling Instrumentation

Instrumentation is enabled if either the environment variable `BROCCOLI_VIZ` is
set to `1` or if `EMBER_CLI_INSTRUMENTATION` is set to `1`.

If `BROCCOLI_VIZ=1` then in addition to instrumentation hooks being invoked, a
serialized form of the instrumentation information is written to disk, that is
appropriate for consumption by [broccoli-viz](https://github.com/ember-cli/broccoli-viz) which is the current behaviour.

### Instrumentation

#### Hook

An addon that implements `instrumentation` will have this hook invoked when
instrumentation is enabled.

```js
module.exports = {
  name: 'my-great-addon',

  instrumentation(name, payload) {
    // format of instrumentation payload outlined below
  }
};
```

##### name

The `name` argument indicates what phase the instrumentation payload describes.
In beta and released versions this will always be a string.

On canary it could be a symbol from `lib/experiments` if we add more phases (eg
more fine-grained phases) for instrumentation information.

The initial set of phases this RFC advocates are:
  - `init`
  - `command`
  - `build`
  - `shutdown`

##### payload

`payload` is an object with two properties, `summary` and `graph`.

##### payload.summary

The exact format of `payload.summary` depends on the specific phase for which
the instrumentation hook was called.  In each case, the keys listed are the
minimum keys that are guaranteed to be present, but there is no guarantee that
additional information might not also be present.

###### init

`init` covers the period up to, but not including, command execution.  This
means it's mostly dealing with `require` time.

For `init`, the summary object has the following shape.

```js
{
  totalTime,
  platform: {
    name,
  },
}
```

- `summary.totalTime` The total time spent during `init`
- `summary.platform.name` The value of `process.platform`

###### build

`build` covers the time spent in an individual build or rebuild.

For `build`, the summary object has the following shape.

```js
{
  build: {
    type,
    count,
    outputChangedFiles

    // additional fields for rebuilds
    primaryFile,
    primaryFileCount,
    changedFiles
  },
  platform: {
    name,
  },
  output,
  totalTime,
  buildSteps,
}
```

- `summary.build.type` one of `'initial'` or `'rebuild'`
- `summary.build.count` the number of the build (0 for initial build, > 0 for
  rebuilds).
- `summary.build.outputChangedFiles` an array of paths to output files that
  changed during this build.  These paths are relative to the `dist` directory.
- `summary.build.primaryFile` only present for rebuilds.  Indicates the first
  file the watcher noticed had changed.
- `summary.build.changedFileCount` only present for rebuilds.  The number of
  files the watcher had noticed changed before the build started.
- `summary.build.changedFiles` only present for rebuilds. The first 10 files
  the watcher had noticed changed before the build started.
- `summary.platform.name` The value of `process.platform`
- `summary.output` The temp directory containing the results of the build.
- `summary.totalTime` The total time (in nanoseconds) of the build.
- `summary.buildSteps` The number of broccoli nodes built in this tree

###### command

`command` covers the time spent during a command.  When the command includes a
build, there will be overlap between `command` and `build`.  When the command is
`serve`, this overlap will include only the last build, to avoid memory leaks.

For `command`, the summary object has the following shape.

```js
{
  totalTime,
  platform: {
    name,
  },
  name,
  args
}
```

- `summary.totalTime` The total time spent during `init`
- `summary.platform.name` The value of `process.platform`
- `summary.name` The name of the command that was run
- `summary.args` The args of the command that was run

###### shutdown

`shutdown` covers the period from the command completing to process exit, ie
cleanup time.

For `shutdown`, the summary object has the following shape.

```js
{
  totalTime,
  platform: {
    name,
  },
}
```

- `summary.totalTime` The total time spent during `init`
- `summary.platform.name` The value of `process.platform`

##### payload.graph

`graph` is an object that represents the instrumentation information we have
gathered for the build.  It is a DAG, whose flow is inverted from the broccoli
graph. It has a single source node (currently `TreeMerger (all trees)`).
`payload.graph` is this single source node.

Each node in the graph provides an API for iterating its subgraph as well as
iterating its own stats. The specific nodes in the graph will change over time as
the instrumentation within ember-cli changes.  There is no particular guarantee
about what the nodes will be, although we will continue to ensure that its
`toJSON` format is consumable by
[broccoli-viz](https://github.com/ember-cli/broccoli-viz)

The API that each node supports is:

- `label`
- `toJSON`
- `adjacentIterator`
- `dfsIterator`
- `bfsIterator`

###### label

A POJO property that describes the node.  It will always include a `name`
property and for broccoli nodes will include a `broccoliNode` property.

Example:
```js
node.label === {
  name: 'TreeMerger (allTrees)',
  broccoliNode: true,
}
```

###### toJSON()

Returns a POJO that represents the serialized subgraph rooted at this node (the
entire tree if called on the root node).

There is no particular guarantee about the format, except that whatever it is
will be supported by [broccoli-viz](https://github.com/ember-cli/broccoli-viz).

Example:
```js
// for a graph
//  TreeMerger
//    |- Babel_1
//    |- Babel_2
//    |--|- Funnel
console.log(JSON.stringify(node.toJSON(), null, 2));
// might print
//
{
  nodes: [{
    id: 1,
    children: [2,3],
    stats: {
      time: {
        self: 5000000,
      },
      fs: {
        lstat: {
          count: 2,
          time: 2000000
        }
      },
      own: {
      }
    }
  }, {
    // ...
  }]
}
```

###### adjacentIterator

Returns an iterator that yields each adjacent outbound node.  There is no
guarantee about the order in which they are yielded.

```js
// for a tree
//  TreeMerger
//    |- Babel_1
//    |--|- Funnel
//    |- Babel_2
node.label.name === "TreeMerger";
for (n of node.adjacentIterator()) {
  console.log(n.label.name);
}
// prints
//
// Babel_1
// Babel_2


for (n of node.preOrderIterator(x => x.label.name === 'Babel_2')) {
  console.log(n.label.name);
}
// prints
//
//  TreeMerger
//  |- Babel_1
```

###### dfsIterator(until)

Returns an iterator that yields every node in the subgraph sourced at this node.
Nodes are yielded in depth-first order.  If the optional parameter `until` is
passed, nodes for which `until` returns `true` will not be yielded, nor will
nodes in their subgraph, unless those nodes are reachable by some other path.

Example:
```js
// for a graph
//  TreeMerger
//    |- Babel_1
//    |--|- Funnel
//    |- Babel_2
for (n of node.dfsIterator()) {
  console.log(n.label.name);
}
// prints
//
// TreeMerger
// Babel_1
// Funnel
// Babel_2
```

###### bfsIterator()

Returns an iterator that yields every node in the subgraph sourced at this node.
Nodes are yielded in breadth-first order.  If the optional parameter `until` is
passed, nodes for which `until` returns `true` will not be yielded, nor will
nodes in their subgraph, unless those nodes are reachable by some other path.


Example:
```js
// for a tree
//  TreeMerger
//    |- Babel_1
//    |--|- Funnel
//    |- Babel_2
for (n of node.bfsIterator()) {
  console.log(n.label.name);
}
// prints
//
// TreeMerger
// Babel_1
// Babel_2
// Funnel
```

###### statsIterator()

Returns an iterator that yields `[name, value]` pairs of stat names and values.

Example:
```js
  //  for a typical broccoli node 
  for ([statName, statValue] of node.statsIterator()) {
    console.log(statName, statValue);
  }
  // prints
  //
  // "time.self" 64232794
  // "fs.statSync.count" 40
  // "fs.statSync.time" 401232123
  // ...
  ```

# How We Teach This

This has no effect on day-to-day usage of ember-CLI.  It is a tool to help users
monitor and analyze their build performance, so documentation and teaching
belong primarily in `PERF_GUIDE.md`.  Having said that, we should also add a
section to `https://ember-cli.com/extending/` and the API docs to make using
this feature easier for addon authors and CLI power users.

# Drawbacks

* No drawbacks come to mind, besides the ever present issue of maintenance

# Alternatives

One alternative is the status quo: with `BROCCOLI_VIZ=1` users can output a file
with a similar format that they can post-process offline.  Although this works
for manual analysis, it is considerably more cumbersome for any automated system
(such as ongoing monitoring of build performance).  It also does not include
instrumentation outside of the build, most notably startup.

# Unresolved questions

- [heimdalljs-tree](https://github.com/heimdalljs/heimdalljs-tree) supports
  `Symbol.Iterator`; should we commit to this as part of our API?

