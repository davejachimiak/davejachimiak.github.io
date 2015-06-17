---
layout: post-no-feature
title:  "A Functional Implementation of an Abstract Development Server"
date:   2015-05-19
description:  Let's implement an abstract development server using functional thinking.
categories: development software-design servers
---
### pulling out main ideas...
* modeling the design as a functional program
* maintaining state in a functional program
* explicit state
* what are the natures of the states?
** build process - finite but unknown time
** server process - infinite time (until the main program's process is
                    interuppted)
* why do we need to maintain a reference to the build process?
* what are the levels of the program? (boot, file-watcher, sub-program)
* how do we keep a reference to a system-level process in Haskell?
* how to listen to processes and act differently according to their exit code?

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
  A. Build rocess
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

[This post](2015/05/designing-an-abstract-development-server.html)
determined that a design for an abstract development server would do the
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
innate complexity. There are two points of state that we must consider
in order to cancel builds and restart servers.

We must somehow keep track of those processes. There are many ways we
can do this 

## Purely Functional Programs

Programs need to be able to listen and talk to the world around them in
order to be useful. This means that they require input and output in
order to listen and talk to the world. It's possible that the world
needn't give any information to a program, but then the program would
necessarily return the same result every time. It's also possible that
the program needn't talk to the world around it, but then you wouldn't
be able to persist or output any calculations.

In order to keep our program "pure", we must treat references to input
and output as special cases. In other words, we must explicitly declare
when we're dealing with input, output, and state in our program. This
has an added benefit of making our program more usable; at a glance, we
know the parts of our program that make it unpure.

## The file watcher

By its very nature, a file watcher is an event loop. It sits in a
process, waiting for something to happen – a file event. When something
happens, it runs its program, then waits until something happens again.
And on and on.

## The program's states

There are two states that the larger program needs to contain; the
server process and the build process. We need to maintain the server
process because we need to kill it when we're ready to serve and
maintain a new server process. We need to keep track of any build
processes because if another build is triggered, we need to kill the old
one and maintain the new one.

The file watcher may be thought of as a stateful thing, as its a long
running process. But since the program it triggers it is technically
the same thing every time, it need not be managed as something that
changes.

## Storage of the references to the states

Functional languages have nifty and safe ways to keep track of data that
changes over time. Haskell in particular has a couple of ways, depending
on the what kinds of operations you'd like to have available as you
operate on the container of the state. We'll be using `MVar`s in the
examples below.

State in functional programming is a big deal. The whole point of a
function is to do something to values passed into it, return some value
and then perhaps do something with that value in another function.
There's no obvious need to maintain state in this scenario. Unless, of
course, you're dealing with a program that triggers a sub-program every
time a file event happens. In this case – which is our case –, we need
to maintain the state of two processes so that if that sub-program can
evaluate its logic based on those states. The sub-program is thus
dependent on a couple of pieces of data that change.

## Down into the details

Our program begins with two tasks before it kicks off the file watcher.

1. compile the server executable
2. serve that executable

In terms of keeping track of processes, we'd need to at least keep track
of the server process (2.) before we even start the file watcher. But
we should initialize both before we start the file watcher; the initial
states should be initialized before the file watcher so that the file
watcher itself isn't initializing a new state for every file event.

So, we initialize the both states before starting the file watcher so
that we can pass the states into it.

Once the server is being served, and we've stored a reference to it in
an `MVar`, we can pass that server state and a place to hold future
build states to the file watcher.

## What happens to the states in the file watcher?

The file watcher merely watches if a file changes and triggers a
sub-program. This sub-program is the meat and potatoes of the main
program.

In this sub-program, we first see if there's any build currently
referenced in the build state. If there is, we kill that build and
remove it from the state. If no build is there, we move on to the next
step.

Next, we start a new build process and store it in the build state.

Finally, listen for the build state to finish. If that process finishes
with a successful area code (0), kill the current server process stored
in the server state, start a new server process based on the newly-built 
executable, and store that server process in the server state. If that
process finishes with a unsuccesful area code (1-?), to nothing.

The reason we listen to the build state and its exit code is that we're
potentially killing the the build in the build state in the first step
of this sub-program. We don't want to start a new server if the
executable wasn't fully built.
