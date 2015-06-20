---
layout: post-no-feature
title:  "A Functional Implementation of an Abstract Development Server"
date:   2015-05-19
description:  Let's implement an abstract development server in Haskell.
categories: development software-design servers
---
Implementing an abstract development server in a purely functional
language is fun and interesting. The most interesting part is answering:
what exactly should happen when a file event is fired?

[A previous post](/2015/05/designing-an-abstract-development-server.html)
determined a loose design for an abstract development server:

* * *

1. compile the server executable
2. serve that executable
3. start a file watcher

When a file event happens, the file watcher should run a callback that:

* starts and maintains a build process,
* terminates that process if another file event occurs, and
* restarts the server after the build process finishes, but only if it
  we didn't terminate it.

* * *

Why do we terminate the current build process if a file event happens?

Because concurrent callbacks are possible.

Compiling takes time. If a compilation takes five seconds and a file
event occurs in the third, that means we now have two callbacks running
at the same time. This also means that we want to cancel first
compilation and start a new one; we don't care about old code when
developing our server.

The specification for the above callback is unneccesarily complex. It
listens to file events even though it was triggered by a file event
itself. A simpler version doesn't need to know about file events:

* * *

* Terminate the current build process if one exists.
* Start and maintain a new build.
* Restart the server after that build ends, but only if it we didn't
  terminate it in some other thread.

* * *

This version requires maintaining the current build and server
processes across concurrent callbacks; the file watcher will run the
callback in a new thread for each file event.

