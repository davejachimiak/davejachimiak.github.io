---
layout: post-no-feature
title:  "A Functional Implementation of an Abstract Development Server"
date:   2015-05-19
description:  Let's implement an abstract development server using functional thinking.
categories: development software-design servers
---
I. Levels of the program
  A. Boot
    1. Build the server binary
    2. Run the server binary
  B. File-watcher
    1. This listens to file-change events coming from the operating
       system. It runs the build-and-serve program on every event.
  C. Build-And-Serve program
    1. This cancels an old build if it exists, kicks off a new build,
       kills the current server, and serves the newly-built server.
II. The states of the program
  A. Build process
    1. Why keep track of this?
      a. We want to cancel the current build if a file event triggers
         another build.
    2. nature of the state
      a. It takes finite but unknown time. We know that the build
         process should end. If we store a reference to the running process, we
         can act differently depending on whether the reference has a build
         process or not.
  B. Server process
    1. Why keep track of this?
      a. We need to serve the latest version of our code. Presumably,
         our server is tying up a port on localhost, and if a new server
         comes around we want to kill the current process and start
         another.
    2. takes infinite time
      a. The only reason that a reference to the server process would be
         empty is if the server hasn't started in our program yet, or if the
         server is in the midst of being restarted.
III. Modeling state in a functional program
  A. Haskell has a few ways to maintain state. One way is through
     `MVar`s. We'll use them because Haskell's most popular file watching library,
     `fsnotify`, already has a hard dependency on them.
  B. `MVar` contains state. It wraps any data type and provides an
      interface to get and change the data inside them.
IV. Using `MVar`s to contain our state.
  A. Boot.
    1. Creating the containers for the states. `newEmptyMVar`.
  B. The server process in Boot
    1. We need to put a reference to the server process in the `MVar`
       that will contain our server state. You put stuff into `MVar`s
       with `putMVar`. (example).
  C. The build progress in build-and-serve program
    1. The first thing we do in build-and-serve is see if there's a
       build currently running. We would do that by inspecting the state
       of the `buildState` `MVar`. `MVar`'s interface for inspecting
       whether something is contained within it is `tryTakeMVar`. It
       returns a `Maybe a`, `a` being the thing you put inside of
       the `MVar`. If it's a `Just ProcessHandle`, we'll kill the `ProcessHandle`.
       If it's `Nothing`, we won't do anything.
    2. The second thing we do is kick off a build in a new process.
       Immediately after, we should put that process in our `buildState`
       with `putMVar`. (example). `buildState` will always be empty at
       this point, because we've already removed any possible existing
       process with `tryTakeMVar`.
  D. The server process in the build-and-serve program
    1. If and only if the build process wasn't killed, we want to
       re-serve our server. This means we'll have to take the server
       process out of the `serverState` and kill it. We already have a
       mechanism to take a `ProcessHandle` out of an `MVar` and maybe
       kill it, depending on wheter it exists, from dealing with killing
       a potential build process. We'll use the same mechanism; no need
       to duplicate work. (example)
    2. Once the old server is killed, we'll spawn a new server process and use
       `putMVar` to put it into the `serverState`.
V. Performing actions on processes contained in `MVar`s
  A. Spawning processes
    1. `spawnProcess` takes a string for the command-line. It evaluates
       that string, while returning `IO ProcessHandle`. The
       `ProcessHandle` is an interface to perform operations on a
       the process.
    2. We'll use this to create builds and servers.
  B. Terminating processes
    1. `terminateProcess` takes a `ProcessHandle` and sends a `SIGTERM`
       (or its Windows equivalent, TerminateProcess) signal to it.
    2. We'll use this to terminate the server and build processes in the
       build-and-serve program.
  C. Waiting for processes
    1. `waitForProcess` takes a `ProcessHandle` and returns its
       `ExitCode` when the process finishes.
    2. We'll use this to see whether the build process completed
       successfully. If it does, it returns an `ExitSuccess` value. If
       it doesn't, it returns an `ExitFailure`. We'll use those values
       to determine whether or not to restart the server.

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
functional language. On closer inspection, this program contains
multiple states that must be accessed in at different points in the
program. State is handled explicitly in purely functional languages,
which is very different from imperative langagues. So let's take a
closer look at this program and see how parts of it may be implemented
in Haskell.

## Levels of the program
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
there *is* a build process occuring right now.

### Server process
