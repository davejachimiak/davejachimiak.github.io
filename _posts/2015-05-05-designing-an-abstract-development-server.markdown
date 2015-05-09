---
layout: post-no-feature
title:  "Designing an Abstract Development Server"
date:   2015-05-05 19:44:36
description:  Let's design an abstract development server for any compiled language.
categories: development software-design servers
---
There's a certain kind of program that helps programmers make web
servers in compiled languages. It compiles servers to executables and
re-serves them when a file changes. Web frameworks typically include
programs like this.

It's possible to make such a program that works for any compiled
language. It would be useful when developing smaller servers.

Let's design one. Our command line program should require:

* a server command – the command that runs the server executable
* a build command – the command that compiles the server executable

Let's call our program `serveit`. If we're developing a server in
Haskell and the compiled executable is ./web-server, we'd invoke
`serveit` like so:

{% highlight sh %}
$ serveit ./web-server "cabal build"
{% endhighlight %}

When run, it should:

1. compile the server executable
2. serve that executable
3. start a file watcher

The file watcher is the meat of our program and requires some thought.
This question will guide its design:

### What should happen when you save a file while a build is in progress?

Say we have a build running called build #1. Then a file is saved while
build #1 is running. What should happen?

Solution: kill the current build process and start a new one. It's the
most brute-force solution, but it's the best one.

The alternative is to accumulate build processes, let them run in
parallel, and rerun the server after each build finishes. This
alternative provides no benefits. It doesn't save any time. It gives
extra updates to our server, which sounds nice but isn't. By the time
the second save is invoked, we don't care about the old code. We want to
see the result of our updated code.

The alternative has another fault besides the extra work it does. Back
to build #1. Let's say it takes 10 seconds to finish. It's running, and
a file event occurs 1 second after it starts. In the alternative
scenario, that file event triggers another build called build #2, which
takes only 2 seconds to finish. This means build #2 will finish before
build #1. When build #1 finishes, the server will be serving stale code
when we want fresh code.

There are variations on that alternative, but they're no better than
killing the current build process and start a new one.

## Result

1. compile and serve a server
2. start a file watcher

The file watcher does the following on a file change:

1. Start and maintain a build process.
2. If another file change event occurs, stop the process started by 1.
   and start 1. over.
3. Restart the server, the compiled executable.

I made a version of this in Haskell and called it `waiter`. [Check it
out on Github](https://github.com/davejachimiak/waiter).

To install:

{% highlight sh %}
$ brew tap davejachimiak/tap
$ brew install waiter
{% endhighlight %}

Usage:

{% highlight sh %}
$ waiter -h
: 'Usage: waiter SERVER_COMMAND BUILD_COMMAND [-f|--file-name-regex REGEX]
              [-d|--dir DIR]

Available options:
  SERVER_COMMAND           the command to run your server
  BUILD_COMMAND            the command to build your server
  -f,--file-name-regex REGEX
                           A rebuild is triggered for changes in files
whose
                           names match this regex. Default: .*
  -d,--dir DIR             Look for file changes in this directory.
Default:
                           ./src
  -h,--help                Show this help text
'
{% endhighlight %}