[`System.Process`](https://hackage.haskell.org/package/process-1.2.0.0/docs/System-Process.html)
helps to manage processes in Haskell, and `MVar`s will help to maintain these
processes across threads.

## Managing processes

### Spawning processes
`System.Process.spawnProcess` takes a string – which is evaluated in a
shell – and returns an `IO ProcessHandle`. It's how we'll spawn builds
and servers.

{% highlight haskell %}
currentBuild <- spawnProcess buildCommand
{% endhighlight %}

In this example, `buildCommand` is some string passed into our program
from the command line. `currentBuild` is a `ProcessHandle` that
represents the current server process, and can be passed around to
functions as need be.

### Terminating processes
`System.Process.terminateProcess` takes a `ProcessHandle` and sends a
`SIGTERM` signal to it in Unix. In Windows it sends a `TerminateProcess`
signal. We'll use this to terminate the build and server processes in
the callback to the file watcher.

{% highlight haskell %}
terminateProcess serverProcess
{% endhighlight %}

### Waiting for processes
`System.Process.waitForProcess` takes a `ProcessHandle` and returns an
`ExitCode` after waiting for the process to finish. In the callback to
the file watcher, we need to know whether we've terminated the build
process we initially spawned. If the process exits successfully, it
returns an `ExitSuccess` and we'll know to restart the server. If the
process is terminated, it'll return an `ExitFailure` and we'll know to
do nothing.

{% highlight haskell %}
buildProcessExitCode <- waitForProcess buildProcess
{% endhighlight %}

We'll use `waitForProcess` in an interesting way below.

## Managing processes across threads
We'll use `MVar`s to manage processes across concurrent callbacks. An `MVar`
is a wrapper for a value. The wrapper stores the value in a block of
memory that is shared across threads. Many actions can be performed on
`MVar`s, and they're generally used to synchronize and send data between
threads.

[`Control.Concurrent.MVar`](https://hackage.haskell.org/package/base-4.8.0.0/docs/Control-Concurrent-MVar.html)
is only one way of managing shared memory in Haskell. We'll probably be
using
[`System.FSNotify`](https://hackage.haskell.org/package/fsnotify-0.1.0.3/docs/System-FSNotify.html)
to listen to file events, which has a hard dependency on `MVar`. So
there's no need to bring in another mechanism.

### Creating empty `MVar`s
`Control.Concurrent.MVar.newEmptyMVar` creates a new, empty `MVar`. We
need to create empty `MVar`s to prepare for storing references to the
build and server processes.

{% highlight haskell %}
currentServer <- newEmptyMVar
currentBuild <- newEmptyMVar
{% endhighlight %}

### Putting a value into an `MVar`
`Control.Concurrent.MVar.putMVar` takes an `MVar` and a value and puts
the value in the `MVar`.  We'll use it to store a reference to the
server process to terminate it later. We'll also use it to put build
processes into `currentBuild`.

{% highlight haskell %}
putMVar currentServer serverProcess
{% endhighlight %}

If `serverProcess` is some `ProcessHandle` of an already running server,
`currentServer` in the above code now shares the `serverProcess` across
threads.

### Taking a value out of an `MVar`
`Control.Concurrent.MVar.tryTakeMVar` is one way of taking a value out
of an `MVar`. It takes an `MVar` and returns `Maybe a`. `a` will always
be a `ProcessHandle` in our case. If a `ProcessHandle` is wrapped in the
`MVar` it returns `Just ProcessHandle`. If nothing is wrapped in the
`MVar` it returns `Nothing`.

We'll use `tryTakeMVar` to see whether there's a build current running.
If a build process is in `currentBuild`, we'll end it with
`terminateProcess`. Otherwise, we won't do anything.

We can create a reusable function for that called
`maybeTerminateProcessFromMVar`.

{% highlight haskell %}
maybeTerminateProcessFromMVar currentBuild

maybeTerminateProcessFromMVar :: MVar ProcessHandle -> IO ()
maybeTerminateProcessFromMVar currentBuild = do
    process <- tryTakeMVar currentBuild

    case process of
        Just process -> terminateProcess process
        Nothing -> return
{% endhighlight %}

## Putting it all together

The callback to the file watcher is the meat and potatoes of our
abstract development server. So we'll concern ourselves with that only.

We'll call it `buildAndServe`.

{% highlight haskell %}
buildAndServe :: CommandLine
                -> MVar ProcessHandle
                -> MVar ProcessHandle
                -> IO ()
buildAndServe commandLine currentBuild currentServer = do
    maybeTerminateProcessFromMVar currentBuild

    startBuild (buildCommand commandLine) currentBuild
        >>= waitForProcess
        >>= restartServer commandLine serverProcess
{% endhighlight %}

`buildAndServe` takes a `CommandLine` and the `MVar`s that may contain
our build and server processes. The `CommandLine` contains options
passed from the user.

{% highlight haskell %}
data CommandLine = CommandLine
    { serverCommand :: String
    , buildCommand :: String }
{% endhighlight %}

First, we terminate the currently running build with
`maybeTerminateProcessFromMVar`. Here it is again:

{% highlight haskell %}
maybeTerminateProcessFromMVar :: MVar ProcessHandle -> IO ()
maybeTerminateProcessFromMVar currentBuild = do
    process <- tryTakeMVar currentBuild

    case process of
        Just process -> terminateProcess process
        Nothing -> return
{% endhighlight %}

Then we start a new build with `startBuild`.

{% highlight haskell %}
startBuild :: String -> MVar ProcessHandle -> IO ProcessHandle
startBuild buildCommand currentBuild = do
    build <- spawnCommand buildCommand
    putMVar currentBuild build
    return build
{% endhighlight %}

`startBuild` takes the build command and the `currentBuild` `MVar`. It
spawns the build process, puts it into the `currentBuild` `MVar`, and
returns the build `ProcessHandle` wrapped in `IO`.

Returning `IO ProcessHandle` gives us the concision of calling
`waitForProcess` in a [pointfree](https://wiki.haskell.org/Pointfree)
manner.

{% highlight haskell %}
startBuild (buildCommand commandLine) currentBuild
    >>= waitForProcess
{% endhighlight %}

That is equivalent to calling `waitForProcess build`, where `build` is
the `ProcessHandle` from `spawnCommand buildCommand` in `startBuild`.
The second argument to bind (`>>=`) must be a function. That function is
applied to the value wrapped in the monadic constructor returned from
the first argument of `>>=`. Our value wrapped in that monadic
constructor – `IO` in this case – is the `build`. So `waitForProcess` is
applied to `build`.

We reap the benefits of the pointfree style again when `restartServer` is
applied to the exit code that `waitForProcess` returns.

{% highlight haskell %}
startBuild (buildCommand commandLine) currentBuild
    >>= waitForProcess
    >>= restartServer commandLine serverProcess
{% endhighlight %}

To reiterate: `waitForProcess` is applied to the `build` returned and
wrapped in `IO` from `startBuild`. Then `restartServer` is applied to
`commandLine`, `serverProcess`, and the `ExitCode` returned and wrapped
in `IO` from `waitForProcess`.

`restartServer` is as follows:

{% highlight haskell %}
restartServer :: CommandLine -> MVar ProcessHandle -> ExitCode -> IO ()
restartServer commandLine currentServer ExitSuccess = do
    maybeTerminateProcessFromMVar currentServer
    spawnCommand (serverCommand commandLine) >>= putMVar serverProcess
restartServer _ _ _ = return ()
{% endhighlight %}

Here, the pointfree style lends itself to easy pattern-matching on the
`ExitCode` value constructors. Remember: we don't want to restart the
server if the build process was terminated in some other thread. So we
won't do anything if the `ExitCode` is not `ExitSuccess`:

{% highlight haskell %}
restartServer _ _ _ = return ()
{% endhighlight %}

## Waiter
This program exists as a command line tool called `waiter`. [Check it
out on Github](https://github.com/davejachimiak/waiter). The end result
differs from the description here, but contains the main idea.

To install:

{% highlight sh %}
$ brew tap davejachimiak/tap
$ brew install waiter
{% endhighlight %}

Usage: `waiter SERVER_COMMAND BUILD_COMMAND [-f|--file-name-regex REGEX] [-d|--dir DIR]`

See the Github repo for more information.
