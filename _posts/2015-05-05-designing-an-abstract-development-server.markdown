---
layout: post-no-feature
title:  "Designing an Abstract Development Server"
date:   2015-05-05 19:44:36
description:  Let's design an abstract web server that runs from the command line
categories: development software-design servers
---

For compiled languages, web frameworks usually bake in a mechanism that
compiles a binary and re-serves the development server when a file
changes. Such mechanisms are custom-built according to the philosophies
of the framework. Yet, it's possible to create a general-purpose
mechanism for any compiled language.

The command line program should take at least:

* a build command
* a server command

It should do the following when it's run:

1. build the binary + serve the binary
2. initialize a file watcher

The first step is straightforward. The second one less so.

Two questions will guide our design of the file watcher.

1. What should happen when multiple files are saved at the same time?
2. What should happen when a file is saved while a compilation is in
   progress and the resulting compilation is faster than the one in
   progress?

### Multiple files saved at the "same time"

Delay compilation until all files are saved. We can't know when the last
file event from the batch will get fired. So we'll pick a small unit of
time such that we can vitually guarantee all files events will have
fired. On my OSX system using vim, one-tenth of a second is plenty of
time.

### A file event occurs while compilation is in progress

Kill the current compilation process and start a new one. It's the
simplest and most brute-force solution, but it's also the best. The
alternative is to let build processes accumulate and rerun the server
after each build finishes. This is OK, but doesn't present any positives
over killing the current compilation process and starting a new one. For
one, it doesn't save any time. It gives extra updates to our server
which sounds nice, but really isn't. By the time the second save rolls
around, we don't care about any intermittent code. We just want to see
the current state of our updated code.

### Conclusion

Yup there it is.
