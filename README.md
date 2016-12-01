# The problem

If you want to transition between pages, your current option is to fetch the new page with JavaScript, update the URL with [`pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History_API#The_pushState()_method), and animate between the two.

Having to reimplement navigation for a simple transition is a bit much, often leading developers to use large frameworks where they could otherwise be avoided. This proposal provides a low-level way to create transitions while maintaining regular browser navigation.

# Goals

* Enable complex transitions.
* Allow transitions to start while the next page is being fetched.
* Allow transitions to differ between navigation from clicking a link, back button, forward button, reload button, etc.
* Allow transitions to cater for a non-zero scroll position in the navigated-to page.

# Experiments, talks & reading materials

## Web Navigation Transitions (2014): CSS-based experiment by Google Chrome team
* [**Spec:** Initial (deprecated) spec for experimental Web Navigation Transitions prototype (2014)](https://docs.google.com/document/d/17jg1RRL3RI969cLwbKBIcoGDsPwqaEdBxafGNYGwiY4/edit)
* [**Talk (video):** Ryan Schoen's talk, Wicked Fast: Performance investments (Chrome Dev Summit 2014)](https://www.youtube.com/watch?v=v0xRTEf-ytE)
    * [**Demo (video):** Web Navigation Transitions in Chrome on Android](https://www.youtube.com/watch?v=v0xRTEf-ytE&t=968)
    * [**Demo (video):** Android's Activities Transitions API and experimental Web Navigation Transitions in Chrome on Android](https://www.youtube.com/watch?v=v0xRTEf-ytE&t=1075)

## Web Navigation Transitions (2015): CSS-based proposal by Chris Lord (Mozilla, Firefox OS)
* [**Article:** Web Navigation Transitions by Chris Lord (Mozilla)](http://chrislord.net/index.php/2015/04/24/web-navigation-transitions/)
* [**Examples:** Web Navigation Transitions examples (emulated with CSS Animations, custom CSS media queries, events, `<iframe>`s, `postMessage`, etc.)](http://chrislord.net/files/mozilla/gaia-navigator/examples/)
* [**Code samples:** Source code of "Gaia Nagiator" Web Navigation Transitions examples](https://github.com/Cwiiis/gaia-navigator)

## Web Navigation Transitions (2016): New proposal by Jake Archibald (Google Chrome)
* [**Talk (video):** Intro to this proposal (Chrome Dev Summit 2016)](https://www.youtube.com/watch?v=J2dOTKBoTL4&t=1815)

# API sketch

```js
window.addEventListener('navigate', event => {
  // …
});
```

The `navigate` event fires when the document is being navigated in a way that would replace the current document.

* `event.type` - The name of this event, `navigate`.
* `event.reason` - The way in which the document is being navigated to. One of the following strings:
  * `back` - User is navigating back.
  * `forward` - User is navigating forward.
  * `reload` - Reload-triggered navigation.
  * `normal` - Not one of the above.
* `event.url` - The URL being navigated to. An empty string if the URL is of another origin.
* `event.newWindow` - A promise for a `WindowProxy` being navigated to. Resolves with `undefined` if another origin is involved in the navigation (i.e., the initial URL or URLs of redirects). Rejects if the navigation fails. Cancels if the navigation cancels (dependent on cancelable promises).
* `event.transitionUntil(promise)` - Keep this document alive and potentially visible until `promise` settles, or once another origin is involved in the navigation (i.e., the initial URL or URLs of redirects).

**Note:** The same-origin restrictions are to avoid new URL leaks and timing attacks.

# Simple cross-fade transition

```js
window.addEventListener('navigate', event => {
  event.transitionUntil(
    event.newWindow.then(newWin => {
      if (!newWin) return;

      // assuming newWin.document.interactive means DOM ready
      return newWin.document.interactive.then(() => {
        return newWin.document.documentElement.animate([
          {opacity: 0}, {opacity: 1}
        ], 1000).finished;
      });
    })
  );
});
```

# Slide-in/out transition

```js
window.addEventListener('navigate', event => {
  if (event.reason == 'reload') return;

  const fromRight = [
    {transform: 'translate(100%, 0)'},
    {transform: 'none'}
  ];

  const toLeft = [
    {transform: 'none'},
    {transform: 'translate(-100%, 0)'}
  ];

  const fromLeft = toLeft.slice().reverse();
  const toRight = fromRight.slice().reverse();

  event.transitionUntil(
    event.newWindow.then(newWin => {
      if (!newWin) return;
 
      return newWin.document.interactive.then(() => {
        return Promise.all([
          newWin.document.documentElement.animate(
            event.reason == 'back' ? fromLeft : fromRight, 500
          ).finished,
          document.documentElement.animate(
            event.reason == 'back' ? toRight : toLeft, 500
          ).finished
        ]);
      });
    })
  );
});
```

# Immediate slide-in/out transition

The above examples don't begin to animate until the new page has fetched and become interactive. That's ok, but this API allows the current page to transition while the new page is being fetched, improving the perception of performance:

```js
window.addEventListener('navigate', event => {
  if (event.reason == 'reload') return;

  const newURL = new URL(event.url);

  if (newURL.origin !== location.origin) return;

  const documentRect = document.documentElement.getBoundingClientRect();

  // Create something that looks like the shell of the new page
  const pageShell = createPageShellFor(event.url);
  document.body.appendChild(pageShell);

  const directionMultiplier = event.reason == 'back' ? -1 : 1;

  pageShell.style.transform = `translate(${100 * directionMultiplier}%, ${-documentRect.top}px)`;

  const slideAnim = document.body.animate({
    transform: `translate(${100 * directionMultiplier}%, 0)`
  }, 500);

  event.transitionUntil(
    event.newWindow.then(newWin => {
      if (!newWin) return;
 
      return slideAnim.finished.then(() => {
        return newWin.document.documentElement
          .animate({opacity: 0}, 200).finished;
      });
    })
  );
});
```

# Rendering & interactivity

During the transition, the document with the highest `z-index` on the `documentElement` will render on top. If `z-index`es are equal, the entering document will render on top. Both `documentElement`s will generate stacking contexts.

If the background of `html`/`body` is transparent, the underlying document will be visible through it. Beneath both documents is the browser's default background (usually white).

During the transition, the render-box of the documents will be clipped to that of the viewport size. This means `html { transform: translate(0, -20px); }` on the top document will leave a 20-pixel gap at the bottom, through which the bottom document will be visible. After the transition, rendering switches back to using the regular model.

We must guarantee that the new document doesn't visibly appear until `event.newWindow`'s reactions have completed.

As for interactivity, both documents will be at least scrollable, although developers could prevent this using `pointer-events: none` or similar.

Apologies for the hand-waving.

# Place within the navigation algorithm

It feels like the event should fire immediately after step 10 of [`navigate`](https://html.spec.whatwg.org/multipage/browsers.html#navigate). If `transitionUntil` is called, the browser would consider the pages to be transitioning.

The rest of the handling would likely be in the ["update the session history with the new page" algorithm](https://html.spec.whatwg.org/multipage/browsers.html#update-the-session-history-with-the-new-page). The unloading of the current document would be delayed but without delaying the loading of the new document.

Yep, more hand-waving.

# Potential issues & questions

* Can transitions/animations be reliably synchronised between documents? They at least share an event loop.
* Any issues with firing transitions/animations for nested contexts?
* What if the promise passed to `transitionUntil` never resolves? Feels like it should have a timeout.
* What happens on low-end devices that can't display two documents at once?
* What if the navigation is cancelled (e.g., use of a `Content-Disposition` response header). `event.newWindow` could also cancel.
* How does this interact with browsers that have a [back-forward cache](https://developer.mozilla.org/en-US/docs/Working_with_BFCache)?
* How should redirects be handled?
* How should interactivity during the transition be handled?
* During a sliding transitions, is it possible to switch a fake shell for the actual page's document mid-transition? Feels like this is something the [Animation API](https://www.w3.org/TR/web-animations-1/) should be able to do.
