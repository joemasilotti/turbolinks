# Turbolinks

**Turbolinks makes navigating your web application faster.** In standard browser navigation every page is loaded anew. Resources are downloaded, JavaScript is evaluated, and CSS is processed. This takes time. But in most web applications these resources don't change between requests. So why spend time reloading them? Turbolinks speeds up navigation by persisting the current page and updating its contents in place.

With Turbolinks you get the performance benefits of a single-page application without the added complexity of a client-side JavaScript framework. Use HTML to render your views on the server side and link to pages as usual. When you follow a link, Turbolinks automatically fetches the page, swaps in its `<body>`, and merges its `<head>`, all without incurring the cost of a full page load.

* graphic *

## Features

- * good web citizen: works with back, reload automatically *
- Optimizes navigation automatically. No need to annotate links or specify which parts of the page should change.
- No server-side cooperation necessary. Respond with full HTML pages, not fragments.
- Instant navigation with caching. Recently-visited pages are redisplayed immediately and updated when a fresh response arrives.
- Custom adapters allow for precise, fine-grained control of the navigation lifecycle.

## Supported Browsers

Turbolinks works in all modern desktop and mobile browsers. It depends on the [HTML5 History API](http://caniuse.com/#search=pushState) and [Window.requestAnimationFrame](http://caniuse.com/#search=requestAnimationFrame) and degrades gracefully in their absence. In unsupported browsers navigation proceeds normally.

## Installation for Rails Applications

1. Add the `turbolinks` gem, version 5, to your Gemfile: `gem 'turbolinks', '~> 5.0.0.beta'`
2. Run `bundle install`.
3. Add `//= require turbolinks` to your JavaScript manifest file (usually found at `app/assets/javascripts/application.js`).

## Using Turbolinks Outside of a Rails Application

Simply include [`dist/turbolinks.js`](dist/turbolinks.js) in your app's JavaScript bundle.

# Concepts

Turbolinks works by listening for clicks on `<a>` elements referencing an HTML document on the current origin. When an eligible link is clicked, Turbolinks requests its location via XHR, loads the response, merges its `<head>`, and swaps in its `<body>`. Critically, `<script>`, `<style>`, and `<link>` elements in the `<head>` are considered *permanent* and not replaced.

Turbolinks is designed to emulate standard browser behavior as closely as possible: location, history, page title, and scroll position all behave exactly as you'd expect.

## TODO: Navigating with Turbolinks

Internally, Turbolinks models navigation as a *visit* to a *location* with an *action*. Actions are named for the effect they have on history.

_(introduce graphics, split into three paragraphs)_

New navigation (e.g. clicking a link) has an action of either *advance* or *replace* and creates or updates a history entry respectively. In both cases Turbolinks will request the given location over the network. If available, a cached version will be shown immediately and updated when the response arrives.

History navigation (e.g. clicking the “back” or “forward” buttons in your browser) has an action of *restore*. If available, a cached version of the restored location will be shown immediately and *no* network request will be made to refresh it. Otherwise a request will be performed. In either case, scroll position will be restored.

## TODO: Previews and Caching

Turbolinks caches the 10 most-recently-visited pages in memory for instant display on the next visit. The current page is saved to the cache just prior to it being replaced, ensuring that changes made to the DOM after the initial load will be reflected.

_(document cloning)_

Observe the `turbolinks:before-cache` event if you need to make changes or clean up any state before the page is saved.

You can clear the page cache at any time by calling `Turbolinks.clearCache()`.

## TODO: Lifecycle of a Visit

_(timeline graphic)_

# Basic Usage

## Specifying a Navigation Action

The default action when clicking a link is `advance`. To specify that `replace` be used instead, annotate your link with `data-turbolinks-action=replace`.

```html
<a href="/edit" data-turbolinks-action=replace>Edit</a>
```

The `restore` action is reserved for internal use during history navigation.

## Navigating Programmatically

To navigate programatically call `Turbolinks.visit` with a *location* and an optional *action*. The action can be either `advance` or `replace`. The default action is `advance`.

```javascript
// Visit this location and push a new history entry
Turbolinks.visit("/new")
Turbolinks.visit("/new", { action: "advance" })

// Replace the current history entry
Turbolinks.visit("/edit", { action: "replace" })
```

## Handling Significant Events

Turbolinks emits several events on `document` that allow you to track the navigation lifecycle and respond to page loading. While an exhaustive list is included later in this document, the following events are noteworthy. You’ll use these the most frequently.

**Event: `turbolinks:load`**
**Purpose: Initialize the DOM after the page has changed**

Fired in response to `DOMContentLoaded` on the initial page load and again after every Turbolinks visit, `turbolinks:load` signals that the page has loaded and the DOM is ready. It’s an appropriate time to bind event listeners, initialize behavior, or manipulate elements. Use `turbolinks:load` in place of `DOMContentLoaded` or jQuery’s `$.ready()` which are only fired on a full page load and not after Turbolinks navigation.

**Event: `turbolinks:before-cache`**
**Purpose: Clean up the DOM before it’s saved to cache**

Fired just before a snapshot of the current page is saved to the cache, `turbolinks:before-cache` is your opportunity to prepare the DOM for storage and eventual redisplay. Use it to reset form fields, close expandable elements, undo non-idempotent DOM transformations, teardown third-party code, or preserve any required state. Note that you needn’t uninstall event listeners as they’re not copied to the cache.

**Event: `turbolinks:before-visit`**
**Purpose: Prevent Turbolinks from visiting a location**

Fired just before Turbolinks visits a location, `turbolinks:before-visit` gives you an opportunity to opt-out of  cancelable navigation before it begins. Access the proposed location via the `Event` object’s `data.url` and if desired, cancel navigation by calling `event.preventDefault()`. Note that `turbolinks:before-visit` does not fire for visits that originate from history. As history has already been changed, these can’t be prevented.


## Displaying Progress

Because Turbolinks navigation proceeds without a full load, the browser's native progress indicator won't be activated. Turbolinks ships with a JavaScript and CSS-based progress bar to compensate.

The progress bar is implemented as a `<div>` element with the class name `turbolinks-progress-bar`. Its default styles are included first in the document head such that they can be overridden by rules that come later. For example, a thick green progress bar:

```css
.turbolinks-progress-bar {
  height: 5px;
  background-color: green;
}
```

## Reloading When Assets Change

When you navigate with Turbolinks, external assets like JavaScript and CSS aren’t reloaded on each request. But let’s say you’ve deployed your application with changes to those assets – how can you ensure that Turbolinks is always using their latest versions?

Turbolinks can track asset elements in the page `<head>` and reload automatically when the next navigation reveals them to have changed. Denote tracked elements with `data-turbolinks-track=reload` and include some value in the asset’s URL to indicate its revision. This could be a version number, a last-modified timestamp, or more commonly, a digest of the asset’s contents, as in the following example.

```html
<head>
  ...
  <link rel="stylesheet" href="/application-258e88d.css" data-turbolinks-track=reload>
  <script src="/application-cbd3cd4.js" data-turbolinks-track=reload></script>
</head>
```

When Turbolinks attempts to load a page whose tracked asset elements differ from those of the current page, it ceases further processing and loads the page in full. Note that when this occurs the page will be requested twice: once when it’s determined that tracked assets have changed, and again when it’s loaded in full.

## Opting Out

Turbolinks is automatically enabled for internal links to HTML documents on the same origin. You can opt out of Turbolinks explicitly by annotating a link or any of its ancestors with `data-turbolinks=false`. To reenable when an ancestor has opted out, use `data-turbolinks=true`.

```html
<a href="/">Enabled</a>
<a href="/" data-turbolinks=false>Disabled</a>

<div data-turbolinks=false>
  <a href="/">Disabled</a>
  <a href="/" data-turbolinks=true>Enabled</a>
</div>
```

# TODO: Building Your Turbolinks Application

_(discuss SJR in the context of updating the page; contrast with normal navigation)_

## TODO: Observing Page Loads

- Listen for `turbolinks:load`
- `turbolinks:load` fires once on the initial page load in response to DOMContentLoaded, and again on every Turbolinks visit, whether it’s triggered by history, a link click, or a call to `Turbolinks.visit()`.
- Keep track of elements you've already processed by adding a data attribute
- DOM transformations should be idempotent. It should be safe to apply them at any time.

## TODO: Handling Dynamic Updates

Prefer using event delegation on `document.documentElement`, `document`, or `window`. Consider using `MutationObserver` to install behavior on elements as they’re added to the page.

## Previews, Caching, and Clone Safety

Before rendering a new page, Turbolinks clones the current page’s `<body>` and saves it to the snapshot cache. Whenever Turbolinks displays a cached page—either by a restore visit using the Back or Forward buttons, or by showing a preview during an advance visit to an already-visited location—all elements are freshly cloned, which means they have no attached event listeners or associated data.

The benefits of this approach are that it’s simpler to reason about when to register event listeners (no need to distinguish between page “change” and page “load”), and that Turbolinks is less likely to leak memory (because existing event listeners are discarded).

The constraint with this approach is that all DOM manipulation must be idempotent. If you transform the document with JavaScript, you must make sure it’s safe to perform that transformation again, particularly on a cloned copy of the element. In practice, this usually means using a data attribute or some other heuristic to detect when an element has already been processed.

## Following Redirects

Turbolinks makes requests using `XMLHttpRequest`, and XHR transparently follows redirects. If you visit location A and it redirects to location B, we want B to be reflected in history and the address bar. To make this work requires cooperation from the server. There's no way to tell whether an XHR request was redirected via JavaScript alone.

Turbolinks will look for the `Turbolinks-Location` header in response to a visit and use its value to update history and the address bar. Send this header from the server when responding with a page that was arrived at by redirection, and whose location you want reflected.

Consider the following Turbolinks visit and abbreviated HTTP conversation.

```
Turbolinks.visit("/one")

> GET /one
< 302 Moved Temporarily
< Location: http://localhost/two

> GET /two # XHR follows the redirect
< 200 OK
< Turbolinks-Location: http://localhost/two

window.location.pathname # => "/two"
```

We visit “/one” and are redirected to “/two”, which XHR dutifully follows. The response from “/two” includes a  `Turbolinks-Location` header to inform Turbolinks of the location change. If the header were omitted, `window.location.pathname` would still be “/one”.

If you're using Turbolinks with a Rails application `Turbolinks-Location` is set automatically when using `redirect_to` in response to a Turbolinks visit. Other frameworks are encouraged to provide similar integration.

## Redirecting After a Form is Submitted

Submitting an HTML form to the server and redirecting in response is a common pattern in web applications. Standard form submission is similar to navigation, resulting in a full page load. Using Turbolinks you can improve the performance of form submission without complicating your server-side code.

Instead of submitting forms normally, submit them with XHR. In response to an XHR submit on the server, return JavaScript that performs a Turbolinks visit to be evaluated by the browser.

```javascript
Turbolinks.visit(destination)
```

If form submission has resulted in a state change on the server that will affect cached pages, consider clearing Turbolinks’ cache with `Turbolinks.clearCache()`.

If you're using Turbolinks with a Rails application this optimization will happen automatically for non-GET XHR requests that redirect using `redirect_to`.


# Advanced Usage

## Designating Permanent Elements

Consider a Turbolinks application with a shopping cart. At the top of each page is an icon with the number of items currently in the cart. This counter is updated dynamically with JavaScript as items are added and removed.

Now imagine a user who has navigated to several pages in this application. She adds an item to her cart, then presses the Back button in her browser. Upon navigation, Turbolinks restores the previous page’s state from cache, and the cart item count erroneously changes from 1 to 0.

To avoid this problem, Turbolinks allows you to mark certain elements as _permanent_. Permanent elements persist across page loads, so that any changes made to those elements do not need to be reapplied after navigation.

Designate permanent elements by giving them an HTML `id` and annotating them with `data-turbolinks-permanent`. Before each render, Turbolinks matches all permanent elements by `id` and transfers them from the original page to the new page, preserving their data and event listeners.

## TODO: Setting a Root Location

## Full List of Events

Turbolinks emits events that allow you to track the navigation lifecycle and respond to page loading. Except where noted, events are fired on `document`.

- `turbolinks:click` fires when a Turbolinks-enabled link is clicked. The clicked element is the event target. Access the requested location with `event.data.url`. Cancelable.
- `turbolinks:visit` fires before visiting a location. Does not fire when navigating by history. Access the requested location with `event.data.url`. Cancelable.
- `turbolinks:request-start` fires before issuing a network request to fetch a page.
- `turbolinks:request-end` fires after a network request completes.
- `turbolinks:before-cache` fires before the current page is saved to the cache.
- `turbolinks:before-render` fires before rendering the page.
- `turbolinks:render` fires after rendering the page. Fires twice when advancing to a cached location: once after rendering the cached version and again after rendering the fresh version.
- `turbolinks:load` fires after the page is fully loaded.

# TODO: Differences From Earlier Versions

# Known Issues

- Anchored links to the current location deviate from standard browser behavior by making a network request.
- Snapshot#hasAnchor() doesn't consider named anchors like `<a name="top"></a>`.
- Visiting with an unrecognized action triggers an error when history is changed. We should fail sooner when an invalid action is specified.

---

# Contributing to Turbolinks

Turbolinks is open-source software, freely distributable under the terms of an [MIT-style license](MIT-LICENSE). The source code is hosted on GitHub.
