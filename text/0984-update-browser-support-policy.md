---
stage: ready-for-release
start-date: 2023-11-02T15:40:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/984'
project-link:
suite:
---


<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Treat Safari as an Evergreen Browser

## Summary

Safari's release cadence has increased as well as relevant-device compatibilty. This RFC proposes an amendment to the [browser support](https://emberjs.com/browser-support/) policy (introduced in [RFC#685](https://rfcs.emberjs.com/id/0685-new-browser-support-policy)) to treat Safari the same as Chrome and FireFox for Desktop and Mobile devices.

## Motivation

Treating Safari differently from other browsers is no longer necessary due to changes in release cadence. Only being able to adjust Safari support with an RFC and at major versions is unnecessary overhead.

## Detailed design

Ember v6 will support Safari based on usage statistics, the same as other browsers we support.


## How we teach this

### A new browser support policy

Diff:
- remove non-evergreen section
- place Safari under Evergreen Desktop and Evergreen Mobile
- actual numbers subject to change as time passes, this is an example

```diff
{{page-title "Browser Support"}}
<div class="container layout">
  <section aria-labelledby="section-browser-support-policy">
    <h1 id="section-browser-support-policy">Ember.js Browser Support Policy</h1>


+    <h2>Ember 6.0.0</h2>
+
+    <p>
+      In Ember 6.0.0, the framework supports the following major browsers:
+    </p>
+
+    <ul class="list-unstyled layout my-3">
+      <EsCard class="lg:col-2">
+        <div class="text-center text-md">Desktop</div>
+        <ul>
+          <li>Google Chrome >= 103</li>
+          <li>Mozilla Firefox >= 102</li>
+          <li>Microsoft Edge >= 110</li>
+          <li>Safari >= 16.4</li>
+        </ul>
+      </EsCard>
+      <EsCard class="lg:col-2">
+        <div class="text-center text-md">Mobile</div>
+        <ul>
+          <li>Google Chrome >= 112</li>
+          <li>Mozilla Firefox >= 110</li>
+          <li>Safari >= 16.4</li>
+        </ul>
+      </EsCard>
+      <EsCard class="lg:col-2">
+        <div class="text-center text-md">Testing</div>
+        <ul>
+          <li>Google Chrome</li>
+          <li>Mozilla Firefox</li>
+        </ul>
+      </EsCard>
+    </ul>

    <h2>Ember 5.0.0</h2>

    <p>
      In Ember 5.0.0, the framework supports the following major browsers:
    </p>

    <ul class="list-unstyled layout my-3">
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Desktop</div>
        <ul>
          <li>Google Chrome >= 103</li>
          <li>Mozilla Firefox >= 102</li>
          <li>Microsoft Edge >= 110</li>
          <li>Safari >= 12</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Mobile</div>
        <ul>
          <li>Google Chrome >= 112</li>
          <li>Mozilla Firefox >= 110</li>
          <li>Safari >= 12</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Testing</div>
        <ul>
          <li>Google Chrome</li>
          <li>Mozilla Firefox</li>
        </ul>
      </EsCard>
    </ul>

    <h2>Ember 4.0.0</h2>

    <p>
      In Ember 4.0.0, the framework supports the following major browsers:
    </p>

    <ul class="list-unstyled layout my-3">
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Desktop</div>
        <ul>
          <li>Google Chrome >= 92</li>
          <li>Mozilla Firefox >= 91</li>
          <li>Microsoft Edge >= 93</li>
          <li>Safari >= 12</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Mobile</div>
        <ul>
          <li>Chrome Android >= 96</li>
          <li>Firefox Android >= 94</li>
          <li>Safari >= 12</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Testing</div>
        <ul>
          <li>Google Chrome</li>
          <li>Mozilla Firefox</li>
        </ul>
      </EsCard>
    </ul>

    <h2>Ember 3.0.0</h2>

    <p>
      Ember currently targets Internet Explorer 11 as a baseline for support. This means that generally all modern and relatively recent browsers will work with Ember, since browsers are backwards compatible by design. Ember runs tests against the latest desktop versions of the following browsers:
    </p>

    <div class="layout my-3">
      <div class="card lg:col-2 lg:start-3">
        <div class="card__content">
          <ul>
            <li>Google Chrome</li>
            <li>Mozilla Firefox</li>
            <li>Microsoft Edge</li>
            <li>Internet Explorer 11</li>
            <li>Safari</li>
          </ul>
        </div>
      </div>
    </div>

    <p>
      Other browsers may work with Ember.js, but are not explicitly supported. If you
      would like to add support for a new browser, please <a href="https://github.com/emberjs/rfcs">submit an RFC or RFC issue for discussion</a>!
    </p>


    <p>
      We determine support on a browser-by-browser basis. Browsers are categorized as
      either <em>evergreen</em> or <em>non-evergreen</em>. The categorization is as follows:
    </p>

    <h3 class="text-center">Evergreen</h3>

    <ul class="list-unstyled layout my-3">
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Desktop</div>
        <ul>
          <li>Google Chrome</li>
          <li>Mozilla Firefox</li>
          <li>Microsoft Edge</li>
+         <li>Safari</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Mobile</div>
        <ul>
          <li>Google Chrome</li>
          <li>Mozilla Firefox</li>
+         <li>Safari</li>
        </ul>
      </EsCard>
      <EsCard class="lg:col-2">
        <div class="text-center text-md">Testing</div>
        <ul>
          <li>Google Chrome</li>
          <li>Mozilla Firefox</li>
        </ul>
      </EsCard>
    </ul>

-    <h3 class="text-center">Non-evergreen</h3>
-
-    <div class="layout">
-      <ul class="list-unstyled layout lg:col-4 lg:start-2 my-3">
-        <EsCard class="lg:col-3">
-          <div class="text-center text-md">Desktop</div>
-          <ul>
-            <li>Safari</li>
-          </ul>
-        </EsCard>
-        <EsCard class="lg:col-3">
-          <div class="text-center text-md">Mobile</div>
-          <ul>
-            <li>Safari</li>
-          </ul>
-        </EsCard>
-      </ul>
-    </div>

    <p>
-     For evergreen browsers, the minimum version of the browser that we support is
+     For evergreen browsers, the minimum version of the browser that we support can be
      determined at the time of every minor release, following this formula:
    </p>

    <div class="layout my-3">
      <div class="card">
        <div class="card__content">
-          <p>Whichever browser version is greater/more recent out of:</p>
+          <p>
+            Whichever browser version is greater/more recent out of the following,
+            given that the owning entity (e.g.: Apple, Google, Mozilla) still supports the version
+          </p>

          <ol>
            <li>
              The lowest/least recent version that fulfills any one of these properties:
              <ul>
                <li>It is the latest version of the browser.</li>
                <li>It is the latest LTS/extended support version of the browser (such as Firefox ESR).</li>
                <li>It has at least <em>0.25%</em> of global marketshare usage across mobile and
              desktop, based on <a href="https://gs.statcounter.com/">statcounter</a>.</li>
              </ul>
            </li>
            <li>
              The minimum version supported in the previous release
            </li>
          </ol>
        </div>
      </div>
    </div>

    <p>
      To simplify, the supported version either moves forward or stays the same for
      each release based on overall usage and LTS/current release versions.
    </p>

    <p>
      For non-evergreen browsers, support is locked at a specific major version, and
      we support all major versions above that version.
+     However, all supported browsers are considered ever green.
    </p>

-    <div class="layout">
-      <ul class="list-unstyled layout lg:col-4 lg:start-2 my-3">
-        <EsCard class="lg:col-3">
-          <div class="text-center text-md">Desktop</div>
-          <ul>
-            <li>Safari: 12</li>
-          </ul>
-        </EsCard>
-        <EsCard class="lg:col-3">
-          <div class="text-center text-md">Mobile</div>
-          <ul>
-            <li>Safari: 12</li>
-          </ul>
-        </EsCard>
-      </ul>
-    </div>

    Within a version of a browser, we only support the most recent patch release.
  </section>
</div>
```

## Drawbacks

More calculations for us to do when determining browser support for a particular version of ember.
(based on the algorithm described in 

> Whichever browser version is greater/more recent out of:

)


## Alternatives


## Unresolved questions

