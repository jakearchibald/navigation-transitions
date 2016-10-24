# The problem

If you want to transition between pages, your current option is to fetch the new page with JavaScript, update the URL with `pushState`, and animate between the two.

Having to reimplement navigation for a simple transition is a bit much, often leading developers to use large frameworks where they aren't needed. This proposal provides a low-level way to create transitions while maintaining regular browser navigation.

# Goals

* Enable complex transitions.
* Allow transitions to start while the next page is being fetched.
* Allow transisitions to differ between link-clicking, back button, forward button, reload button.
* Allow transitions to cater for a non-zero scroll position in the navigated-to page.

# API sketch

```js
window.addEventListener('navigate', event => {
  // …
});
```

The above event fires when the document is being navigated in a way that would replace the current document.

* `event.type` - One of the following strings:
  * `back` - User is navigating back.
  * `forward` - User is navigating forward.
  * `reload` - Reload-triggered navigation.
  * `normal` - Not one of the above.
* `event.url` - The URL being navigated to.
* `event.window` - A promise for a `WindowProxy` being navigated to, unless the navigation is cross-origin or fails.
* `event.waitUntil(promise)` - Keep this document alive and potentially visible until `promise` settles.

# Simple cross-fade transition

```js
window.addEventListener('navigate', event => {
  event.waitUntil(
    event.window.then(newWin => {
      if (!newWin) return;

      // assuming newWin.document.interactive means DOM ready
      return newWin.document.interactive.then(() => {
        return newWin.document.documentElement
          .animate({opacity: 0}, 1000).finished;
      });
    })
  );
});
```

# Slide in/out transition

```js
window.addEventListener('navigate', event => {
  if (event.type == 'reload') return;

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

  event.waitUntil(
    event.window.then(newWin => {
      if (!newWin) return;
 
      return newWin.document.interactive.then(() => {
        return Promise.all([
          newWin.document.documentElement.animate(
            event.type == 'back' ? fromLeft : fromRight, 500
          ).finished,
          document.documentElement.animate(
            event.type == 'back' ? toRight : toLeft, 500
          ).finished
        ]);
      });
    })
  );
});
```

# Immediate Slide in/out transition

The above examples don't begin to animate until the new page has fetched and become interactive. That's ok, but this API allows the current page to transition while the new page is being fetched, improving the perception of performance:

```js
window.addEventListener('navigate', event => {
  if (event.type == 'reload') return;

  const newURL = new URL(event.url);

  if (newURL.origin != location.origin) return;

  const documentRect = document.documentElement.getBoundingClientRect();
  
  // Create something that looks like the shell of the new page
  const pageShell = createPageShellFor(event.url);
  document.body.appendChild(pageShell);

  const directionMultiplier = event.type == 'back' ? -1 : 1;

  pageShell.style.transform = `translate(${100 * directionMultiplier}%, ${-documentRect.top}px)`;
  const slideAnim = document.documentElement.animate({
    transform: `translate(${100 * directionMultiplier}%, 0)`
  }, 500);

  event.waitUntil(
    event.window.then(newWin => {
      if (!newWin) return;
 
      return slideAnim.finished.then(() => {
        return newWin.document.documentElement
          .animate({opacity: 0}, 200).finished;
      });
    })
  );
});
```

# Rendering

During the transition, the new document will be rendered on top of the current one. If the background of html/body is transparent, the old document will be visible through it.

During the transition, the render-box of the new document will be limited to the viewport size. This means `html { transform: translate(0, -20px); }` on the new document will leave a 20 pixel gap at the bottom, though which the current document will be visible. After the transition, rendering would switch back to the regular model.

Apologies for the hand-waving.

# Place within the navigation algorithm

Feels like the event should fire just after step 10 of [navigate](https://html.spec.whatwg.org/multipage/browsers.html#navigate). If `waitUntil` is called, the browser would consider the pages to be transitioning.

The rest of the handling would likely be in the ["update the session history with the new page" algorithm](https://html.spec.whatwg.org/multipage/browsers.html#update-the-session-history-with-the-new-page). Where the unloading of the current document would be delayed without delaying the loading of the new document.

Yep, more hand-waving.

# Potential issues & questions

* Can animations be reliably syncronised between documents? They at least share an event loop.
* Any issues firing this for nested contexts?
* What if the promise passed to `waitUntil` never resolves? Feels like it should have a timeout.
* What happens on low-end devices that can't display two documents at once?
* What if the navigation is cancelled? Eg `Content-Disposition` response. `event.window` could also cancel.
* How does this interact with browsers that have a bf cache?
* How should redirects be handled?
* How should interactivity during the transition be handled?
* During a sliding aniamtion, is it possible to switch a fake shell for the actual page mid-transition? Feels like this is something the animation API should be able to do.