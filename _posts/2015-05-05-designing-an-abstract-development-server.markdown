---
layout: post-no-feature
title:  "Designing an Abstract Development Server"
date:   2015-05-05 19:44:36
description:  Let's design an an abstract development server for any compiled language.
categories: development software-design servers
---
Web frameworks of compiled languages have command line programs that,
when files change, build servers to binaries and re-serve them. Yet,
it's possible to create a general-purpose mechanism for any compiled
language.

Our command line interface should require:

* a build command
* a server command

Let's call our program `serveit`. If we're developing a server in
Haskell, and the compiled binary is located at ./web-server, we'd invoke
`serveit` like so:

{% highlight sh %}
$ serveit "cabal build" ./web-server
{% endhighlight %}

It should do the following when it's run:

1. compile the server binary
2. serve that binary
3. initialize a file watcher

The file watcher should be thought of as a program in and of itself. Two
questions will guide its design.

1. What should happen when you save many files at the same time?
2. What should happen when you save a file while a build is in
   progress? What if the second build finishes before the
   one first?

### Saving many files at the same time

This happens, for example, when you invoke `:wa` in vim after changing
many files.

Solution: delay the build until the file system fires events for all
files in the batch. We can't know when the last file event from the
batch will get fired. So pick a small unit of time such that we can
guarantee all files events will have fired within it. On my system,
one-tenth of a second gives good coverage.

### A file event occurs while a build is in progress

Solution: kill the current build process and start a new one. It's the
most brute-force solution, but it's also the simplest and best one.

The alternative is to let build processes finish and then rerun the
server after each build finishes. This is OK, but doesn't present any
positives. It doesn't save any time. It gives extra updates to our
server which sounds nice, but isn't. By the time the second save rolls
around, we don't care about any intermittent code. We just want to see
the current state of our updated code.

There is a fault to this alternative along with the extra work it does.
It serves the wrong version of the server if the second build
finishes before the first. In this case, the second build process
finishes and restarts the server first and the first build process
finishes and restarts the server after the second process. The server
is now serving stale code when we want fresh code.

So kill the current build process and start a new one.

### Result

1. compile and server a server
2. start a file watcher

The file watcher does the following on a file change:

1. Wait a tenth of a second and ignore all file change events during
   that time.
2. Start and maintain a compilation process.
3. If another file change event occurs, stop the process started by 2.
   and start 2. over.
4. Restart the server – the compiled binary.
