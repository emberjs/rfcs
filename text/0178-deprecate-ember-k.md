---
Start Date: 2016-11-18
RFC PR: #178
Ember Issue: #14746

---

# Summary

The `Ember.K` utility function is a low level utility that has lost most of its value today.

# Motivation

Let's start explaining what `Ember.K` is.

It is an utility function to avoid boilerplace code and limit the creation of function instances
in Ember's internals. The source code for this API is the following:

```js
Ember.K = function() {
  return this;
}
```

In a world of globals, writing `somefn: Ember.K` was effectively shorter
than writing

```js
someFn: function() {
  return this;
}
```

and generated fewer function allocations.

However with the introduction of ES6 modules and the modularization of Ember
in process (#176), keeping this feature would require to design an import path for it.

While doable, the transpiled output is actually bigger then defining the functions
inline, specially with the ES6 shorthand method syntax, and the perf difference
of saving a few function allocations is negligible.

The second downside of reusing the same instance in many places is that if for
some reason the VM deoptimizes that function, that deoptimization is spreaded
across all the usages of `Ember.K`.

Third, the chainable nature of `Ember.K` tends to surprise the users:

```js
let derp = {
  foo: Ember.K,
  bar: Ember.K,
  baz: Ember.K
}

derp.foo().bar().baz(); // O_o
```

And lastly but more importantly for simplicity. Consider the following code:

```js
export default Component.extend({
  onSubmit() {}
});
```

Any JS developer will understand that this is an empty function and will probably understand
that is a placeholder to provide your own function instead. However, JS developers that come
across `Ember.K` for the first time will se this:


```js
export default Component.extend({
  onSubmit: Ember.K
});
```

and will think that it is some cryptic Ember magic that they have to learn.

# Transition Path

The necessary first step is to make sure Ember, Ember Data and other pieces of the
ecosystem don't use `Ember.K` internally.

Phased approach:
* Deprecate `Ember.K`: Use the deprecation API to signal the deprecation, and deprecation guide entry.
  Target version will be 3.0, as usual.
* Add rule to ember-watson
* Do not include export path in https://github.com/emberjs/rfcs/pull/176, but include it until 3.0 in the "globals" build.

# How We Teach This

Since it is a very low-level utility,
the amount of people that will have to update their code should be a limited set of developers, working mostly on addons.
This allows us to cover most use cases with the following strategy:
* Improve the current documentation to help developers finding the API for the first time in the future;
* Provide an automated path forward through tooling such as [ember-watson](https://github.com/abuiles/ember-watson). (see Addendum 1)
* Introduce the mandatory entry in the deprecations guide referencing the automated tooling.

If this RFC is done as part of https://github.com/emberjs/rfcs/pull/176 as suggested,
it will be in the document or blog post announcing the final transition to modules.

# Drawbacks

Although this utility is not very used, there is a chance that is used by some
addons and as a placeholder of a hook that is called a lot and would trigger
hundreds of deprecation warnings.

# Alternatives

The feature could continue to exist.

# Addenda

## Addendum 1 - Codemod to automatically remove all usages of `Ember.K` on any project.

https://github.com/cibernox/ember-k-codemod

To use it you can install it globally and invoke the command on any app or addon.

The commands **requires** the user to decide the approach to replace occurenced of `Ember.K`. The
possible flags are `--empty` and `--return-this`.

- `--empty` replaces `Ember.K` with an empty function. This leads to the most idiomatic and
  intention-revealing code, but does not allow chaining, like the original `Ember.K` did.
  Despite of that, chaining `Ember.K` was such an uncommon patterns that we thing virtually
  everybody can use this option.
- `--return-this` replaces `Ember.K` with a function that just returns `this`. This allows chaining
  like the original one.


Example usage:

```
npm install -g ember-k-codemod && ember-k-codemod --empty
```

Versions of [ember-watson](https://github.com/abuiles/ember-watson) starting in `0.8.5` wrap this
codemod so you can achieve the same transformation with it:

```
ember-watson remove-ember-k --empty

// or if installed as an addon

ember watson:remove-ember-k --empty
```

## Addendum 2 - `Ember.K` usage across published addons

```
ae-select/addon/components/ae-select.js:  action: Ember.K, // action to fire on change
antd-ember/addon/components/io-searchable-select/searchable-select.js:    'on-change': Ember.K,
antd-ember/addon/components/io-searchable-select/searchable-select.js:    'on-add': Ember.K,
antd-ember/addon/components/io-searchable-select/searchable-select.js:    'on-search': Ember.K,
antd-ember/addon/components/io-searchable-select/searchable-select.js:        return this.get('on-add') !== Ember.K;
antd-ember/addon/components/io-searchable-select/searchable-select.js:        return this.get('on-search') === Ember.K;
ella-list-view/addon/views/list-item.coffee:  prepareContent: Ember.K
ella-list-view/addon/views/list-item.coffee:  teardownContent: Ember.K
ella-list-view/addon/views/list.coffee:  arrayWillChange: Ember.K
ella-list-view/addon/views/list.coffee:  didScrollToTop: Ember.K
ella-list-view/addon/views/list.coffee:  didScrollToBottom: Ember.K
ella-list-view/addon/views/list.coffee:  visibleItemsDidChange: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:    if @didRequestRange isnt Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:    unless (@didRequestLength is Ember.K) or get(@, 'isRequestingLength')
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  sparseContentWillChange: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  sparseContentDidChange: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  didReplaceSparseArrayItem: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  didRequestIndex: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  didRequestRange: Ember.K
ella-sparse-array-controller/addon/controllers/sparse-array.coffee:  didRequestLength: Ember.K
elvis-network/addon/components/visjs-child.js:  didCreateLayer: Ember.K,
elvis-network/addon/components/visjs-child.js:  willDestroyLayer: Ember.K,
ember-animate/ember-animate.js:        willAnimateIn : Ember.K,
ember-animate/ember-animate.js:        willAnimateOut : Ember.K,
ember-animate/ember-animate.js:        didAnimateIn : Ember.K,
ember-animate/ember-animate.js:        didAnimateOut : Ember.K,
ember-animate/ember-animate.js:        _currentViewWillChange : Ember.K,
ember-aupac-typeahead/addon/components/aupac-typeahead.js:  action: Ember.K, //@public
ember-aupac-typeahead/addon/components/aupac-typeahead.js:  source : Ember.K, //@public
ember-autoresize/addon/mixins/autoresize.js:let trim = Ember.K;
ember-bootstrap/addon/components/bs-form-element.js:  setupValidations: Ember.K,
ember-bootstrap/addon/components/bs-select.js:  action: Ember.K, // action to fire on change
ember-bootstrap-components/addon/components/bs-select.js:  action: Ember.K, // action to fire on change
ember-bugsnag/app/instance-initializers/bugsnag.js:  let originalOnError = Ember.onerror || Ember.K;
ember-charts/addon/components/bubble-chart.js:      return Ember.K;
ember-charts/addon/components/bubble-chart.js:      return Ember.K;
ember-charts/addon/components/horizontal-bar-chart.js:      return Ember.K;
ember-charts/addon/components/horizontal-bar-chart.js:      return Ember.K;
ember-charts/addon/components/pie-chart.js:      return Ember.K;
ember-charts/addon/components/pie-chart.js:      return Ember.K;
ember-charts/addon/components/scatter-chart.js:      return Ember.K;
ember-charts/addon/components/scatter-chart.js:      return Ember.K;
ember-charts/addon/components/stacked-vertical-bar-chart.js:      return Ember.K;
ember-charts/addon/components/stacked-vertical-bar-chart.js:      return Ember.K;
ember-charts/addon/components/time-series-chart.js:      return Ember.K;
ember-charts/addon/components/time-series-chart.js:      return Ember.K;
ember-charts/addon/components/vertical-bar-chart.js:      return Ember.K;
ember-charts/addon/components/vertical-bar-chart.js:      return Ember.K;
ember-charts/addon/mixins/legend.js:      return Ember.K;
ember-charts/addon/mixins/legend.js:      return Ember.K;
ember-charts/addon/mixins/resize-handler.js:  onResizeStart: Ember.K,
ember-charts/addon/mixins/resize-handler.js:  onResizeEnd: Ember.K,
ember-charts/addon/mixins/resize-handler.js:  onResize: Ember.K,
ember-charts/dependencies/ember-addepar-mixins/resize_handler.js:  onResizeStart: Ember.K,
ember-charts/dependencies/ember-addepar-mixins/resize_handler.js:  onResizeEnd: Ember.K,
ember-charts/dependencies/ember-addepar-mixins/resize_handler.js:  onResize: Ember.K,
ember-cli-adapter-pattern/tests/dummy/app/starships/starship.js:  _makeItSo: Ember.K
ember-cli-airbrake/README.md:In all cases, an `airbrake` service will be exposed. If airbrake isn't configured the airbrake service uses the Ember.K "no-op" function for its methods. This facilitates the usage of the airbrake service without having to add environment-checking code in your app.
ember-cli-airbrake/README.md:exist, but all its methods will be no-ops (`Ember.K`). This way your tests will still run happily even
ember-cli-airbrake/app/instance-initializers/ember-cli-airbrake.js:  let originalOnError = Ember.onerror || Ember.K;
ember-cli-analytics/addon/integrations/base.js:  trackPage: Ember.K,
ember-cli-analytics/addon/integrations/base.js:  trackEvent: Ember.K,
ember-cli-analytics/addon/integrations/base.js:  trackConversion: Ember.K,
ember-cli-analytics/addon/integrations/base.js:  identify: Ember.K,
ember-cli-analytics/addon/integrations/base.js:  alias: Ember.K
ember-cli-analytics/tests/unit/mixins/trackable-test.js:  const analytics = { trackPage: Ember.K };
ember-cli-aptible-shared/tests/unit/utils/changeset-test.js:      initialValue: Ember.K
ember-cli-aptible-shared/tests/unit/utils/changeset-test.js:      key: Ember.K
ember-cli-bugsnag/app/instance-initializers/bugsnag.js:      const originalDidTransition = router.didTransition || Ember.K;
ember-cli-coreweb/app/initializers/ember-coreweb.js:  initialize: Ember.K
ember-cli-dialog/packages/ember-dialog/lib/ember-initializer.js:// var K = Ember.K;
ember-cli-dimple/addon/components/dimple-chart/component.coffee:  customizeChart: Ember.K
ember-cli-dimple/addon/components/dimple-chart/component.js:  customizeChart: Ember.K,
ember-cli-dimple/addon/mixins/resize.js:  onResizeStart: Ember.K,
ember-cli-dimple/addon/mixins/resize.js:  onResizeEnd: Ember.K,
ember-cli-dimple/addon/mixins/resize.js:  onResize: Ember.K,
ember-cli-dynamic-forms/addon/components/dynamic-form.js:  renderSchema: Ember.K,
ember-cli-erraroo/addon/erraroo.js:    const oldEmberOnerror = Ember.onerror || Ember.K;
ember-cli-fullpagejs-view/addon/initializers/remove-fullpage.js:  initialize: Ember.K
ember-cli-infinite-scroll/addon/mixins/infinite-scroll.js:  beforeInfiniteQuery: Ember.K,
ember-cli-ion-rangeslider/addon/ember-ion-rangeslider.js:          onChange: Ember.K,
ember-cli-ion-rangeslider/addon/ember-ion-rangeslider.js:      options.onFinish = Ember.K;
ember-cli-jsoneditor/addon/components/json-editor.js:  onChange: Ember.K,
ember-cli-jsoneditor/addon/components/json-editor.js:  onError: Ember.K,
ember-cli-jsoneditor/addon/components/json-editor.js:  onModeChange: Ember.K,
ember-cli-jsoneditor/addon/components/json-editor.js:  onEditable: Ember.K,
ember-cli-maskedinput/addon/components/masked-input.js:  'on-change': Ember.K,
ember-cli-nvd3/addon/components/nvd3-chart.js:  beforeSetup: Ember.K,
ember-cli-nvd3/addon/components/nvd3-chart.js:  afterSetup: Ember.K,
ember-cli-nvd3-multichart/addon/components/nvd3-multichart.js:  chartContextFn: Ember.K,
ember-cli-remote-inspector/tests/acceptance.js:    this.ember.kill('SIGINT');
ember-cli-remote-inspector/tests/acceptance.js:    this.ember.kill('SIGINT');
ember-cli-selectize/addon/components/ember-selectize.js:  _groupedContentArrayWillChange: Ember.K,
ember-cli-visjs/addon/components/visjs-child.js:  didCreateLayer: Ember.K,
ember-cli-visjs/addon/components/visjs-child.js:  willDestroyLayer: Ember.K,
ember-collection/addon/components/ember-collection.js:          willChange: Ember.K,
ember-collection/addon/components/ember-collection.js:          willChange: Ember.K,
ember-concurrency/addon/utils.js:      let disposer = typeof maybeDisposer === 'function' ? maybeDisposer : Ember.K;
ember-confirm-dialog/addon/components/confirm-dialog.js:  confirmAction: Ember.K,//optional action executed when user confirms the dialog
ember-confirm-dialog/addon/components/confirm-dialog.js:  cancelAction: Ember.K,//optional action executed when user cancels confirmation dialog
ember-cookie-consent-cnil/app/mixins/click-else-where.js:  onOutsideClick: Ember.K,
ember-data-model-fragments/addon/states.js:  propertyWasReset: Ember.K,
ember-data-model-fragments/addon/states.js:  becomeDirty: Ember.K,
ember-data-model-fragments/addon/states.js:  rolledBack: Ember.K,
ember-data-model-fragments/addon/states.js:      pushedData: Ember.K,
ember-data-model-fragments/addon/states.js:      didCommit: Ember.K,
ember-data-sails/addon/initializers/ember-data-sails.js:      methods[level] = Ember.K;
ember-dev-fixtures/private/utils/dev-fixtures/module.js:      define(this.get('fullPath'), ['exports'], Ember.K);
ember-dp-map/addon/components/_dp-base-map-element.js:  didLoadMap: Ember.K
ember-drag-drop/addon/mixins/droppable.js:  acceptDrop: Ember.K,
ember-drag-drop/addon/mixins/droppable.js:  handleDragOver: Ember.K,
ember-drag-drop/addon/mixins/droppable.js:  handleDragOut: Ember.K,
ember-form-object/tests/unit/forms/model-form-test.js:  }).catch(Ember.K);
ember-froala/addon/components/froala-editor.js:        var buttons = this.get('customButtons') || Ember.K;
ember-google-charts/tests/integration/components/options-change-test.js:    this.on('chartDidRender', Ember.K);
ember-img-cache/app/initializers/ember-img-cache.js:  initialize: Ember.K
ember-img-manager/app/utils/img-manager/img-clone-holder.js:  this.handler = Ember.K;
ember-img-manager/app/utils/img-manager/img-clone-holder.js:      this.handler = Ember.K;
ember-img-manager/app/utils/img-manager/img-clone-holder.js:   * @param {Function} [handler=Ember.K]
ember-img-manager/app/utils/img-manager/img-clone-holder.js:    this.handler = handler || Ember.K;
ember-infinity/tests/unit/mixins/route-test.js:      pushObjects: Ember.K,
ember-infinity/tests/unit/mixins/route-test.js:        pushObjects: Ember.K,
ember-infinity/tests/unit/mixins/route-test.js:        pushObjects: Ember.K,
ember-jsonapi-resources/addon/adapters/application.js:    let cleanup = Ember.K;
ember-jsonapi-resources/tests/unit/adapters/application-test.js:  adapter.serializer = {deserialize: sandbox.spy(), deserializeIncluded: Ember.K};
ember-jsonapi-resources/tests/unit/adapters/application-test.js:  adapter.serializer = { deserialize: sandbox.spy(), deserializeIncluded: Ember.K };
ember-jsonapi-resources/tests/unit/mixins/fetch-test.js:  this.subject.serializer = { deserialize: function(res) { return res.data; }, deserializeIncluded: Ember.K };
ember-jsonapi-resources/tests/unit/mixins/resource-operations-test.js:        trigger: Ember.K
ember-jsonapi-resources/tests/unit/mixins/resource-operations-test.js:      guns: {kind: 'hasMany', mapBy: Ember.K }, // hasMany('guns')
ember-jsonapi-resources/tests/unit/mixins/resource-operations-test.js:      horse: {kind: 'hasOne', get: Ember.K } // hasOne('horse')
ember-jsonapi-resources-form/addon/components/resource-form.js:    if (!action) { return Ember.K; /* fail silently if no action */ }
ember-jsonapi-resources-list/addon/mixins/controllers/jsonapi-list.js:    filtering: Ember.K,
ember-key-responder/app/key-responder.js:    comments for Ember.KeyResponderStack above for more insight.
ember-list-card/addon/components/list-card/header-dropdown-item.js:   onSelect: Ember.K,
ember-list-card/addon/components/list-card/header-dropdown-item.js:   onDeselect: Ember.K,
ember-list-card/addon/components/list-card/header-dropdown.js:  onItemSelect: Ember.K,
ember-list-card/addon/components/list-card/header.js:  onQueryOptionSelect: Ember.K,
ember-material-design/addon/mixins/events.js:    start: Ember.K,
ember-material-design/addon/mixins/events.js:    move: Ember.K,
ember-material-design/addon/mixins/events.js:    end: Ember.K
ember-material-design/addon/mixins/gesture-events.js:    onStart: Ember.K,
ember-material-design/addon/mixins/gesture-events.js:    onMove: Ember.K,
ember-material-design/addon/mixins/gesture-events.js:    onEnd: Ember.K,
ember-material-design/addon/mixins/gesture-events.js:    onCancel: Ember.K,
ember-material-design/app/services/ripple.js:            return Ember.K;
ember-metrics/tests/dummy/app/metrics-adapters/local-dummy-adapter.js:  init: Ember.K,
ember-metrics/tests/dummy/app/metrics-adapters/local-dummy-adapter.js:  willDestroy: Ember.K
ember-mixpanel/addon/mixpanel.js:          return Ember.K;
ember-mixpanel/addon/mixpanel.js:      return Ember.K;
ember-notifyme/addon/objects/notification-message.js:  onClick: Ember.K,
ember-notifyme/addon/objects/notification-message.js:  onClose: Ember.K,
ember-notifyme/addon/services/notification-service.js:           onClick: options.onClick || Ember.K,
ember-notifyme/addon/services/notification-service.js:           onClose: options.onClose || Ember.K,
ember-notifyme/addon/services/notification-service.js:           onCloseTimeout: options.onCloseTimeout || Ember.K,
ember-off-canvas-components/addon/initializers/custom-events.js:  initialize: Ember.K
ember-pardon/addon/mixins/ember-pardon.js:	beforeDestroy: Ember.K,
ember-phoenix-channel/tests/integration/components/socket-message-log-test.js:  on: Ember.K
ember-pikaday/addon/components/pikaday-inputless.js:  onPikadayOpen: Ember.K,
ember-pikaday/addon/components/pikaday-inputless.js:  onPikadayClose: Ember.K,
ember-pikaday/addon/mixins/pikaday.js:  onOpen: Ember.K,
ember-pikaday/addon/mixins/pikaday.js:  onClose: Ember.K,
ember-pikaday/addon/mixins/pikaday.js:  onSelection: Ember.K,
ember-pikaday/addon/mixins/pikaday.js:  onDraw: Ember.K,
ember-pikaday-with-time/addon/components/pikaday-input.js:  onPikadayOpen: Ember.K,
ember-pikaday-with-time/addon/components/pikaday-input.js:  onPikadayRedraw: Ember.K,
ember-processes/addon/utils.js:      let disposer = typeof maybeDisposer === 'function' ? maybeDisposer : Ember.K;
ember-render-stack/addon/route-mixin.js:  renderStack: Ember.K,
ember-restless/dist/ember-restless.js:  var noop = Ember.K;
ember-restless/dist/ember-restless.js:    _onPropertyChange: Ember.K
ember-restless/src/model/read-only-model.js:  _onPropertyChange: Ember.K
ember-restless/src/model/state.js:var noop = Ember.K;
ember-reveal-js/addon/components/reveal-presentation/component.js:      before: Ember.K,
ember-rl-dropdown/addon/mixins/rl-dropdown-component.js:  onOpen: Ember.K,
ember-rl-dropdown/addon/mixins/rl-dropdown-component.js:  onClose: Ember.K,
ember-searchable-select/addon/components/searchable-select.js:  'on-change': Ember.K,
ember-searchable-select/addon/components/searchable-select.js:  'on-add': Ember.K,
ember-searchable-select/addon/components/searchable-select.js:  'on-search': Ember.K,
ember-searchable-select/addon/components/searchable-select.js:  'on-close': Ember.K,
ember-searchable-select/addon/components/searchable-select.js:    return this.get('on-add') !== Ember.K;
ember-searchable-select/addon/components/searchable-select.js:    return this.get('on-search') === Ember.K;
ember-searchable-select/tests/dummy/app/pods/components/options-table/template.hbs:    on-change                 | Specify your own named action to trigger when the selection changes. eg. <code>(action "update")</code> <br> For single selection (default behaviour), the selected object is sent as an argument. For multiple selections, an array of options is sent. | Ember action  | Ember.K
ember-searchable-select/tests/dummy/app/pods/components/options-table/template.hbs:    on-add                    | Allow unfound items to be added to the content array by specifying your own named action. eg. `(action "addNew")` The new item name is sent as an argument. You must handle adding the item to the content array and selecting the new item outside the component. | Ember action  | Ember.K
ember-searchable-select/tests/dummy/app/pods/components/options-table/template.hbs:    provided.     | Ember action    | Ember.K
ember-searchable-select/tests/dummy/app/pods/components/options-table/template.hbs:    on-close                  | Specify your own named action to trigger when the menu closes. Useful hook for clearing out content that was previously passed in with AJAX. | Ember action | Ember.K
ember-select-list/addon/components/select-list.js:  action: Ember.K, // action to fire on change
ember-smart-banner/addon/components/smart-banner.js:      const visitFn = Ember.getWithDefault(this, 'attrs.onvisit', Ember.K);
ember-smart-banner/addon/components/smart-banner.js:      const closeFn = Ember.getWithDefault(this, 'attrs.onclose', Ember.K);
ember-sqlite-adapter/addon/migration.js:  run: Ember.K,
ember-stripe-service/addon/services/stripe.js:      this.card[name] = Ember.K;
ember-table/addon/components/ember-table.js:    addColumn: Ember.K,
ember-table/addon/components/ember-table.js:    sortByColumn: Ember.K
ember-table/addon/mixins/mouse-wheel-handler.js:  onMouseWheel: Ember.K,
ember-table/addon/mixins/resize-handler.js:  onResizeStart: Ember.K,
ember-table/addon/mixins/resize-handler.js:  onResizeEnd: Ember.K,
ember-table/addon/mixins/resize-handler.js:  onResize: Ember.K,
ember-table/addon/mixins/scroll-handler.js:  onScroll: Ember.K,
ember-table/addon/mixins/touch-move-handler.js:  onTouchMove: Ember.K,
ember-table/addon/models/column-definition.js:  setCellContent: Ember.K,
ember-table/addon/views/lazy-item.js:  prepareContent: Ember.K,
ember-table/addon/views/lazy-item.js:  teardownContent: Ember.K,
ember-ted-select/README.md:      <td><code>Ember.K</code> (noop)</td>
ember-ted-select/tests/dummy/app/pods/application/template.hbs:        <td><code>Ember.K</code> (noop)</td>
ember-theater/addon/ember-theater/director/directions/sound.js:  fadeTo(volume, duration, callback = Ember.K) {
ember-to-string/tests/unit/helpers/to-string-test.js:    lookup: Ember.K
ember-ui-components/addon/mixins/click-outside.js:  handleClickOutside: Ember.K,
ember-unauthorized/tests/unit/mixins/access-test.js:  subject.set('authorize', Ember.K);
ember-unauthorized/tests/unit/mixins/access-test.js:  subject.set('authorize', Ember.K);
ember-unauthorized/tests/unit/mixins/route-access-test.js:      transitionTo: Ember.K
ember-watson/tests/fixtures/resource-router-mapping/new-complex-ember-cli-sample.js:    }, Ember.K);
ember-watson/tests/fixtures/resource-router-mapping/old-complex-ember-cli-sample.js:    this.resource('dashboard', Ember.K);
emberx-select/addon/components/x-select.js:  "on-blur": Ember.K,
emberx-select/addon/components/x-select.js:  "on-click": Ember.K,
emberx-select/addon/components/x-select.js:  "on-change": Ember.K,
emberx-select/addon/components/x-select.js:  "on-focus-out": Ember.K,
fireplace/addon/collections/indexed.js:      then(Ember.K.bind(this));
fireplace/addon/collections/object.js:    const promise = this.listenToFirebase().then(Ember.K.bind(this));
justa-table/addon/components/table-vertical-collection.js:  'on-row-click': Ember.K
list-view/addon/list-view-mixin.js:  _scrollTo: Ember.K,
list-view/addon/list-view-mixin.js:  arrayWillChange: Ember.K,
list-view/addon/reusable-list-item-view.js:  prepareForReuse: Ember.K,
mantel/addon/fireplace/collections/indexed.js:      then(Ember.K.bind(this));
mantel/addon/fireplace/collections/object.js:    var promise = this.listenToFirebase().then(Ember.K.bind(this));
plaid/addon/mixins/dimensions.js:  didMeasureDimensions: Ember.K,
plaid/addon/mixins/global-resize.js:  didResize: Ember.K
spree-ember-paypal-express/addon/services/paypal-express.js:  spree: Ember.K,
torii/addon/services/torii-session.js:  setUnknownProperty: Ember.K,
torii/tests/unit/redirect-handler-test.js:    close: Ember.K
torii/tests/unit/services/popup-test.js:    focus: Ember.K,
torii/tests/unit/services/popup-test.js:    close: Ember.K
```

## Addendum 3 - `Ember.K` usage via destructuring across published addons

```
CogAuth/tests/helpers/flash-message.js:const { K } = Ember;
ember-annotative-models/addon/utils/action.coffee:{K, isBlank, A} = Ember
ember-annotative-models/tests/unit/utils/action-test.coffee:{K} = Ember
ember-cli-airbrake/addon/utils/get-client.js:const { K } = Ember;
ember-cli-flash/blueprints/ember-cli-flash/files/tests/helpers/flash-message.js:const { K } = Ember;
ember-cli-mapkit/addon/components/ui-abstract-map.js:const {isEmpty, computed, on, K, run} = Ember;
ember-cli-pixijs/addon/components/pixi-canvas.js:const { Component, computed, K } = Ember;
ember-click-outside/addon/mixins/click-outside.js:const { computed, K } = Ember;
ember-composable-helpers/addon/-private/create-needle-haystack-helper.js:const { K, isEmpty } = Ember;
ember-composable-helpers/tests/unit/helpers/pipe-test.js:const { RSVP: { resolve, reject }, K } = Ember;
ember-composable-helpers/tests/unit/helpers/queue-test.js:const { RSVP: { resolve, reject }, K } = Ember;
ember-d3-helpers/tests/unit/helpers/d3-line-test.js:const { K } = Ember;
ember-form-object/addon/forms/model-form.js:const { ObjectProxy, computed, computed: { readOnly }, assert, Logger, run, A: createArray, K: noop, String: { camelize } } = Ember;
ember-form-tool/addon/mixins/drag-drop.coffee:{K, Mixin, computed: {equal}} = Ember
ember-functional-helpers/addon/-private/create-needle-haystack-helper.js:const { K, isEmpty } = Ember;
ember-functional-helpers/tests/unit/helpers/pipe-test.js:const { RSVP: { resolve, reject }, K } = Ember;
ember-functional-helpers/tests/unit/helpers/queue-test.js:const { RSVP: { resolve, reject }, K } = Ember;
ember-imdt-magic-crud/tests/helpers/flash-message.js:const { K } = Ember;
ember-keyword-complete/addon/components/keyword-complete.js:const {observer, computed, run, assert, K, $} = Ember;
ember-leaflet/addon/components/base-layer.js:const { assert, computed, Component, run, K, A, String: { classify } } = Ember;
ember-light-table/tests/helpers/responsive.js:const { K, getOwner } = Ember;
ember-metrics/tests/unit/services/metrics-test.js:const { get, set, K } = Ember;
ember-paper/addon/components/paper-autocomplete.js:const { assert, computed, inject, isNone, defineProperty, K: emberNop } = Ember;
ember-paper/addon/mixins/events-mixin.js:const { Mixin, K } = Ember;
ember-redux/app/services/redux.js:const { assert, isArray, K } = Ember;
ember-responsive/blueprints/ember-responsive/files/tests/helpers/responsive.js:const { K, getOwner } = Ember;
ember-select-box/addon/mixins/select-box/select-box/inputtable.js:const { K } = Ember;
ember-shepherd/addon/services/tour.js:const { Evented, K, Service, isPresent, run, $, isEmpty, observer } = Ember;
ember-simple-auth/addon/session-stores/cookie.js:const { RSVP, computed, run: { next, cancel, later, scheduleOnce }, isEmpty, typeOf, testing, isBlank, isPresent, K, A } = Ember;
ember-simple-auth/tests/unit/internal-session-test.js:const { RSVP, K, run: { next } } = Ember;
ember-simple-auth/tests/unit/session-stores/shared/store-behavior.js:const { run: { next }, K } = Ember;
ember-simple-auth-chrome-app/tests/unit/session-stores/shared/store-behavior.js:const { K, run: { next } } = Ember;
ember-sinon-qunit/tests/helpers/assert-sinon-in-test-context.js:const { K: EmptyFunc, typeOf } = Ember;
ember-user-activity/tests/unit/services/user-activity-test.js:const { A: emberArray, K: noOp, typeOf } = Ember;
ui-bootstrap/tests/helpers/flash-message.js:const { K } = Ember;
yes-or-no/tests/helpers/responsive.js:const { K, getOwner } = Ember;
```
