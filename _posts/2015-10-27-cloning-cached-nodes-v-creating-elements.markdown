---
layout: post-no-feature
title:  "DOM performance: Creating elements vs. cloning cached elements"
date:   2015-10-27
categories: javascript performance
---

I became interested in comparing these after checking out
implementations of virtual DOMs.

I hypothesized that cloning cached DOM nodes would perform way better,
but you never really know when you're dealing with the DOM. Maybe behind
the scenes in some browsers, creating a bare `<div>` with
`createElement` is already an alias to cloning a cached bare `<div>`?

But no, it's exactly what you would think. `cloneNode` runs O(1) off of
an already created element while `createElement` runs O(n) where n is
the number of times you create an element. Nothing surprising, but
still good to know.

Check out the [jsperf
test](http://jsperf.com/caching-nodes-v-creating-nodes).
