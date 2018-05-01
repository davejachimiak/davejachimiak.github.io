---
layout: post-no-feature
title: "The Elements of Computing Systems: Drawing Pixels on the Screen"
date: 2018-04-30
categories: languages, graphics, Jack
---

<a href="https://www.amazon.com/Elements-Computing-Systems-Building-Principles/dp/0262640686">
  <img style="width: auto;" src="/images/the-elements-of-computing-systems.jpg" />
</a>

Like I said in my post about [memory management on the Jack
platform](/2018/04/jack-alloc.html), I love this book.

The Elements of Computing Systems has you build a computer from scratch. By the
end you build a platform called Jack, which includes some virtual hardware, an
assembler, a virtual machine, a compiler for the Jack language. It's a lot of
fun.

<hr>

When you write Jack programs that manipulate the screen, you can see the result
of the program in the VM emulator.

![vm-emulator-with-screen](/images/vm-emulator-with-screen.png)

The screen is a "device" to the Jack platform. The computer only provides the
memory space for the device to look at. The responsibiilty for changing what the
screen looks like based on the memory map belongs to the device. So a contract
between the computer and the device needs to exist so that graphics can be
displayed correctly. Specifically, both device designers and Jack programmers
need to have a mutual understanding of the answer to the question: what do the
1s and 0s mean in screen's address space?

The contract is:

* The address mappings for the screen device are from 16384 to 24575, inclusive.
(The virtual hardware includes a 23476-address memory space with 16-bit words.)
* Each bit in those addresses corresponds to a pixel on the screen.
16-bit words means there are 131,072 pixels. There are 512 pixels on the x-axis
and 256 pixels on the y-axis.
* If a bit is "on", or 1, the screen device paints the corresponding pixel black.
If it's "off", or 0, the screen device paints it white.

<hr>

We need to keep track of some global variables across the lifespan of the
program.

```
class Screen {
  static int SCREEN, COLOR, BLACK, WHITE;

  function void init() {
    let SCREEN = 16384;
    let BLACK = 1;
    let WHITE = 0;
    let COLOR = BLACK;
    return;
  }

  function void setColor(boolean b) {
    // ...
  }

  function void drawPixel(int x, int y) {
    // ...
  }
}
```

In order to know what color to draw `Screen` also keeps track of what the
current `COLOR` is. We set the default to black.

```
function void setColor(boolean b) {
  if (b) {
    let COLOR = BLACK;
  } else {
    let COLOR = WHITE;
  }

  return;
}
```

`Screen` has a function called `setColor(boolean b)`. If `true` is passed in, it
changes the color to black. Otherwise it changes the color to white.

The current `COLOR` determines how we'll calculate the changes to screen
addresses in `drawPixel`. We'll explore that more below.

### `Screen.drawPixel(int x, int y)`

`drawPixel` takes an (x,y) coordinate. (0,0) is in the upper left corner of the
screen. (0,1) is one pixel below, and (1,0) is one pixel to the right. (511,255)
is the lower right corner of the screen. Any value for x and y that's greater
than 511 and 255 is invalid, but behavior for those is unspecified.

The first challenge is to find the address that has the mapping to the pixel.
The formula can be described as: `SCREEN + (y * 32) + (x / 16)`. Let's break
that down.

Some bit in some address maps to pixel (x,y). The bit is located in some address
that has 16 bits. As we move from address 16384 to the last address — 24575 —,
we're moving from left to right on the screen and down the screen when we come
up against the pixel limit in a row. Pixel (0,0) is in address 16384 and so is
pixel (0,15). Pixel (1,0) is in address 16416, and so is (1,15). Since every row
has 512 pixels and there are 16 pixels per address, we know that there are 32
addresses represent each row's pixels. So we can multiply `y * 32` to get the
first address of the pixel row. To get the exact address we want to change in
`drawPixel`, we divide `x` by the number of bits per address and add that answer
to the first address in the row.

The next challenge is finding the bit in the address that represents the pixel
(x,y). This one's easy — we just take the remainder of `x / 16`. Jack doesn't
have a modulo operator, nor a modulo function in the `Math` library. Were a
modulo implementation to exist, it should go in the `Math` library. But since
we're just focusing on `Screen` right now and the `Math` library isn't available
to be "opened up," we'll just create a `Screen.modSixteen(int dividend)`.

Finally, we need to somehow modify the address so that we only change the pixel
(x,y). We'll setup a modifier that we can apply the the address. The modifier
will be a value who's corresponding bit is 1 and the rest are zero.

Numbers in Jack are stored as 16-bit signed [two's
complement](https://en.wikipedia.org/wiki/Two%27s_complement). With two's
complement binary representations, we can shift bits to the left by multiplying
the number by 2. Using that, we can get the modifier by calculating the exponent
of 2 to the corresponding, zero-based bit position. Let's say that we find our
corresponding bit to be in the 0th position. From there we can calculate 2 to
the 0 and get 1. 1 in 16-bit two's complement binary is `0000000000000001`. If
we find that our corresponding bit is in the 4th position, can calculate 2 to
the 4 and get 16. 16 in two's complement binary is `0000000000010000`.

(Keep in mind that how we read binary numbers is reversed from its representation
in the machine. So while 1 in binary is read by us as `0000000000000001`, it's
actually stored as `1000000000000000`.)

How we'll use that modifier depends on the current color. The challenge is to
modify the address to change the pixel (x,y) but not disturb the other pixels in
the address. We can do this with some clever bit arithmetic.

If `COLOR` is `BLACK`, we apply the `OR` (`|`) operation to the address and the
modifier. Applying the modifier to the address with `|` has the effect of
idempotently changing only the modifier's flipped bit. For example, if the
binary value of the address is `1111111100000001` and our modifer is
`0000000000000010`, applying the modifier with `OR` results in
`1111111100000011`. Now the 2nd pixel in the address has been painted black
while leaving everything else the same.

If `COLOR` is `WHITE`, we apply the `AND` (`&`) operation to the address and a
negated version of the modifier. Doing this has the effect of idempotently
changing only the negated modifier's 0 bit in the address. For example, if the
binary value of the address is `1111111100000001` and our modifier is
`0000000000000001`, applying the negated modifer — `111111111111110` with `AND`
results in `1111111100000000`. Now the 1st pixel in the address has been painted
white while leaving the other bits the same.

<hr>

Other function in `Screen` — `drawLine()` and `drawCircle()` — use `drawPixel()`
to draw pixels while determining which pixels to draw.
