---
layout: post-no-feature
title: "Avoid Negative Variable Names"
date: 2018-05-21
categories: code
---

Avoid negative variable names. They make code harder to read.

(Related: [--noooooooooooooooooo](http://davej.io/2018/04/nooo.html))

A negative variable name holds a boolean that tells of whether we should *not*
do something. In imperative code they're usually evaluated in conditional
expressions in logical statements, like `if` and `while`.

(psuedo-code)
 
```
if do_not_do_it
else
  it.execute()
end
```

`do_not_do_it` tells whether we shouldn't do it.

In certain contexts, I can see how telling that we should *not* do something
makes sense as you're writing the code. Maybe *not* doing something is
overriding the default, which is to do something, and *not* doing something is
way more important than actually doing it.

```
do_not_do_it = false

if argv.include?("--dont-you-fucking-do-it")
  do_not_do_it = true
end

// ...

return if do_not_do_it

it.execute()
```

When other people start extending the code, chances are they'll want to do other
things with that boolean, like negate it.

```
if !do_not_do_it
  log("we're gonna do it, but first we need to prepare some stuff.")
  some_stuff.prepare()
end
```

Now we're negating a value at runtime that has a negative name: "If we're not
not doing it, then we're gonna prepare some stuff." To the mind we're not
dealing with a positive and a negative anymore.  We're dealing with a negative
and a double negative. By doing this you force people reading this code to spend
extra brain cycles to flipping the double negative into a positive. It's worth
considering the effect negative names have on your team and your company over
time.

Just use a positive name.

```
do_it = true

if argv.include?("--dont-you-fucking-do-it")
  do_it = false
end

// ...
// ...

if do_it
  it.execute()
end
```

Seems easier to follow to me. 

Using more descriptive names is good if the negative-ness of the variable is
crucial to carry on through the program. For example, if we're not doing
something that's the norm, then we're *skipping* it.

```
skip_it
```
