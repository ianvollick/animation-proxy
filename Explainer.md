JavaScript-Driven Accelerated Animations
===============
{vollick, jbroman, ajuma, rjkroege}@chromium.org

Problem
-------
JavaScript-driven animations are extremely flexible and powerful, but are subject to main thread jank (by jank, I mean variability in the rate that animations are serviced on a thread due to other, unrelated work on that thread). And although JavaScript can run on a web worker, DOM access and CSS property animations are not permitted. Despite their susceptibility to main thread jank, main thread animations are widely used; they're the only cross-browser way to create common effects such as parallax, position-sticky, image carousels, custom scroll animations, iphone-style contact lists, and physics-based animations. In a perfect world, the main thread would always be responsive enough to guarantee that a JavaScript animation callback would be serviced every frame. In reality, this is extremely hard to achieve, both for user agents and developers of large sites composed of disparate components. The result is a lot of janky pages.

The normal guidance for avoiding janky animations is to use only accelerated CSS animations.  This works because the browser can service the animation on another thread (not subject to the jank on the main thread).  Unfortunately, this severely limits the expressiveness of the animation, as the author must be able to describe the animation in a simple non-extensible declarative language.

Why can't apps just update CSS properties from javascript running on a worker thread? The problem is that updating CSS properties _could_ effect a style recalc or a layout and those operations must happen on the main thread. Browsers get around this for accelerated CSS animations by accelerating only certain 'layout-free' properties that can be modified without these side effects. Layout-free properties include transform, opacity and scroll offset. Allowing these properties to be set on any thread in javascript (instead of just by the browser) would provide a simple way to enable smooth custom animations.

Goal
----
The goal here is to allow web developers to drive accelerated animations of certain CSS properties via JavaScript from a worker thread. We will focus on transform, opacity and scroll offset animations initially since virtually all user agents already support accelerated animation of these properties, either by accelerated CSS animations or threaded scrolling.

High Level Proposal
-------------------
Allow the creation of proxy objects that provide access to certain layout-free properties. These proxies could then be used (asynchronously) on web workers.

A Tiny, But Surprisingly Expressive Kernel
------------------------------------------
What does this proxy buy us? A number of proposals that have been drafted to address some of the use cases mentioned in the problem statement could largely be implemented on top of this kernel. So, too, could some existing web APIs. Specifically,

 - Some bits of [Web Animations](http://dev.w3.org/fxtf/web-animations/)
 - [Touch-based Animation Scrubbing](https://docs.google.com/document/d/1vRUo_g1il-evZs975eNzGPOuJS7H5UBxs-iZmXHux48/edit)
 - [position:sticky](http://updates.html5rocks.com/2012/08/Stick-your-landings-position-sticky-lands-in-WebKit)
 - [Smooth Scrolling](http://dev.w3.org/csswg/cssom-view/), Sections 4, 5, 7, 12, and 13.
 - Accelerated CSS animations. ([I/O talk](http://www.youtube.com/watch?v=hAzhayTnhEI))

There have been many attempts at creating web APIs to make authoring smooth animations both possible and easy. It would be wonderful if the problem could be addressed with a small, easy-to-implement kernel. This would let creative web developers devise new animation authoring libraries that are just as performant as their native counterparts. A simple threaded-animation kernel would help ‘explain the web’ in the [Extensible Web Manifesto](extensiblewebmanifesto.org) sense. In particular, it explains accelerated animations. This brings us to...

One Potential Concern
---------------------
We want to explain the web, not the particular implementation details of a particular browser at a particular point in its history. Are we marrying ourselves to implementation details here?

I think the answer is no. Virtually all user agents support (via CSS) accelerated opacity and transform animations, and they’re going to have to support them for the foreseeable future. Threaded scrolling is also increasingly common. By whatever means browsers are currently able to guarantee that things can slide around, scroll and fade in and out efficiently, they could potentially permit these effects to be driven by JavaScript on a web worker. It doesn’t, for example, tie us to the idea of a composited layer or a layer tree, concepts that may not be meaningful in all browser implementations. The animated elements might, say, be redrawn by the GPU each frame. But this doesn’t matter. These implementation details are orthogonal to the animation proxy concept.

What About The Details?
-----------------------
This explainer was kept intentionally high level. Our hope is to convince you of the merit of this problem and idea before diving into a debate about our particular solution. We, of course, have some thoughts about what the proxy API could look like, how you might create them, and how we could address timing issues, but those details will get worked out as we discuss this idea with you!

...well, maybe it wouldn't hurt to give a few examples using a hypothetical API (which will probably look a lot different than the final version). Before getting to the code, I want to emphasize that I do not expect that the average web developer will ever have to deal with animation proxies directly. It would be a simple matter to wrap the animation proxy code in a jQuery-like JavaScript library that manages the animation web worker thread and all proxies. I imagine that most developers would just use such a library. That said, here are a few snippets.

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
  self.scheduleTick(tick);
}

function tick(timestamp) {
  overflow_scroll_proxy.getScrollOffset().then(do_parallax, error);
}

function do_parallax(offset) {
  offset.scale(parallax_scroll_scale);
  parallax_scroll_proxy.setScrollOffset(offset)
    .then(function(result) { self.scheduleTick(tick); },
          error);
}
```

### position: sticky
```JavaScript
// This is just the meat of the web worker side. Work would definitely need to be done on
// the main thread to compute min_offset and max_offset and to pass that information to
// the worker.
function tick(timestamp) {
  overflow_scroll_proxy.getScrollOffset().then(do_sticky, error);
}

function do_sticky(offset) {
  var compensation = min_offset - Math.min(Math.max(offset.y, min_offset), max_offset);
  var transform = new CSSMatrix();
  transform.Translate(0, compensation);
  sticky_element_proxy.setTransform(transform)
    .then(function(result) { self.scheduleTick(tick); },
          error);
}
```
