- 2018-01-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Deprecation of Ember.Logger

## Summary

This RFC provides for the deprecation and eventual removal of Ember.Logger.

## Motivation

There are a variety of features of Ember designed to support old browsers,
features that are no longer needed. Ember.Logger came into being because
the browser support for the console was inconsistent. In some browsers,
like Internet Explorer 9, the console only existed when the developer tools
panel was open, which caused references to the console object's methods to
fail to execute. Ember.Logger provided methods that would route to the 
console when it was available.

With Ember 3.x, Ember no longer supports these older browsers, and hence this
feature no longer serves a purpose. Removing it will make Ember smaller and 
lighter.

## Detailed design

### For framework developers

Remove the following 8 direct uses of Ember.Logging from ember.js and 
ember-data: 

ember-debug:
*  deprecate (deprecate.js) - Logger.warn
*  debug (index.js) - Logger.info
*  warn (warn.js) - Logger.warn
ember-routing (router.js):
*  transitioned to - Logger.log
*  preparing to transition to - Logger.log
*  intermediate-transitioned to - Logger.log
ember-testing:
*  Testing paused (pause_test.js) - Logger.info
*  Catch-all handler (adapter.js) - Logger.error
ember-data:
*  tests\test-helper.js:      Ember.Logger.log(reason, reason.stack);
(Note: I found one usage in ember-data, but emberobserver claims two more.)

Adjust all test code that redirects logging and sets it back:

ember:
*    routing: basic_test.js (adjust)
ember-application - system: 
*    logging_test.js (remove?)
*    dependency_injection : default_resolver.test.js (adjust)
ember-glimmer - integration: 
*    helpers: log-test.js (remove)

None of the uses of Ember.Logging in ember.js or ember-data are using 
Ember.debug, which will simplify matters.

### For Framework Users

For the most part, this is a 1:1 substitution of the global console object 
for Ember.Logging. The only anomaly remaining in Javascript engine console 
support is that versions of Node earlier than Node 9 do not support 
console.debug, so console.log will need to be used as the replacement for 
Logging.debug, rather than console.debug, unless the minimum Node support 
level is set to Node 9.

The following high-impact packages (9 or 10 or a * on EmberObserver) are 
calling Ember.Logging and should probably be given an early heads-up to 
adjust their code to use console in advance of the release containing the 
deprecation to limit the level of pain that their users experience. In 
the order of their reliance on Ember.Logging:

* ember-concurrency (15)
* ember-cli-deprecation-workflow (9)
* ember-stripe-service (9)
* semantic-ui-ember (7)
* ember-resolver (6)
* ember-cli-page-object (4) 
* ember-cli-sentry (3)
* ember-islands (3)
* ember-states (3)
* ember-cli-pagination (2)
* ember-cli-clipboard (1)
* ember-cli-fastboot (1)
* ember-elsewhere (1)
* ember-i18n (1)
* ember-simple-auth-token (1)
* ember-svg-jar (1)
* liquid-fire (1)

### Timeline

1. Remove all uses of Ember.Logging from other Ember and ember-data code.
This is particularly important for any methods involved in reporting 
deprecation. [Date? This could start now.]
2. Write and test codemod to change code to use console and make it available. [Any time]
3. Inform users that Ember.Logging will be deprecated and in what release. [When we know schedule]
4. Call deprecation APIs from all Logging methods in @ember-console/index.js [Once codemod is out there]
5. In documentation, offer suggestions for babel plugins available to suppress [Before we release]
console calls in production builds with any special instructions for how to 
use them in ember-cli broccoli builds.
6. In due season, the deprecation will appear in a release. [When?]

## How we teach this

### Official code bases and documentation

We do not currently actively teach the use of Ember.Logger. We will need to 
remove any passing references to Ember.Logger from the Ember guides 
and from the Super Rentals tutorial, if any such references exist. 

Once it is gone from the code, we need to verify it no longer appears in the 
API listings.

Likewise, we will need to remove the use of Ember.Logger from ember, ember-data, 
and the website.

## Drawbacks

191 projects in Ember Inspector are using it. It has been there and documented 
for a long time. So this deprecation will cause API churn on a lot of projects.
Hopefully providing a codemod to replace Ember.Logger calls with the 
corresponding console calls will make this transition relatively painless.

However, those using Logging.debug as something different from Logging.log
may beg to differ, even though under the covers Logging.debug only calls 
console.debug if it exists, calling console.log otherwise.

