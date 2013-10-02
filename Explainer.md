JavaScript-Driven Accelerated Animations
===============
{vollick, jbroman, ajuma, rjkroege}@chromium.org

Problem
-------
JavaScript-driven animations are extremely flexible and powerful, but are subject to main thread jank. And although JavaScript can run on a web worker, DOM access and css property animations are not permitted. Despite their susceptibility to main thread jank, main thread animations are widely used; they're the only way to create common effects such as parallax, position-sticky, image carousels, custom scroll animations, iphone-style contact lists, and physics-based animations. In a perfect world, the main thread would always be responsive enough that the JavaScript animation callback would get a chance to run every frame. In reality this is extremely hard to achieve (both for user agents and developers of large sites composed of disparate components). The result is lots of janky pages.

Why can't we update css properties from a worker thread? Updating css properties _could_ effect a style recalc or a layout, and those operations must happen on the main thread. That said, there are certain 'layout-free' properties that can be modified without these side effects. These properties include transform, opacity and scroll offset. Clearly identifying these layout-free properties, and allowing them to run from any thread, would provide a simple and powerful way to acheive smooth animations.

Goal
----
To allow authors to create accelerated animations of css properties via JavaScript. At first we will restrict ourselves to accelerating animations of transform, opacity and scroll offset since virtually all UAs already support accelerated animation of these properties, either by accelerated css animations or threaded scrolling.

High Level Proposal
-------------------
Allow the creation of a proxy to layout-free properties. This proxy could then be used (asynchronously) on web workers. It will also be convenient to allow requestAnimationFrame callbacks to be executed on web worker threads.

A Tiny, But Surprisingly Expressive Kernel
------------------------------------------
What does this proxy buy us? Well, a number of proposals that have been drafted to address some of the use cases mentioned in the problem statement could largely be implemented as polyfills on top of this kernel. So, too, could some existing web APIs. Specifically,

 - Some bits of [Web Animations](http://dev.w3.org/fxtf/web-animations/)
 - [Touch-based Animation Scrubbing](https://docs.google.com/document/d/1vRUo_g1il-evZs975eNzGPOuJS7H5UBxs-iZmXHux48/edit)
 - [position:sticky](http://updates.html5rocks.com/2012/08/Stick-your-landings-position-sticky-lands-in-WebKit)
 - [Smooth Scrolling](http://dev.w3.org/csswg/cssom-view/), Sections 4, 5, 7, 12, and 13.
 - Accelerated CSS animations. ([I/O talk](http://www.youtube.com/watch?v=hAzhayTnhEI))

It’s clear from the number of attempts at solving the smooth animation problem that this is extremely important for the web, and it would be wonderful if this problem could be addressed with a small, easy-to-implement kernel. It would also let creative web developers create new animation authoring libraries that are just as performant as their native counterparts. In addition to its simplicity, another benefit of this approach is that it ‘explains the web’ in the [Extensible Web Manifesto](extensiblewebmanifesto.org) sense. In particular, it explains accelerated animations. This brings us to...

The Big Concern
---------------
We want to explain the web, not the particular implementation details of a particular browser at a particular point in its history. Are we marrying ourselves to implementation details here?

No! Virtually all user agents support, via css, accelerated opacity and transform animations. And they’re going to have to support them for the foreseeable future. By whatever means these browsers are able to guarantee that things can slide around and fade in and out efficiently for css animations, they will be able to permit these effects to be driven by JavaScript. It doesn’t, for example, tie us to the idea of a composited layer or a layer tree, concepts that may not even exist in all browser implementations. The animated DOM elements might, say, be redrawn by the GPU each frame. But this doesn’t matter. These implementation details are orthogonal to the animation proxy concept.

What About The Details?
-----------------------
This explainer was kept intentionally high level. Our hope is to convince you of the merit of this problem and idea before diving into a debate about our particular solution. We, of course, have some idea of what the proxy API might look like, how you might create them and how we could address timing issues, but that conversation will come later. We look forward to having it with you!

...well, maybe it wouldn't hurt to give a few examples. Before getting to the code, I want to emphasize that I do not expect that the average web developer will ever have to deal with animation proxies directly. It would be a simple matter to wrap the animation proxy code in a jQuery-like JavaScript library that manages the animation web worker thread and all proxies. I imagine that most developers would just use such a library. That said, here are a few snippets.

### parallax
```JavaScript
// On the main thread.
overflow_scroll_proxy = overflow_scroll_element.getAnimationProxy();
parallax_scroll_proxy = parallax_scroll_element.getAnimationProxy();

worker.postMessage({
  'scale': 0.9,
  'overflow_proxy': overflow_scroll_proxy,
  'parallax_proxy': parallax_scroll_proxy,
});

// On the web worker.
function on_message(msg) {
  parallax_scroll_scale = msg.scale;
  overflow_scroll_proxy = msg.overflow_proxy;
  parallax_scroll_proxy = msg.parallax_proxy;
  self.requestAnimationFrame(raf_callback);
}

function raf_callback(timestamp) {
  overflow_scroll_proxy.getScrollOffset().then(do_parallax, error);
}

function do_parallax(offset) {
  offset.scale(parallax_scroll_scale);
  parallax_scroll_proxy.setScrollOffset(offset)
    .then(function(result) { self.requestAnimationFrame(raf_callback); },
          error);
}
```

### position: sticky
```JavaScript
// This is just the meat of the web worker side. Work would definitely need to be done on
// the main thread to compute min_offset and max_offset and to pass that information to
// the worker.
function raf_callback(timestamp) {
  overflow_scroll_proxy.getScrollOffset().then(do_sticky, error);
}

function do_sticky(offset) {
  var compensation = min_offset - Math.min(Math.max(offset.y, min_offset), max_offset);
  var transform = new CSSMatrix();
  transform.Translate(0, compensation);
  sticky_element_proxy.setTransform(transform)
    .then(function(result) { self.requestAnimationFrame(raf_callback); },
          error);
}
```
