---
layout: post-no-feature
title: "The Elements of Computing Systems: Memory Management in Jack"
date: 2018-04-29
categories: languages
---

<a href="https://www.amazon.com/Elements-Computing-Systems-Building-Principles/dp/0262640686">
  <img style="width: auto;" src="/images/the-elements-of-computing-systems.jpg" />
</a>

I love this book.

The Elements of Computing Systems has you build a computer from scratch. By the
end you build a platform called Jack, which includes some virtual hardware, an
assembler, a virtual machine, and a compiler for the Jack language. Fun!

It also has an operating system. It isn't what you would consider a full-blown
operating system. It's rather a collection of libraries with system calls that
are available to user programs. There's no process scheduler, for example.

One of the libraries is `Memory`. It has functions that make memory management
easier for the Jack programmer: `alloc(int size)` and `deAlloc(int block)`. The
book’s authors suggest a couple ways to manage memory. One suggestion is simple:
give the `alloc` caller a pointer to the next available memory block and do
nothing in `deAlloc`. This would speed up memory allocation in Jack programs,
but you'd run out of memory pretty fast.

The authors suggest another way that's more useful. The idea is to maintain a
linked list of available memory segments called `freeList`. We grab free
segments from the `freeList` in `alloc` and insert segments back in `deAlloc`.
This recycles memory so we can reallocate it later.

To make `freeList` work across `alloc` and `deAlloc`, we need a place to store a
segment's length and a pointer to the next segment. So we'll store the segment's
length in its first address and a pointer to the next segment in its second
address. We know we're at the end of the list if there's no pointer in a
segment's second address.

The language you'll see below is Jack. It feels like Java, but it's a lot
simpler.

```
class Memory {
  static int freeList, HEAP_BASE, HEAP_LIMIT, SIZE, NEXT_POINTER;

  function void init() {
    let SIZE = 0;
    let NEXT_POINTER = 1;
    let HEAP_BASE = 2048;
    let HEAP_LIMIT = 16383;
    let freeList = HEAP_BASE;
    let freeList[SIZE] = HEAP_LIMIT - HEAP_BASE;
  }
}
```

