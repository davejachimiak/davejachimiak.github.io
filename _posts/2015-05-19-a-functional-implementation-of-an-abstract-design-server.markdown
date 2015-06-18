---
layout: post-no-feature
title:  "A Functional Implementation of an Abstract Development Server"
date:   2015-05-19
description:  Let's implement an abstract development server using functional thinking.
categories: development software-design servers
---
[This post](2015/05/designing-an-abstract-development-server.html)
determined that a minimal abstract development server would do the
following:

1. compile the server executable
2. serve that executable
3. start a file watcher

When a file changes, the file watcher should:

1. Start and maintain a build process.
2. If another file change event occurs, kill the process started by 1. and
   start 1. over.
3. Restart the server, the compiled executable.

This program is bound to be interesting when implemented with a purely
functional language. On closer inspection, this program contains two
states that must be accessed whenever a sub-program is invoked through a
file event. State is handled explicitly in purely functional languages,
which is very different from imperative langagues. Also, our program
will be manipulating processes, which means talking to the operating
system through IO. So let's take a closer look at this program and see
how parts of it may be implemented in Haskell.

## The Parts of the Program
Let's break the program into different parts in order to understand it
better: Boot, the File Watcher, and the Build-And-Serve sub-program.

### Boot
The program builds the server binary and run it when the program boots.

### File watcher
The file watcher listens to the operating system for file-change events.
It runs the build-and-serve sub-program on each file-change event.

### Build-And-Serve sub-program
The build-and-serve sub-program terminates a currently running build if
it exists. It then kicks off a new build. If the build process ends
successfully, it terminates the current server process and starts a new
one.

## The states of the program

### Build process
Why keep track of the current build process? We want to cancel the
current build if a file event happens while it's occuring.

We'll keep track of build processes in references. The state of a
reference will determine it's meaning. In terms of the build processes,
an empty reference will mean that there's no build process occuring
right now. A reference that contains a build process will mean that
there is a build process occuring right now.

### Server process
Why keep track of the current server process? We need to serve the
latest version of our development server. Our server is tying up a port
on localhost, and if a new server is built, we want to terminate the
current server process and start and store another one.

## A way to model and store state in Haskell
Haskell has a few ways to maintain state. Using
`Control.Concurrent.MVar` is one way.  That's what we'll use here, since
the file watching abstraction we'll be using, `FSNotify`, already has a
dependency on it.

An `MVar` is a way to manage shared memory between threads. When an
`MVar` is fulfilled, it contains a reference to any type of data that
was put in there.

## Using `MVar`s to manage state

### Boot

#### Creating `MVar`s
We need to create empty `MVar`s to prepare for storing references to the
build process and server process. We'll call those references
`buildState` and `serverState`. Create new, empty `MVar`s with
`newEmptyMVar`.

{% highlight haskell %}
serverState <- newEmptyMVar
currentBuild <- newEmptyMVar
{% endhighlight %}

#### Storing a reference to the server process
We should store a reference to the server process once we build and
spawn it. Use `putMVar` to put stuff into `MVar`s. Where serverProcess
is some `ProcessHandle` of an already running server:

{% highlight haskell %}
putMVar serverState serverProcess
{% endhighlight %}

### Build-and-Serve Sub-Program

#### The Build State
The first thing we do is determine whether there's a build currently
running. We do that by inspecting `buildState`. `MVar`'s interface for
inspecting whether something is contained within it is `tryTakeMVar`. It
takes the item out the `MVar` if it's there and returns a `Maybe a`. In
our case `a` is a `ProcessHandle`. If `tryTakeMVar` returns `Just a`, we
terminate the process in the `ProcessHandle`. If it's `Nothing`, we
don't do anything.

{% highlight haskell %}
maybeTerminateProcessFromMVar :: MVar ProcessHandle -> IO ()
maybeTerminateProcessFromMVar buildProcess = do
    process <- tryTakeMVar buildProcess

    case process of
        Just process -> terminateProcess process
        Nothin -> return
{% endhighlight %}

The second thing we do is spawn a new build process. Immediately after
we should put that process in our `buildState` with `putMVar`.

{% highlight haskell %}
putMVar currentBuild buildProcess
{% endhighlight %}

`currentBuild` will always be empty before this point. We've already
removed any possible existing process contained within `currentBuild`
with `tryTakeMVar`.

#### The Server State
We want to re-serve our server – that is, if and only if the build
process wasn't killed. This means we should take the server process out
of the `serverState` and terminate it. We already have a meachnism to
take a `ProcessHandle` out of an `MVar` and maybe terminate it
– `maybeTerminateProcessFromMVar`. So we can just use that; no need to
duplicate the work.

{% highlight haskell %}
maybeTerminateProcessFromMVar serverState
{% endhighlight %}

Once the old server is terminated, we'll spawn a new serve process and
use `putMVar` to put it into the `serverState`.

{% highlight haskell %}
putMVar serverState serverProcess
{% endhighlight %}

## Performing actions on `ProcessHandle`s contained in the `MVar`s
Many useful functions create and operate on
`System.Process.ProcessHandle`s. Here's a few of them that will help us
out in our program.

### Spawning processes
We need to spawn processes in order to create builds and servers.
`spawnProcess` takes a string which is evaluated in a shell and returns
an `IO ProcessHandle`. The `ProcessHandle` type class provides an
interface to perform actions on processes.

{% highlight haskell %}
buildProcess <- spawnProcess buildCommand
putMVar currentBuild buildProcess
{% endhighlight %}

### Terminating processes
We need to terminate the server and buil process in the build-and-serve
sub-program. `terminateProcess` (as already shown above) takes a
`ProcessHandle` and sends a `SIGTERM` (or its Windows equivalent,
TerminateProcess) signal to it.

{% highlight haskell %}
terminateProcess serverProcess
{% endhighlight %}

### Waiting for processes
In the build-and-serve sub-program, we need to know whether a build
process has terminated or not. We can wait for the process to finish
with `waitForProcess`. `waitForProcess` will return the exit code as a
value constructor of the `ExitCode` data type. If the process exits
successfully, it returns an `ExitSucess` and we can continue to
restarting the server. If the process is terminated, it'll return an
`ExitFailure` and we can end our sub-program.

## Waiter
By the way, this exists as a command line tool called `waiter`. [Check
it out on Github](https://github.com/davejachimiak/waiter).

To install:

{% highlight sh %}
$ brew tap davejachimiak/tap
$ brew install waiter
{% endhighlight %}

Usage: `waiter SERVER_COMMAND BUILD_COMMAND [-f|--file-name-regex REGEX] [-d|--dir DIR]`

See the Github repo for more information.