## Alternatives

1. Leave things as they are, perhaps providing @ember/console module interface.

2. Extract Ember.Logging into its own (tiny) @ember/logging package as a shim for users.

## Unresolved questions

What mechanisms are available with Babel to suppress console calls in production builds? 
Which if any would we point users toward?

How do I write a codemod? I may need help on this part.

How do we deal with Logging.debug in the codemod? Do we provide separate options for those
who might use Node < 9 and those who are confident they will only use Node 9 or later? 

## Raw Data - Remove

ember\lib\index.js:import Logger from 'ember-console';
ember\lib\index.js:Ember.Logger = Logger;
ember\tests\routing\basic_test.js:import Logger from 'ember-console';
ember\tests\routing\basic_test.js:let Router, App, router, registry, container, originalLoggerError, originalRenderSupport;
ember\tests\routing\basic_test.js:      originalLoggerError = Logger.error;
ember\tests\routing\basic_test.js:      Logger.error = originalLoggerError;
ember\tests\routing\basic_test.js:  Logger.error = function(initialMessage, errorMessage, errorStack) {
ember\tests\routing\basic_test.js:  Logger.error = function(initialMessage, errorMessage, errorStack) {
ember\tests\routing\basic_test.js:  Logger.error = function(initialMessage) {
ember\tests\routing\basic_test.js:  let originalLoggerError = Logger.error;
ember\tests\routing\basic_test.js:  Logger.error = function(initialMessage, errorMessage) {
ember\tests\routing\basic_test.js:  Logger.error = originalLoggerError;
ember\tests\routing\basic_test.js:  Logger.error = function() {
ember-application\tests\system\dependency_injection\default_resolver_test.js:    assert.equal(infoCount, 0, 'Logger.info should not be called if LOG_RESOLVER is not set');
ember-application\tests\system\logging_test.js:import Logger from 'ember-console';
ember-application\tests\system\logging_test.js:    this._originalLogger = Logger.info;
ember-application\tests\system\logging_test.js:    Logger.info = (_, {fullName}) => {
ember-application\tests\system\logging_test.js:    Logger.info = this._originalLogger;
ember-console\lib\index.js:  @class Logger
ember-console\lib\index.js:    Ember.Logger.log('log value of foo:', foo);
ember-console\lib\index.js:   @for Ember.Logger
ember-console\lib\index.js:    Ember.Logger.warn('Something happened!');
ember-console\lib\index.js:   @for Ember.Logger
ember-console\lib\index.js:    Ember.Logger.error('Danger! Danger!');
ember-console\lib\index.js:   @for Ember.Logger
ember-console\lib\index.js:    Ember.Logger.info('log value of foo:', foo);
ember-console\lib\index.js:   @for Ember.Logger
ember-console\lib\index.js:    Ember.Logger.debug('log value of foo:', foo);
ember-console\lib\index.js:   @for Ember.Logger
ember-console\lib\index.js:   If the value passed into `Ember.Logger.assert` is not truthy it will throw an error with a stack trace.
ember-console\lib\index.js:    Ember.Logger.assert(true); // undefined
ember-console\lib\index.js:    Ember.Logger.assert(true === false); // Throws an Assertion failed error.
ember-console\lib\index.js:    Ember.Logger.assert(true === false, 'Something invalid'); // Throws an Assertion failed error with message.
ember-console\lib\index.js:   @for Ember.Logger
ember-debug\lib\deprecate.js:import Logger from 'ember-console';
ember-debug\lib\deprecate.js:    Logger.warn(`DEPRECATION: ${updatedMessage}`);
ember-debug\lib\deprecate.js:      Logger.warn(`DEPRECATION: ${updatedMessage}${stackStr}`);
ember-debug\lib\index.js:import Logger from 'ember-console';
ember-debug\lib\index.js:    Logger.debug(`DEBUG: ${message}`);
ember-debug\lib\index.js:    Logger.info.apply(undefined, arguments);
ember-debug\lib\warn.js:import Logger from 'ember-console';
ember-debug\lib\warn.js:    Logger.warn(`WARNING: ${message}`);
ember-debug\lib\warn.js:    if ('trace' in Logger) {
ember-debug\lib\warn.js:      Logger.trace();
ember-glimmer\tests\integration\helpers\log-test.js:import Logger from 'ember-console';
ember-glimmer\tests\integration\helpers\log-test.js:    this.originalLog = Logger.log;
ember-glimmer\tests\integration\helpers\log-test.js:    Logger.log = (...args) => {
ember-glimmer\tests\integration\helpers\log-test.js:    Logger.log = this.originalLog;
ember-routing\lib\system\router.js:import Logger from 'ember-console';
ember-routing\lib\system\router.js:        routerMicrolib.log = Logger.debug;
ember-routing\lib\system\router.js:        Logger.log(`Transitioned into '${EmberRouter._routePath(infos)}'`);
ember-routing\lib\system\router.js:        Logger.log(`Preparing to transition from '${EmberRouter._routePath(oldInfos)}' to '${EmberRouter._routePath(newInfos)}'`);
ember-routing\lib\system\router.js:        Logger.log(`Intermediate-transitioned into '${EmberRouter._routePath(infos)}'`);
ember-routing\lib\system\router.js:  Logger.error.apply(this, errorArgs);
ember-testing\lib\helpers\pause_test.js:import Logger from 'ember-console';
ember-testing\lib\helpers\pause_test.js:  Logger.info('Testing paused. Use `resumeTest()` to continue.');
ember-testing\lib\test\adapter.js:import Logger from 'ember-console';
ember-testing\lib\test\adapter.js:  Logger.error(error.stack);

ember-data:
    tests\test-helper.js:      Ember.Logger.log(reason, reason.stack);
    
    already using console.log in several places, mostly in addons:
    addon\-private\system\model\model.js:      console.log(field, kind);
    addon\-private\system\model\model.js:      console.log(name, meta);
    addon\-private\system\model\model.js:      console.log(field, type);
    addon\-private\system\model\model.js:      console.log(name, meta);
    addon\-private\system\model\model.js:      console.log(name, type);
    addon\-private\system\model\states.js:      console.log("Received myEvent with", param);
    addon\-private\system\record-arrays\adapter-populated-record-array.js:    console.log(admins.get("length")); // 42
    addon\-private\system\record-arrays\adapter-populated-record-array.js:    console.log(admins.get("length")); // 123
    addon\-private\system\store.js:      console.log(`Currently logged in as ${username}`);
    addon\-private\system\store.js:      console.log(user); // null
    benchmarks\config.js:    // { key: "stats.store.modelFor", name: 'modelFor', transform: function(t, c) { console.log(t); return t; }, rollup: false },
    node-tests\nodetest-runner.js:  console.log('Verifing `.only` in tests');
    node-tests\nodetest-runner.js:    console.log('No `.only` found');
    tests\test-helper.js:      Ember.Logger.log(reason, reason.stack);

Really useful URL: https://emberobserver.com/code-search?codeQuery=Ember.Logger
Results:
    191 Add-Ons use Ember.Logger.
    Notables:
        ember-source (19 usages)
        ember-concurrency (15 usages)
        ember-stripe-service (9 usages)
        ember-cli-page-object (4 usages) 
        ember-data (3 usages)
        ember-cli-fastboot (1 usage)
        ember-i18n (1 usage)
        ember-simple-auth-token (1 usage)
        ember-svg-jar (1 usage)
        liquid-fire (1 usage)

Preparation for writing this section:

0. Read the source of Ember.Logger and enumerate the methods, understanding any corner cases they may raise for replacement with console commands. [Done]

ember-console exports assert, debug, error, info, log, warn.
Chrome, Firefox, Edge, IE11, Safari support all of these.
Node 6 supports all of them except debug, which doesn't appear until Node 9.x.

So the only thing even remotely edgy surrounds what to do with Logger.debug(). The current implementation checks if console supports debug and either issues console.debug() or console.log() accordingly.

1. Mentally implement the deprecation itself on each method and enumerate the likely fallout, if any. [Done]

The mechanism that reports deprecations uses these calls, so we would need that code to call the console directly before we attempted to flag these calls for deprecation. In fact, we would probably want to replace all eight of the internal usages.

A number of tests redefine Ember.Logger temporarily as a means of interception. If all of those uses are going to console, it should still be possible to redirect those, but it may make debugging uncomfortable.

2. Mentally implement the codemod to replace Ember.Logging calls with console calls. Understand what will be easy or difficult.

I have real doubts about my ability to do this - especially if it involves familiarity with ASTs - but it could be an opportunity to learn.

3. Enumerate the places in the emberjs source that call Ember.Logger and its methods. Get a feeling for how many do and what methods they use a lot of. [Done]

4. Research any Babel plugins which remove console calls.

