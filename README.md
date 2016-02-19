# Turbolinks

**Turbolinks makes navigating your web application faster.** In standard browser navigation every page is loaded anew. Resources are downloaded, JavaScript is evaluated, and CSS is processed. This takes time. But in most web applications these resources don't change between requests. So why spend time reloading them? Turbolinks speeds up navigation by persisting the current page and updating its contents in place.

With Turbolinks you get the performance benefits of a single-page application without the added complexity of a client-side JavaScript framework. Use HTML to render your views on the server side and link to pages as usual. When you follow a link, Turbolinks automatically fetches the page, swaps in its `<body>`, and merges its `<head>`, all without incurring the cost of a full page load.

![Turbolinks](https://s3.amazonaws.com/turbolinks-docs/images/turbolinks.gif)

## Features

- _(good web citizen: works with back, reload automatically)_
- Optimizes navigation automatically. No need to annotate links or specify which parts of the page should change.
- No server-side cooperation necessary. Respond with full HTML pages, not fragments.
- Instant navigation with caching. Recently-visited pages are redisplayed immediately and updated when a fresh response arrives.
- Custom adapters allow for precise, fine-grained control of the navigation lifecycle.

## Supported Browsers

Turbolinks works in all modern desktop and mobile browsers. It depends on the [HTML5 History API](http://caniuse.com/#search=pushState) and [Window.requestAnimationFrame](http://caniuse.com/#search=requestAnimationFrame) and gracefully degrades to standard navigation in their absence.

## Installation

Simply include [`dist/turbolinks.js`](dist/turbolinks.js) in your app's JavaScript bundle.

### Rails Integration

Turbolinks features framework-level integration for Rails applications.

1. Add the `turbolinks` gem, version 5, to your Gemfile: `gem 'turbolinks', '~> 5.0.0.beta'`
2. Run `bundle install`.
3. Add `//= require turbolinks` to your JavaScript manifest file (usually found at `app/assets/javascripts/application.js`).


# Understanding Turbolinks Navigation

Turbolinks intercepts all clicks on `<a href>` links to the same domain. When an eligible link is clicked, Turbolinks prevents the browser from following it. Instead, Turbolinks changes the browser's URL using the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History), requests the new page using [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest), and then renders the HTML response.

During rendering, Turbolinks replaces the current `<body>` element outright and merges the contents of the `<head>` element. The JavaScript `window` and `document` objects, and the HTML `<html>` element, persist from one rendering to the next.

## Each Navigation is a Visit

Turbolinks models navigation as a *visit* to a *location* (URL) with an *action*.

Visits represent the entire navigation lifecycle from click to load. That includes issuing the network request, restoring a copy of the page from cache, changing browser history, rendering the final response, and updating the scroll position.

There are two types of visit: an _application visit_, which has an action of _advance_ or _replace_, and a _restoration visit_, which has an action of _restore_.

## Application Visits

Application visits can be initiated by clicking a Turbolinks-enabled link, or programmatically by calling `Turbolinks.visit(location)`.

An application visit always issues a network request. When the response arrives, Turbolinks renders its HTML and completes the visit.

If possible, Turbolinks will render a preview of the page from cache immediately after the visit starts, while the network request completes in the background. This improves the perceived speed of frequent navigation between the same pages.

If the visit's location includes an anchor, Turbolinks will attempt to scroll to the anchored element. Otherwise, it will scroll to the top of the page.

Application visits result in a change to the browser's history; the visit's _action_ determines how.

![Advance visit action](https://s3.amazonaws.com/turbolinks-docs/images/advance.svg)

The default visit action is _advance_. During an advance visit, Turbolinks pushes a new entry onto the browser’s history stack using [`history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState).

Applications using the Turbolinks iOS adapter typically handle _advance_ visits by pushing a new view controller onto the navigation stack. Similarly, applications using the Android adapter typically push a new activity onto the back stack.

![Replace visit action](https://s3.amazonaws.com/turbolinks-docs/images/replace.svg)

Sometimes you may wish to visit a location without pushing a new history entry onto the stack. The _replace_ visit action uses [`history.replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) to discard the topmost history entry and replace it with the new location.

To specify that following a link should trigger a _replace_ visit, annotate the link with `data-turbolinks-action=replace`:

```html
<a href="/edit" data-turbolinks-action=replace>Edit</a>
```

To programmatically visit a location with the _replace_ action, pass the `action: "replace"` option to `Turbolinks.visit`:

```js
Turbolinks.visit("/edit", { action: "replace" })
```

Applications using the Turbolinks iOS adapter typically handle _replace_ visits by dismissing the topmost view controller and pushing a new view controller onto the navigation stack without animation.

## Restoration Visits

Turbolinks automatically initiates a restoration visit when you navigate with the browser's Back or Forward buttons. Applications using the iOS or Andriod adapters initiate a restoration visit when moving backward in the navigation stack.

![Restore visit action](https://s3.amazonaws.com/turbolinks-docs/images/restore.svg)

If possible, Turbolinks will render a copy of the page from cache without making a request. Otherwise, it will retrieve a fresh copy of the page over the network.

Turbolinks saves the scroll position of each page before navigating away and automatically returns to this saved position on restoration visits.

Restoration visits have an action of _restore_ and are reserved for internal use. You should not attempt to annotate links or invoke `Turbolinks.visit` with an action of `restore`.

## Canceling Visits Before They Start

Application visits can be canceled before they start, regardless of whether they were initiated by a link click or a call to `Turbolinks.visit`.

Listen for the `turbolinks:before-visit` event to be notified when a visit is about to start, and use `event.data.url` (or `$event.originalEvent.data.url`, when using jQuery) to check the visit's location. Then cancel the visit by calling `event.preventDefault()`.

Restoration visits cannot be canceled and do not fire `turbolinks:before-visit`. Restoration visits are issued in response to history navigation that has *already taken place*, typically via the browser’s Back and Forward buttons, and consequently cannot be prevented.

## Disabling Turbolinks on Specific Links

Turbolinks can be disabled on a per-link basis by annotating a link or any of its ancestors with `data-turbolinks=false`. To reenable when an ancestor has opted out, use `data-turbolinks=true`.

```html
<a href="/">Enabled</a>
<a href="/" data-turbolinks=false>Disabled</a>

<div data-turbolinks=false>
  <a href="/">Disabled</a>
  <a href="/" data-turbolinks=true>Enabled</a>
</div>
```

Links with Turbolinks disabled will be handled normally by the browser, which usually means they'll result in a full page load.

# Building Your Turbolinks Application

## Observing Significant Events

Turbolinks emits several events on `document` that allow you to track the navigation lifecycle and respond to page loading. While an exhaustive list is included later in this document, the following events are noteworthy. You’ll use these the most frequently.

#### `turbolinks:load`
**Purpose: Initialize the DOM after the page has changed**

Fired in response to `DOMContentLoaded` on the initial page load and again after every Turbolinks visit, `turbolinks:load` signals that the page has loaded and the DOM is ready. It’s an appropriate time to bind event listeners, initialize behavior, or manipulate elements. Use `turbolinks:load` in place of `DOMContentLoaded` or jQuery’s `$.ready()` which are only fired on a full page load and not after Turbolinks navigation.

#### `turbolinks:before-cache`
**Purpose: Clean up the DOM before it’s saved to cache**

Fired just before a snapshot of the current page is saved to the cache, `turbolinks:before-cache` is your opportunity to prepare the DOM for storage and eventual redisplay. Use it to reset form fields, close expandable elements, undo non-idempotent DOM transformations, teardown third-party code, or preserve any required state. Note that you needn’t uninstall event listeners as they’re not copied to the cache.

#### `turbolinks:before-visit`
**Purpose: Prevent Turbolinks from visiting a location**

Fired just before Turbolinks visits a location, `turbolinks:before-visit` gives you an opportunity to opt-out of  cancelable navigation before it begins. Access the proposed location via the `Event` object’s `data.url` and if desired, cancel navigation by calling `event.preventDefault()`. Note that `turbolinks:before-visit` does not fire for visits that originate from history. As history has already been changed, these can’t be prevented.

## Previews, Caching, and Clone Safety

Before rendering a new page, Turbolinks clones the current page’s `<body>` and saves it to the snapshot cache. Whenever Turbolinks displays a cached page—either by a restore visit using the Back or Forward buttons, or by showing a preview during an advance visit to an already-visited location—all elements are freshly cloned, which means they have no attached event listeners or associated data.

The benefits of this approach are that it’s simpler to reason about when to register event listeners (no need to distinguish between page “change” and page “load”), and that Turbolinks is less likely to leak memory (because existing event listeners are discarded).

The constraint with this approach is that all DOM manipulation must be idempotent. If you transform the document with JavaScript, you must make sure it’s safe to perform that transformation again, particularly on a cloned copy of the element. In practice, this usually means using a data attribute or some other heuristic to detect when an element has already been processed.

## Designating Permanent Elements

Consider a Turbolinks application with a shopping cart. At the top of each page is an icon with the number of items currently in the cart. This counter is updated dynamically with JavaScript as items are added and removed.

Now imagine a user who has navigated to several pages in this application. She adds an item to her cart, then presses the Back button in her browser. Upon navigation, Turbolinks restores the previous page’s state from cache, and the cart item count erroneously changes from 1 to 0.

To avoid this problem, Turbolinks allows you to mark certain elements as _permanent_. Permanent elements persist across page loads, so that any changes made to those elements do not need to be reapplied after navigation.

Designate permanent elements by giving them an HTML `id` and annotating them with `data-turbolinks-permanent`. Before each render, Turbolinks matches all permanent elements by `id` and transfers them from the original page to the new page, preserving their data and event listeners.

## Handling Dynamic Updates

Prefer using event delegation on `document.documentElement`, `document`, or `window`. Consider using `MutationObserver` to install behavior on elements as they’re added to the page.


# Advanced Usage

## Displaying Progress

Because a Turbolinks visit doesn’t issue a full page load, the browser's native progress indicator won't be activated when you navigate. To compensate, Turbolinks includes a JavaScript and CSS-based progress bar that’s activated automatically.

The progress bar is implemented as a `<div>` element with the class name `turbolinks-progress-bar`. Its default styles are included first in the document; they can be overridden by rules that come later.

For example, the following will result in a thick green progress bar:

```css
.turbolinks-progress-bar {
  height: 5px;
  background-color: green;
}
```

## Reloading When Assets Change

When you navigate with Turbolinks, external assets like JavaScript and CSS aren’t reloaded. But let’s say you’ve deployed your application with changes to those assets. How can you ensure Turbolinks is always using their latest versions?

Turbolinks can track asset elements in the page `<head>` and reload automatically when the next navigation reveals them to have changed.

Denote tracked elements with `data-turbolinks-track=reload` and include some value in the asset’s URL to indicate its revision. This could be a version number, a last-modified timestamp, or more commonly, a digest of the asset’s contents, as in the following example.

```html
<head>
  ...
  <link rel="stylesheet" href="/application-258e88d.css" data-turbolinks-track=reload>
  <script src="/application-cbd3cd4.js" data-turbolinks-track=reload></script>
</head>
```

When Turbolinks attempts to load a page whose tracked asset elements differ from those of the current page, it ceases further processing and loads the page in full.

Note that when this occurs the page will be requested twice: once when it’s determined that tracked assets have changed, and again when it’s loaded in full.

## Setting a Root Location

TODO

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

## Full List of Events

Turbolinks emits events that allow you to track the navigation lifecycle and respond to page loading. Except where noted, events are fired on `document`.

- `turbolinks:click` fires when a Turbolinks-enabled link is clicked. The clicked element is the event target. Access the requested location with `event.data.url`. Cancelable.
- `turbolinks:before-visit` fires before visiting a location. Does not fire when navigating by history. Access the requested location with `event.data.url`. Cancelable.
- `turbolinks:visit` fires immediately after a visit starts.
- `turbolinks:request-start` fires before issuing a network request to fetch a page.
- `turbolinks:request-end` fires after a network request completes.
- `turbolinks:before-cache` fires before the current page is saved to the cache.
- `turbolinks:before-render` fires before rendering the page.
- `turbolinks:render` fires after rendering the page. Fires twice when advancing to a cached location: once after rendering the cached version and again after rendering the fresh version.
- `turbolinks:load` fires after the page is fully loaded.



---

# Contributing to Turbolinks

Turbolinks is open-source software, freely distributable under the terms of an [MIT-style license](MIT-LICENSE). The source code is hosted on GitHub.
