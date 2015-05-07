---
layout: post-no-feature
title:  "An Abstraction For Currying In Ruby"
date:   2014-01-14
categories: currying functional-programming
---
This post works towards an abstraction for currying in Ruby. It begins with an illustration of currying with pseudocode, then reviews the way Ruby does currying with procs. By thinking through the properties curried functions, it  results in a [`Curryable` mixin](https://gist.github.com/davejachimiak/8262343) that can be extended on a singleton class.

## Currying

Here's the first sentence of the [Wikipedia entry for currying](http://en.wikipedia.org/wiki/Currying)<sup><a id="curry-reference-1" href="#curry-postscript-1">1</a></sup>:

> In mathematics and computer science, currying is the technique of transforming a function that takes multiple arguments (or a tuple of arguments) in such a way that it can be called as a chain of functions, each with a single argument (partial application)...

Take this function in pseudocode that adds three numbers:

```
addThreeNumbers(x, y, z) = { x + y + z }
```

Call `addThreeNumbers(1, 1, 1)` and it would return 3. But how is the function applied to those arguments? `addThreeNumbers` as a Ruby method would use the arguments at any time they were referenced within it. It would raise an error if it were called with less than 3 arguments, given no arguments were optional.

```ruby
def add_three_numbers x, y, z
  x + y + z
end

add_three_numbers 1, 1, 1
#=> 3
add_three_numbers
#=> ArgumentError: wrong number of arguments (0 for 3)
add_three_numbers 1
#=> ArgumentError: wrong number of arguments (1 for 3)
add_three_numbers 1, 1
#=> ArgumentError: wrong number of arguments (2 for 3)
```

`add_three_numbers` would act a lot differently if it were a curried function.

Let's curry `addThreeNumbers` in our pseudocode. This means we're going to transform it to accept one argument. The application of the function to that argument (the first) will return another function that accepts one argument and holds a reference to the argument from the previous application. Applying that function to an argument (the second) returns yet another function that accepts one argument and holds references to the arguments from the previous function applications. Applying the function to yet another argument (the third and final) returns the sum of all three arguments.

```
curry(addThreeNumbers) =
  f(x) {
    g(y) {
      h(z) {
        x + y + z
      }
    }
  }
```

If open-closed parentheses were left-associative function applicators in our pseudocode, then `curry(addThreeNumbers)(1)(1)(1)` would return 3.<sup><a id="curry-reference-2" href="#curry-postscript-2">2</a></sup> Furthermore, we could apply the function to less than 3 arguments and it would return a partially applied function. That partially applied function could be reused by applying it to more arguments later on.

```
addTwoToYAndZ = curry(addThreeNumbers)(2)
addTwoToYAndZ(8)(1)
#=> 11
addTenToZ = addTwoToYAndZ(8)
addTenToZ(10)
#=> 20
```

## an aside: Currying with lambdas in Ruby > 1.8

Ruby already allows for currying with lambdas. Calling `#curry` on a lambda proc returns a curried version of the proc.

```ruby
add_three_numbers = lambda { |x, y, z| x + y + z }.curry
#=> #<Proc:0x007fdc02494950 (lambda)>
add_three_numbers.(1).(1).(1)
#=> 3
add_one_to_y_and_z = add_three_numbers.(1)
#=> #<Proc:0x007fdc024a7230 (lambda)>
add_one_to_y_and_z.(8, 2)
#=> 11
add_ten_to = add_one_to_y_and_z.(9)
#=> #<Proc:0x007fdc024bfb00 (lambda)>
add_ten_to_z.(1)
#=> 11
```

You probably noticed above that Ruby allows for multiple arguments to be called on curried lambdas. In our mixin abstraction for currying, we'll stick to applying functions to one argument at a time.

## A mixin abstraction for currying

What would an abstraction for currying look like in Ruby? Given a way to define a function, how might a `Curryable` mixin for a singleton be implemented? With it, we could do this:

```ruby
class AddThreeNumbers
  extend Curryable

  def_function { |x, y, z| x + y + z }
end

AddThreeNumbers.(1).(2).(3)
#=> 6
add_one_to_x_and_y = AddThreeNumbers.(1)
#=> #<Proc:0x007f8bdbc6e030 (lambda)>
add_one_to_x_and_y.(8).(2)
#=> 11
```

Thinking about the properties of curried functions will help us toward an abstraction. Here are a few:

* The length of the curried function's function chain is equal to the function's arity<sup><a id="curry-reference-3" href="#curry-postscript-3">3</a></sup>.
* A function returns a partially applied function if it is applied to any number of arguments less than its arity. A partially-applied function has references to previous arguments.
* The function chain terminates when it is applied to the last argument and returns the result of the defined function.

A few things are clear. First, we need to allow for defining the function. Then we need a mechanism to generate a function chain that has the length of the function's arity. Finally, we need to decide on a function applicator which will act as a mechanism to apply the function chain to each argument.

Allowing for the definition of a function is straightforward: just save a reference to the defined function on an instance variable.

```ruby
module Curryable
  def def_function &block
    @function = block
  end
end
```

Generating a function chain is less straightforward. Let's take a look at a function chain for `add_three_numbers` that uses lambdas.

```ruby
def add_three_numbers
  lambda |x| do
    lambda |y| do
      lambda |z| do
        x + y + z
      end
    end
  end
end

add_three_numbers.(1).(2).(3)
#=> 6
```

Given the properties mentioned above, we can abstract away those repetitive lambdas with a little recursion.<sup><a id="curry-reference-4" href="#curry-postscript-4">4</a></sup>

```ruby
module Curryable
  …
  …
  def function_chain *previous_args
    lambda do |arg|
      args = previous_args.dup << arg

      if terminate_chain?(args.size)
        @function.(*args)
      else
        function_chain(*args)
      end
    end
  end

  def terminate_chain? args_size
    (arity - args_size) == 0
  end

  def arity
    @function.arity
  end
end
```

Finally, we need a function applicator. We'll be using lambda procs in our function chain. Procs are invoked with `#call`, and objects that implement that method get its alias, the epsilon method, for free.<sup><a id="curry-reference-5" href="#curry-postscript-5">5</a></sup> Starting out with the same method will give us a consistent applicator along the function chain.

```ruby
module Curry
  def call arg
    function_chain.(arg)
  end
  …
  …
end
```

Here's the full version:

```ruby
module Curryable
  def def_function &block
    @function = block
  end

  def call arg
    unless @function
      raise StandardError, 'must define a function with def_function'
    end

    function_chain.(arg)
  end

  private

  def function_chain *previous_args
    lambda do |arg|
      args = previous_args.dup << arg

      if terminate_chain?(args.size)
        @function.(*args)
      else
        function_chain(*args)
      end
    end
  end

  def terminate_chain? args_size
    (arity - args_size) == 0
  end

  def arity
    @function.arity
  end
end
```

*Thanks to [John Norman](http://7fff.com/) for reading a draft of this post.*

***
<div class="postscript">
  <p>
    <a id="curry-postscript-1" href="#curry-reference-1">1</a>
      I actually rewrote a definition several times, attempting to pack in all of currying's intricacies. The first sentence of the Wikipedia entry explained it better than I ever could.
  </p>

  <p>
    <a id="curry-postscript-2" href="#curry-reference-2">2</a>
      Calling a Haskell implementation with the same arguments looks like <code>addThreeNumbers 1 1 1</code>. All of Haskell's functions are curried and spaces are left-associative function applicators. In fact, <code>addThreeNumbers 1 1 1</code> is syntactic sugar for <code>((addThreeNumbers 1) 1) 1</code>. The latter version makes clearer how the function is applied to each argument at its particular point in the chain.
    </a>
  </p>

  <p>
    <a id="curry-postscript-3" href="#curry-reference-3">3</a>
      The arity of a function is the number of arguments it accepts.
  </p>

  <p>
    <a id="curry-postscript-4" href="#curry-reference-4">4</a>
      This function chain turns out to reflect a more verbose version of <a href="https://github.com/rubinius/rubinius/blob/f290f2a4f7c59894ea567f9cb05cbe00a31a6654/kernel/common/proc.rb#L95-L107">Rubinius' implementation of <code>#curry</code>.</a>
  </p>

  <p>
    <a id="curry-postscript-5" href="#curry-reference-5">5</a>
      The epsilon method is the method with zero characters. So for an object that implements <code>#call</code> we would get the same result by doing <code>object.call(arg1, arg2)</code> or <code>object.(arg1, arg2)</code>.
  </p>
</div>
