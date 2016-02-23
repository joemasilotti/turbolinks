# Turbolinks

**Turbolinks makes navigating your web application faster.** When you follow a link, Turbolinks updates the browser's history, fetches the page, swaps in its `<body>`, and merges its `<head>`, all without incurring the cost of a full page load. You get the performance benefits of a single-page application without the added complexity of a client-side JavaScript framework.

![Turbolinks](https://s3.amazonaws.com/turbolinks-docs/images/turbolinks.gif)

## Features

- **Optimizes navigation automatically.** No need to annotate links or specify which parts of the page should change.
- **No server-side cooperation necessary.** Respond with full HTML pages, not partial page fragments or JSON.
- **Respects the web.** The Back and Reload buttons work just as you’d expect. Search engine-friendly by design.
- **Supports mobile apps.** Adapters for iOS and Android let you build hybrid applications using native navigation controls.

## Supported Browsers

Turbolinks works in all modern desktop and mobile browsers. It depends on the [HTML5 History API](http://caniuse.com/#search=pushState) and [Window.requestAnimationFrame](http://caniuse.com/#search=requestAnimationFrame). In unsupported browsers, Turbolinks gracefully degrades to standard navigation.

## Installation

Include [`dist/turbolinks.js`](dist/turbolinks.js) in your application’s JavaScript bundle.

### Rails Integration

The Turbolinks gem is packaged as a Rails engine and integrates seamlessly with the Rails asset pipeline. To install:

1. Add the `turbolinks` gem, version 5, to your Gemfile: `gem 'turbolinks', '~> 5.0.0.beta'`
2. Run `bundle install`.
3. Add `//= require turbolinks` to your JavaScript manifest file (usually found at `app/assets/javascripts/application.js`).

# Understanding Turbolinks Navigation

Turbolinks intercepts all clicks on `<a href>` links to the same domain. When you click an eligible link, Turbolinks prevents the browser from following it. Instead, Turbolinks changes the browser's URL using the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History), requests the new page using [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest), and then renders the HTML response.

During rendering, Turbolinks replaces the current `<body>` element outright and merges the contents of the `<head>` element. The JavaScript `window` and `document` objects, and the HTML `<html>` element, persist from one rendering to the next.

## Each Navigation is a Visit

Turbolinks models navigation as a *visit* to a *location* (URL) with an *action*.

Visits represent the entire navigation lifecycle from click to render. That includes changing browser history, issuing the network request, restoring a copy of the page from cache, rendering the final response, and updating the scroll position.

There are two types of visit: an _application visit_, which has an action of _advance_ or _replace_, and a _restoration visit_, which has an action of _restore_.

## Application Visits

Application visits are initiated by clicking a Turbolinks-enabled link, or programmatically by calling `Turbolinks.visit(location)`.

An application visit always issues a network request. When the response arrives, Turbolinks renders its HTML and completes the visit.

If possible, Turbolinks will render a preview of the page from cache immediately after the visit starts. This improves the perceived speed of frequent navigation between the same pages.

If the visit's location includes an anchor, Turbolinks will attempt to scroll to the anchored element. Otherwise, it will scroll to the top of the page.

Application visits result in a change to the browser's history; the visit's _action_ determines how.

![Advance visit action](https://s3.amazonaws.com/turbolinks-docs/images/advance.svg)

The default visit action is _advance_. During an advance visit, Turbolinks pushes a new entry onto the browser’s history stack using [`history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState).

Applications using the Turbolinks iOS adapter typically handle advance visits by pushing a new view controller onto the navigation stack. Similarly, applications using the Android adapter typically push a new activity onto the back stack.

![Replace visit action](https://s3.amazonaws.com/turbolinks-docs/images/replace.svg)

You may wish to visit a location without pushing a new history entry onto the stack. The _replace_ visit action uses [`history.replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) to discard the topmost history entry and replace it with the new location.

To specify that following a link should trigger a replace visit, annotate the link with `data-turbolinks-action=replace`:

```html
<a href="/edit" data-turbolinks-action=replace>Edit</a>
```

To programmatically visit a location with the replace action, pass the `action: "replace"` option to `Turbolinks.visit`:

```js
Turbolinks.visit("/edit", { action: "replace" })
```

Applications using the Turbolinks iOS adapter typically handle replace visits by dismissing the topmost view controller and pushing a new view controller onto the navigation stack without animation.

## Restoration Visits

Turbolinks automatically initiates a restoration visit when you navigate with the browser's Back or Forward buttons. Applications using the iOS or Andriod adapters initiate a restoration visit when moving backward in the navigation stack.

![Restore visit action](https://s3.amazonaws.com/turbolinks-docs/images/restore.svg)

If possible, Turbolinks will render a copy of the page from cache without making a request. Otherwise, it will retrieve a fresh copy of the page over the network.

Turbolinks saves the scroll position of each page before navigating away and automatically returns to this saved position on restoration visits.

Restoration visits have an action of _restore_ and Turbolinks reserves them for internal use. You should not attempt to annotate links or invoke `Turbolinks.visit` with an action of `restore`.

## Canceling Visits Before They Start

Application visits can be canceled before they start, regardless of whether they were initiated by a link click or a call to `Turbolinks.visit`.

Listen for the `turbolinks:before-visit` event to be notified when a visit is about to start, and use `event.data.url` (or `$event.originalEvent.data.url`, when using jQuery) to check the visit's location. Then cancel the visit by calling `event.preventDefault()`.

Restoration visits cannot be canceled and do not fire `turbolinks:before-visit`. Turbolinks issues restoration visits in response to history navigation that has *already taken place*, typically via the browser’s Back or Forward buttons.

## Disabling Turbolinks on Specific Links

Turbolinks can be disabled on a per-link basis by annotating a link or any of its ancestors with `data-turbolinks=false`.

```html
<a href="/" data-turbolinks=false>Disabled</a>

<div data-turbolinks=false>
  <a href="/">Disabled</a>
</div>
```

To reenable when an ancestor has opted out, use `data-turbolinks=true`:

```html
<div data-turbolinks=false>
  <a href="/" data-turbolinks=true>Enabled</a>
</div>
```

Links with Turbolinks disabled will be handled normally by the browser.

# Building Your Turbolinks Application

Turbolinks is fast because it doesn't reload the page when you follow a link. Instead, your application becomes a persistent, long-running process in the browser. This requires you to rethink the way you structure your JavaScript.

In particular, you can no longer depend on a full page load to reset your environment every time you navigate. The JavaScript `window` and `document` objects retain their state across page changes, and any other objects you leave in memory will stay in memory.

With awareness and a little extra care, you can design your application to gracefully handle this constraint without tightly coupling it to Turbolinks.

## Running JavaScript When a Page Loads

You may be used to installing JavaScript behavior in response to the `window.onload`, `DOMContentLoaded`, or jQuery `ready` events. With Turbolinks, these events will fire only in response to the initial page load—not after any subsequent page changes.

In many cases, you can simply adjust your code to listen for the `turbolinks:load` event instead, which fires once on the initial page load, and again after every Turbolinks visit.

```js
document.addEventListener("turbolinks:load", function() {
  // ...
})
```

When possible, use event delegation to ...

## Understanding Caching

Turbolinks maintains a cache of recently visited pages. This cache serves two purposes: to display pages without accessing the network during restoration visits, and to improve perceived performance by showing temporary previews during application visits.

When navigating by history (via a restoration visit), Turbolinks will restore the page from cache without loading a fresh copy from the network, if possible.

Otherwise, during standard navigation (via an application visit), Turbolinks will immediately restore the page from cache and display it as a preview while simultaneously loading a fresh copy from the network. This gives the illusion of instantaneous page loads for frequently accessed locations.

Turbolinks saves a copy of the current page to its cache immediately before rendering a new page. Note that Turbolinks copies the page using [`cloneNode(true)`](https://developer.mozilla.org/en-US/docs/Web/API/Node/cloneNode), which means any attached event listeners and associated data are discarded.

Listen for the `turbolinks:before-cache` event if you need to prepare the document before Turbolinks caches it. You can use this event to reset forms, collapse expanded UI elements, or tear down any third-party widgets so the page is ready to be displayed again.

```js
document.addEventListener("turbolinks:before-cache", function() {
  // ...
})
```

## Making Transformations Idempotent

- If your page is being rendered from cache, transformations may have already been applied
- On the next turbolinks:load event, you want to avoid processing elements twice
- Keep state in the DOM (e.g. using data attributes) when elements have been processed, and check for that in your processing function

## Responding to Page Updates

Turbolinks may not be the only source of page updates. New HTML can appear at any time from Ajax requests, WebSocket connections, or other client-side rendering operations, and this content will need to be initialized as if it came from a fresh page load.

- Use MutationObserver or custom elements for precise lifecycle callbacks
- Perform transformations and attach behavior when elements are added to the page
- Remove behavior when elements are removed from the page
- Benefit: your application is decoupled from Turbolinks


## Designating Permanent Elements

Consider a Turbolinks application with a shopping cart. At the top of each page is an icon with the number of items currently in the cart. This counter is updated dynamically with JavaScript as items are added and removed.

Now imagine a user who has navigated to several pages in this application. She adds an item to her cart, then presses the Back button in her browser. Upon navigation, Turbolinks restores the previous page’s state from cache, and the cart item count erroneously changes from 1 to 0.

To avoid this problem, Turbolinks allows you to mark certain elements as _permanent_. Permanent elements persist across page loads, so that any changes you make to those elements do not need to be reapplied after navigation.

Designate permanent elements by giving them an HTML `id` and annotating them with `data-turbolinks-permanent`. Before each render, Turbolinks matches all permanent elements by `id` and transfers them from the original page to the new page, preserving their data and event listeners.

## Handling Dynamic Updates

**TODO**

- Prefer using event delegation on `document.documentElement`, `document`, or `window`.
- Consider using `MutationObserver` to install behavior on elements as they’re added to the page.


# Advanced Usage

## Displaying Progress

During Turbolinks navigation, the browser will not display its native progress indicator. Turbolinks installs a CSS-based progress bar to provide feedback while issuing a request.

The progress bar is enabled by default. It appears automatically for any page that takes longer than 500ms to load.

The progress bar is a `<div>` element with the class name `turbolinks-progress-bar`. Its default styles appear first in the document and can be overridden by rules that come later.

For example, the following CSS will result in a thick green progress bar:

```css
.turbolinks-progress-bar {
  height: 5px;
  background-color: green;
}
```


## Reloading When Assets Change

Turbolinks can track the URLs of asset elements in `<head>` from one page to the next, and automatically issue a full reload if they change. This ensures that users always have the latest versions of your application’s scripts and styles.

Annotate asset elements with `data-turbolinks-track=reload` and include a version identifier in your asset URLs. The identifier could be a number, a last-modified timestamp, or better, a digest of the asset's contents as in the following example.

```html
<head>
  ...
  <link rel="stylesheet" href="/application-258e88d.css" data-turbolinks-track=reload>
  <script src="/application-cbd3cd4.js" data-turbolinks-track=reload></script>
</head>
```

Note that Turbolinks will only consider tracked assets in `<head>` and not elsewhere on the page.

## Setting a Root Location

TODO

## Following Redirects

When you visit location “/one” and the server redirects you to location “/two”, you expect the browser’s address bar to display the redirected URL.

However, Turbolinks makes requests using `XMLHttpRequest`, which transparently follows redirects. There’s no way for Turbolinks to tell whether a request resulted in a redirect without additional cooperation from the server.

To work around this problem, send the `Turbolinks-Location` header in response to a visit that was redirected, and Turbolinks will replace the browser’s topmost history entry with the value you provide.

The Turbolinks Rails engine sets `Turbolinks-Location` automatically when using `redirect_to` in response to a Turbolinks visit.

## Redirecting After a Form Submission

Submitting an HTML form to the server and redirecting in response is a common pattern in web applications. Standard form submission is similar to navigation, resulting in a full page load. Using Turbolinks you can improve the performance of form submission without complicating your server-side code.

Instead of submitting forms normally, submit them with XHR. In response to an XHR submit on the server, return JavaScript that performs a `Turbolinks.visit` to be evaluated by the browser.

If form submission results in a state change on the server that affects cached pages, consider clearing Turbolinks’ cache with `Turbolinks.clearCache()`.

The Turbolinks Rails engine performs this optimization automatically for non-GET XHR requests that redirect with the `redirect_to` helper.

## Full List of Events

Turbolinks emits events that allow you to track the navigation lifecycle and respond to page loading. Except where noted, Turbolinks fires events on the `document` object.

- `turbolinks:click` fires when you click a Turbolinks-enabled link. The clicked element is the event target. Access the requested location with `event.data.url`. Cancel this event to let the click fall through to the browser as normal navigation.
- `turbolinks:before-visit` fires before visiting a location, except when navigating by history. Access the requested location with `event.data.url`. Cancel this event to prevent navigation.
- `turbolinks:visit` fires immediately after a visit starts.
- `turbolinks:request-start` fires before Turbolinks issues a network request to fetch the page.
- `turbolinks:request-end` fires after the network request completes.
- `turbolinks:before-cache` fires before Turbolinks saves the current page to cache.
- `turbolinks:before-render` fires before rendering the page. Access the new `<body>` element with `event.data.newBody`.
- `turbolinks:render` fires after Turbolinks renders the page. This event fires twice during an application visit to a cached location: once after rendering the cached version, and again after rendering the fresh version.
- `turbolinks:load` fires once after the initial page load, and again after every Turbolinks visit. Access visit timing metrics with the `event.data.timing` object.



---

# Contributing to Turbolinks

Turbolinks is open-source software, freely distributable under the terms of an [MIT-style license](LICENSE). The [source code is hosted on GitHub](https://github.com/turbolinks/turbolinks).
Development is sponsored by [Basecamp](https://basecamp.com/).

We welcome contributions in the form of bug reports, pull requests, or thoughtful discussions in the [GitHub issue tracker](https://github.com/turbolinks/turbolinks/issues).

## Building From Source

Turbolinks is written in [CoffeeScript](https://github.com/jashkenas/coffee-script) and compiled to JavaScript with [Blade](https://github.com/javan/blade). To build from source you’ll need a recent version of Ruby. From the root of your Turbolinks directory, issue the following commands to build the distributable files in `dist/`:

```
$ gem install bundler
$ bundle install
$ bin/blade build
```

## Running Tests

Follow the instructions for _Building From Source_ above. Then run `bin/blade runner` and visit the displayed URL in your browser. The Turbolinks test suite will start automatically.