The heap ranges from addresses 2048 to 16383 on the Jack platform. We initialize
`freeList` as a static variable with its value being the `HEAP_BASE`. (Static
variables are available to the class' class-level functions.) To record the
segment's size in its first address, we set `freeList[0]` to `HEAP_LIMIT -
HEAP_BASE`. (Jack gives direct access to memory addresses through a little hack:
you can set a variable to a pointer and use array-style syntax to refer to
addresses relative to the pointer.)

### `Memory.alloc(int size)`

User code should call `alloc` when it wants some memory for itself. `alloc`
iterates through segments in the `freeList` to look for a block to return to the
caller.

```
function int alloc(int size) {
  var bool defragged;
  var int currentFreeSegment, block, newFreeSegment, previousFreeSegment;

  let defragged = false;
  let currentFreeSegment = freeList;

  while (~block) {
    if (currentFreeSegment[SIZE] > size) {
      // ...
    }

    // ...
  }

  // ...
}
```

A segment in `freeList` is available if the `size` passed in is less than the
segment length.

```
  if (currentFreeSegment[SIZE] > size) {
    if (currentFreeSegment = freeList) {
      if (currentFreeSegment[NEXT_POINTER]) {
        let freeList = currentFreeSegment[NEXT_POINTER];
      } else {
        let freeList = currentFreeSegment + size + 1;
        let freeList[SIZE] = currentFreeSegment[SIZE] - size - 1;
      }
    } else {
      // ...
    }

    let block = currentFreeSegment + 1;
  }
```

The first segment of `freeList` is a special case — `freeList`'s pointer needs to
change if the first segment is allocatable. If the first segment has a
`NEXT_POINTER`, we just set the `freeList` pointer to the `NEXT_POINTER`. If
not, we calculate that the next free segment is at `freeList + size + 1`, we set
`freeList` to that, and update its `SIZE`.

```
  if (currentFreeSegment[SIZE] > size) {
    if (currentFreeSegment = freeList) {
      // ...
    } else {
      if (size < (currentFreeSegment[SIZE] - 2)) {
        let newFreeSegment = currentFreeSegment + size + 1;
        let newFreeSegment[SIZE] = currentFreeSegment[SIZE] - size - 1;
        let previousFreeSegment[NEXT_POINTER] = newFreeSegment;
      } else {
        let previousFreeSegment[NEXT_POINTER] = currentFreeSegment[NEXT_POINTER];
      }
    }

    let block = currentFreeSegment + 1;
  }
```

We need to handle two possibilities for all other segments in the `freeList`.
One is that `size` is less than `currentFreeSegment[SIZE]` by 3 or more. And the
other is that `size` less than `currentFreeSegment[SIZE]` by 2 or less.

For the first possibility, we should create a new segment by figuring where that
`newFreeSegment` should start, calculating what it's size should be, and inserting it
into the `freeList`. We do that by pointing to the `newFreeSegment` in
`previousFreeSegment[NEXT_POINTER]`.

For the second possiblity, we should just update `previousFreeSegment[NEXT_POINTER]`
to `currentFreeSegment[NEXT_POINTER]`. This is because there's no room to create a
new segment in the `freeList` — we need at least two addresses to hold a `SIZE`
and a `NEXT_POINTER`.

```
  if (currentFreeSegment[SIZE] > size) {
    // ...
  } else {
    if (currentFreeSegment[NEXT_POINTER] = null) {
      if (defragged) {
        do Sys.error(OOM);
      } else {
        do Memory.defrag();
        let defragged = true;
        let currentFreeSegment = freeList;
      }
    } else {
      // ...
    }
  }
```

If we can’t allocate the current segment but there is no next segment, it means
we need to `defrag()` or we’re out of memory. We `defrag` the `freeList` if we
haven’t defragged before. But if we have defragged before, we return an out of
memory error. We’ll talk about `defrag()` below, where we'll combine contiguous
segments in `freeList` to potentially create a segment that's bigger than
`size`.

```
  if (currentFreeSegment[SIZE] > size) {
    // ...
  } else {
    if (currentFreeSegment[NEXT_POINTER] = null) {
      // ...
    } else {
      let previousFreeSegment = currentFreeSegment;
      let currentFreeSegment = currentFreeSegment[NEXT_POINTER];
    }
  }
```

If there _is_ another segment to check, we shift `previousFreeSegment` and
`currentFreeSegment` to handle the next segment, and we continue looking for
allocatable segments.

```
function int alloc(int size) {
  // ...

  while (~block) {
    if (currentFreeSegment[SIZE] > size) {
      // ...

      let block = currentFreeSegment + 1;
    } else {
      // ...
    }
  }

  let block[-1] = size + 1;
  let block[0] = null;

  return block;
}
```

Before we return the `block` pointer to the caller, we set the size of the
segment in the address before the block pointer. We set that address to `size +
1` so that we can account for the address that's holding the size.

We also ensure that the block's first address is `null`; it's where we may have
previously stored `currentFreeSegment[NEXT_POINTER]`.

### `Memory.defrag()`

`defrag` combines contiguous segments in `freeList`.

```
function void defrag() {
  var int currentFreeSegment, nextFreeSegment;

  let currentFreeSegment = freeList;
  let nextFreeSegment = currentFreeSegment[NEXT_POINTER];

  while (~(nextFreeSegment = 0)) {
    if ((currentFreeSegment + currentFreeSegment[SIZE]) = nextFreeSegment) {
      let currentFreeSegment[SIZE] = currentFreeSegment[SIZE] + nextFreeSegment[SIZE];
      let currentFreeSegment[NEXT_POINTER] = nextFreeSegment[NEXT_POINTER];
      let nextFreeSegment[SIZE] = null;
      let nextFreeSegment[NEXT_POINTER] = null;
    }

    let currentFreeSegment = nextFreeSegment;
    let nextFreeSegment = currentFreeSegment[NEXT_POINTER];
  }

  return;
}
```

If `currentFreeSegment + currentFreeSegment[SIZE]` equals
`currentFreeSegment[NEXT_POINTER]`, we have two contiguous memory segments.

When two contiguous memory segments are detected, we:

1. update the `currentFreeSegment`'s size to that of the two segments,
2. update `currentFreeSegment`'s `NEXT_POINTER` to that of the `nextFreeSegment`, and
3. nullify addresses that kept `nextFreeSegment`'s information. 

### `Memory.deAlloc(int block)`

`deAlloc` iterates through the `freeList` to find a place to put the `block`'s
`segment`.

```
function void deAlloc(int block) {
  var int i, segment, segmentLength, oldFreeList, currentFreeSegment, oldNextSegment;

  let i = 1;
  let segment = block - 1;
  let segmentLength = segment[0];

  while (i < (segmentLength - 1)) {
    let segment[i] = null;
    let i = i + 1;
  }

  // ...

  return;
}
```

The `block`'s `segment` that starts `block[-1]`. Since we’ve maintained the
length the the block at `segment[0]` (or `block[-1]`), we know we must nullify
addresses `segment[1..(segment[0] - 1)]`.

The trickiest part here is correctly inserting the `segment` back into the
`freeList`. There are three possibilities: `segment` is before `freeList`,
`segment` is in between entries in the `freeList`, and `segment` is after
`freeList`'s last segment.

```
function void deAlloc(int block) {
  // ...
 
  let segment = block - 1;

  // ...

  if (segment < freeList) {
    let oldFreeList = freeList;
    let freeList = segment;
    let freeList[NEXT_POINTER] = oldFreeList;
    return;
  }

  // ...
}
```

When `segment` is before `freeList`, we note the value of `freeList` in some
variable like `oldFreeList`, update the `freeList` to be `segment`, and update
the next segment pointer which is now `freeList[NEXT_POINTER]`, to
`oldFreeList`.

Sometimes `segment` will be between entries in the `freeList`. We’ll know this
when `segment` is between some `currentFreeSegment` and its `NEXT_POINTER`. If
`currentFreeSegment < segment > currentFreeSegment[NEXT_POINTER]`, then we know we
have to insert the newly freed `segment` there.

```
  let segment = block - 1;

  // ...

  if (segment < freeList) {
    // ...
  }

  let currentFreeSegment = freeList[NEXT_POINTER];

  while ((currentFreeSegment[NEXT_POINTER] < segment) & currentFreeSegment[NEXT_POINTER]) {
    let currentFreeSegment = currentFreeSegment[NEXT_POINTER];
  }

  if (currentFreeSegment[NEXT_POINTER] > 0) {
    // we're in between segments in the freeList
    let oldNextSegment = currentFreeSegment[NEXT_POINTER];
    let currentFreeSegment[NEXT_POINTER] = segment;
    let segment[NEXT_POINTER] = oldNextSegment;
  } else {
    // we're at the end of the list
    // ...
  }
```

To insert a `segment` in between existing `freeList` segments, we note the value
of `currentFreeSegment[NEXT_POINTER]` into some variable like `oldNextSegment`,
update `currentFreeSegment[NEXT_POINTER]` to be `segment`, and update
`segment[NEXT_POINTER]` to be `oldNextSegment`.

```
  if (currentFreeSegment[NEXT_POINTER] > 0) {
    // ...
  } else {
    let currentFreeSegment[NEXT_POINTER] = segment;
  }
```

Finally, we're at the end of the `freeList` if we run into a segment that has a
null `NEXT_POINTER`. That means we can just set `currentFreeSegment[NEXT_POINTER]`
to `segment`.

<hr>

I rarely work in situations where I have to manage memory manually, let alone
work directly on the heap. So this was a lot of fun to think through.

I put a full version of the `Memory` library
[here](https://gist.github.com/davejachimiak/daede4ef2af420caed86f3da0237ea23).
