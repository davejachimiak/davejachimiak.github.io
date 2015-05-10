---
layout: post-no-feature
title:  "Designing an Abstract Development Server"
date:   2015-05-05 19:44:36
description:  Let's design an abstract development server for any compiled language.
categories: development software-design servers
---
Our program will run from the command line. It should require:

* a server command – the command that runs the server executable
* a build command – the command that compiles the server executable

If:

* our program is called `waiter`,
* we're developing an HTTP server in Haskell, and
* the compiled executable is located at ./web-server,

we'd invoke `waiter` like so:

{% highlight sh %}
$ waiter ./web-server "cabal build"
{% endhighlight %}

When run, it should:

1. compile the server executable
2. serve that executable
3. start a file watcher

The file watcher should:

1. Start and maintain a build process.
2. If another file change event occurs, kill the process started by 1.
   and start 1. over.
3. Restart the server, the compiled executable.

### Why always kill the current build process on a file event?

The alternative is to accumulate build processes, let them run in
parallel, and rerun the server after each build finishes. This
alternative provides no benefits. It doesn't save any time. It gives
extra updates to our server, which sounds nice but isn't. By the time
the second save is invoked, we don't care about the old code. We want to
see the result of our updated code.

The alternative has another flaw that makes it unusable. Say there's a
running build called build #1 and it takes 10 seconds to finish. Then a
file event occurs 1 second after it starts. In the alternative scenario,
that file event triggers another build called build #2, which takes only
2 seconds to finish. This means build #2 will finish before build #1.
When build #1 finishes, the server will be serving stale code when we
want fresh code.

There are slight variations on this alternative that avoid that flaw.
However, they still present no benefit over killing the current build
process and starting a new one.

### Waiter

`waiter` exists. [Check it out on
Github](https://github.com/davejachimiak/waiter).

To install:

{% highlight sh %}
$ brew tap davejachimiak/tap
$ brew install waiter
{% endhighlight %}

Usage: `waiter SERVER_COMMAND BUILD_COMMAND [-f|--file-name-regex REGEX] [-d|--dir DIR]`

See the Github repo for more information.
