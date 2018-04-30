---
layout: post-no-feature
title: "The Elements of Computing Systems: Drawing Pixels on the Screen"
date: 2018-04-30
categories: languages, graphics, Jack
---

<a href="https://www.amazon.com/Elements-Computing-Systems-Building-Principles/dp/0262640686">
  <img style="width: auto;" src="/images/the-elements-of-computing-systems.jpg" />
</a>

I. Jack platform background
  A. How many memory addresses?
  B. 16-bit addresses
  C. Jack language
    1. Compiles to Jack VM code, which compiles to Jack assembly, which compiles
       to Jack binary.
    2. You can manipulate memory addresses in Jack
      a. Declare a pointer to an address in a variable
      b. Use array-style syntax to access the address relative to the pointer.
II. Jack screen
  A. Where you can see it in the nand2tetris software
    1. Upper right hand corner.
    2. [photo]
  B. It's a "device"
    1. The Jack platform only provides the memory space for the device to look
       at.
    2. The responsibility for changing what the screen looks like based on the
       memory map is belongs to the device.
    3. But a contract between the computer and the device needs to exist so that
       graphics can be displayed correctly: specifically, both the device
       designers and Jack programmers answer the question: what do the 1s and 0s
       mean in the address space for the screen?
  B. Address mappings
    1. address 16384 to 24575, inclusive
  C. Each bit in those addresses corresponds to a pixel on the screen.
    1. That means there are 131,072 pixels.
      a. 512 on the x-axis
      b. 256 on the y-axis
    2. colors
      a. if a bit is 1, the screen device maps to black
      b. 0 is white
III. Drawing pixels to the screen
  A. API for `drawPixel(int x, int y)`
    1. Takes an (x,y) coordinate
    2. (0,0) is in the upper left corner. (0,1) is one pixel below (0,0). (1,0)
       is one pixel to the right of (0,0).
    3. x's maximum is 512; that's how many columns the screen has.
    4. y is maximum 256; that's how many rows the screen has.
  B. Changing the pixel.
    1. Finding the address where the pixel is.
      a. Some bit in the Jack memory maps to the pixel (x,y).
      b. The bit is located in some address that has 16 bits.
    2. Finding the corresponding bit in the address.
    3. The current `COLOR` determines how to modify the address.
      a. WHITE
        1. 
IV. Uses of `drawPixel()`
  A. `drawLine()`
  B. `drawCircle()`
  C. `Output.printChar`
    1. Wait, does `printChar` actually use `drawPixel`?
