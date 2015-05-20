---
layout: post-no-feature
title:  "A Functional Implementation of an Abstract Development Server"
date:   2015-05-19
description:  Let's implement an abstract development server using functional thinking.
categories: development software-design servers
---
[This post](2015/05/designing-an-abstract-development-server.html)
determined that a good design for an abstract design server would do the
following:

1. compile the server executable
2. serve that executable
3. start a file watcher

When a file changes, the file watcher should:

1. Start and maintain a build process.
2. If another file change event occurs, kill the process started by 1. and
   start 1. over.
3. Restart the server, the compiled executable.

The above sounds simple, but an implementation is bound to contain some
innate complexity. The
