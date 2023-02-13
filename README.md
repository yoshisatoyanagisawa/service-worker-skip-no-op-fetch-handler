# Explainer: Skip service worker no-op fetch handler

## Authors

* Yoshisato Yanagisawa (yyanagisawa@chromium.org)

## Participate

* https://github.com/yoshisatoyanagisawa/service-worker-skip-no-op-fetch-handler/issues

## Background

[ServiceWorker](https://www.w3.org/TR/service-workers/) is a web platform feature that brings application-like experience to users.
It provides push notification or offline support for example.  Especially, people use a fetch handler to intercept the navigation to
provide caching, offline capabilities, etc.

## A problem and a solution

Some sites have a no-op (no operation) fetch handler (e.g. `onfetch = () => {}`).  Since having the fetch handler was one of the
requirements to be [a progressive web app (PWA)](https://developer.mozilla.org/docs/Web/Progressive_web_apps), we assume that they did that
to make their site recognized as PWA.  However, it only brings overheads to start a service worker and execute a no-op handler without
bringing any feature benefits like caching or offline capabilities because the code does nothing.  To make the navigation to such pages
faster, we would like to omit the service worker start and the handler execution from the navigation critical path if a user agent
identifies that the service worker fetch handler is no-op.

### What are the fetch handler overheads in Chromium?

For the security reason, Chromium runs a service worker in an isolated process, and kills it after a certain timeout.  It is needed to
minimize a possible threat caused by the service worker.

To execute a service worker fetch handler, Chromium should bootstrap a sandboxed JavaScript runtime if there is no other runtime running
for the service worker.  Then, Chromium loads the service worker script, and runs the no-op function.  Since the fetch handler intercepts
the navigation, it resides in the navigation critical path.

## A proposal

A user agent scans a service worker script before the navigation to the page within the service worker scope.  Note that the scan timing
is discussed in the section “[Approaches to deal with the handler updates after the initialization](
#approaches-to-deal-with-the-handler-updates-after-the-initialization)” below.  If the user agent identifies the fetch handler as no-op,
it may behave as if the service worker did not have a fetch handler during the navigation. i.e. not starting service workers, and not
executing the fetch handler at that time.  Then, the user agent eventually starts the service worker asynchronously to preserve
correctness.

The user agent does static analysis to the fetch handler to identify it is a no-op.  That usually happens on the service worker
initialization, and it decides skip/not-skip decision at this time.  If the handler is set or updated outside of the initialization, the
user agent may change skip/not-skip decisions based on the new handler or updates.  It means that the users should not rely on that fetch
handler changes/additions after the initialization are reflected in the user-agent’s skip/not-skip decision.

### Approaches to deal with the handler updates after the initialization

In the above section, we said that the user agent **may** change skip/not-skip decisions based on the added/updated fetch handlers.
This section talks about the possible options the user agent can take, and pros/cons.

#### Option 1: No updates are allowed after the initialization

This option simply ignores any updates to the fetch handler after the initialization.  The service worker specification allows skip event
dispatch if the event handler is not registered in the initialization (see [Should Skip Event](
https://w3c.github.io/ServiceWorker/#should-skip-event)).  This option extends that to include no-op fetch handlers.

**Pros**
*   Natural expansion of the limitation.  Currently, the service worker does not recommend developers to add handlers after the
    initialization.  It expands the recommendation to a no-op handler case.
*   The user agent code change should be the service worker initialization, and no need to add implementation elsewhere.

**Cons**
*   Breaks service worker code that sets the no-op fetch handler at the initialization and updates it later.
*   Breaks service worker code that occasionally sets no-op fetch handlers.  e.g. doing A/B testing.

#### Option 2: Updates are checked every time when the service worker starts

This option allows the update every time the service worker starts.  The option slightly relaxes the limitation compared to Option 1.
Unlike Option 1, code like `onfetch = Math.round(Math.random()) ? ()=>{} : ()=>console.log("hello")` can move from one to the other because the
service worker starts asynchronously.

**Pros**
*   Natural expansion of the limitation.  Currently, the service worker does not recommend developers to add handlers after the
    initialization.  It expands the recommendation to a no-op handler case.
*   Support service worker code that occasionally sets no-op fetch handlers.  e.g. doing A/B testing.
*   The user agent code change should go to code to start workers.  It still keeps locality on implementation.

**Cons**
*   Breaks service worker code that sets the no-op fetch handler at the initialization and updates it later.

#### Option 3: Updates are checked every time when the event handler changes

This option allows the update every time the fetch handler updates.  It means that the user agent watches the update to the fetch handler, and
analyzes the fetch handler on every fetch handler update.

**Pros**
*   Support service worker code that sets the no-op fetch handler at the initialization and updates it later.
*   Support service worker code that occasionally sets no-op fetch handlers.  e.g. doing A/B testing.

**Cons**
*   The update to the code can be large.  Implementation should go not only initialization but also the event handler update.  Especially
    for Chromium, the update happens in the renderer process and the fetch handler dispatch happens in the browser process.  There should
    be a new IPC to make the update notified from the renderer process to the browser process, which was not needed for the current
    specification because the user agent does not need to dispatch even if the handler did not exist at the initialization time.

Considering the implementation complexity, Chromium is currently exploring Option 2.

## FAQ

### Will this ever cause a behavior change?

Yes.  Assume that a service worker script is like:

```js
fetch("https://example.com/beacon");
onfetch = () => {};
```

Without the proposed change, example.com/beacon fetch likely starts before the 'controlled' fetch (as in, a fetch that should be handled by
this service worker) completes, assuming the service worker was in a terminated state before the controlled fetch.  With the proposed
change and after a user agent detects the fetch handler as no-op, the beacon fetch usually begins after controlled fetch.  The web
developers should not expect the script to be executed before the navigation.  Moreover, the fetch handler can be skipped, and developers
may see that in a debugger.

Assume, you write a following service worker script:

```js
onfetch = () => {};

setTimeout(() => {
  onfetch = (event) => {
    event.respondWith(new Response('hello'));
  }
}, 1)
```

It sets a no-op fetch handler, and updates it with a fetch handler that responds “hello” with non-zero timeout.  Without the proposed
change, the fetch handler that responds with “hello” will be used and the whole page is replaced with “hello”.  However, with the proposed
change, the fetch handler is recognized as no-op at the initialization time, and the user agent may decide to skip executing it.  In that
case, the original page is shown as-is instead of showing “hello” after navigation because the user agent skips the fetch handler.

### How does it work with Navigation Preload?

With [navigation preload](https://web.dev/navigation-preload/), a user agent sends a network request while starting the service worker in
parallel, and executes the service worker fetch handler after network fetch ends.  If the fetch handler does not execute `respondWith`, it
brings network fallback and another network fetch will be done. i.e. if the fetch handler is no-op, it obviously proceeds the network fetch
twice on navigation preload.

You may wonder if a user agent proceeds a network fallback if no-op fetch handler is skipped by the proposal.  There can be three options:

#### Option A: disable the proposal for navigation preload

To avoid unexpected behavior that happens with a mixture of the proposal and the navigation preload, this option disables the proposal if
the navigation preload is set.

**Pros**
*   A specification and an implementation become simple because the proposed feature is just disabled in this situation.

**Cons**
*   Unnecessary service worker fetch handler execution and network fetch still exist in the critical navigation path.

#### Option B: just skip a fetch handler and proceed network fetch twice

This proposal just skips executing fetch handlers while still proceeding network fallback.

**Pros**
*   A straightforward design and implementation of the world with the proposal.

**Cons**
*   Unnecessary network fetch still exists in the critical navigation path.

#### Option C: skip both a fetch handler execution and fallback network fetch

This proposal skips not only executing a no-op fetch handler but also proceeding the network fallback fetch.

**Pros**
*   Negative effect on the navigation critical path is the smallest.

**Cons**
*   The behavior on navigation preload changes a lot because this option does not run the second network fetch.

Considering the purpose of the proposal, Chromium is currently exploring Option C.

## Appendix: other approach considered to deal with the fetch handler update

#### Option 1b: No updates are allowed after the initialization, but with robust static analysis.

For service workers with a fetch handler, the following **static** analysis is used to determine "can the fetch dispatch race the
underlying fetch?"

1. If global importScripts is referenced, return false.
2. If global addEventListener is called with a first argument that cannot be determined to be a static string, return false.
3. If global eval is referenced, return false.
4. If a "with" statement is used, return false.
5. If a property is set on the global in a way that cannot be determined to be a static string, return false. (eg self[getValue()] = …)
6. If global onfetch is referenced outside the initial block, return false.
7. If global addEventListener('fetch', …) is called outside the initial block, return false.
8. If the second argument of addEventListener('fetch', …), or the right-hand side of onfetch=, cannot be determined to be a function with
   an empty body, return false.
9. If in doubt, return false.
10. Return true.

**Pros**
*   Avoids (all?) false positives.

**Cons**
*   False negatives?
*   Implementation complexity?
