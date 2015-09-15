---
layout: post-no-feature
title: "A Tip for Testing the Speed of Your Javascript App"
date: 2015-09-14
categories: javascript DOM performance
---

At least one [layout
(reflow)](http://taligarsiel.com/Projects/howbrowserswork1.htm#Layout)
should occur for every user interaction. Ensure this by triggering user
events in your test code with asynchronous callbacks.

Imagine you have some code that appends children to an element when a
user clicks another element.

{% highlight html %}
<html>
<body>
  <div id="root"></div>
  <div id="append-child"></div>
  <script>
    var root = document.getElementById('root');

    document.getElementById('append-child').onclick = function() {
      var div = document.createElement('div');
      div.textContent = "hi";

      root.appendChild(div);
    }
  </script>
</body>
</html>
{% endhighlight %}

To speed-test how clicking `#append-child` many times manipulates the DOM, don't do this:

{% highlight javascript %}
var startTime = new Date();

for (i=0; i < 100; i++) {
  document.getElementById('append-child').click();
}

console.debug("Inaccurate time: " + (new Date() - startTime));
{% endhighlight %}

<script>
  document.onready = function() {
    var rootBad = document.getElementById('root-bad');
    var rootGood = document.getElementById('root-good');

    document.getElementById('append-child-bad').onclick = function() {
      var div = document.createElement('div');
      div.textContent = "hi";
      div.style.float = "left";

      rootBad.appendChild(div);
    }

    document.getElementById('append-child-good').onclick = function() {
      var div = document.createElement('div');
      div.textContent = "hi";
      div.style.float = "left";

      rootGood.appendChild(div);
    }
  }

  function tryBad() {
    var startTime = new Date();

    for (i=0; i < 100; i++) {
      document.getElementById('append-child-bad').click();
    }

    console.debug("Inaccurate time: " + (new Date() - startTime));
  }

  function tryGood() {
    var startTime = new Date();

    function appendChildren(count) {
      setTimeout(function() {
        document.getElementById('append-child-good').click();

        if (count === 100) {
          return console.debug("Accurate time: " +
            (new Date() - startTime));
        } else {
          appendChildren(count + 1);
        }
      }, 0);
    }

    appendChildren(1);
  }
</script>

<a href="javascript:void(0);" onclick="tryBad()">Try this junk measurement.</a>

<div id="root-bad" style="display:inline-block;"></div>
<span id="append-child-bad" style="left:-10000px;position:absolute;"></span>

<p style="">Instead, do this:</p>

{% highlight javascript %}
var startTime = new Date();

function appendChildren(count) {
  setTimeout(function() {
    document.getElementById('append-child').click();

    if (count === 100) {
      return console.debug("Accurate time: " +
        (new Date() - startTime));
    } else {
      appendChildren(count + 1);
    }
  }, 0);
}

appendChildren(1);
{% endhighlight %}

<a href="javascript:void(0);" onclick="tryGood()">Try this better measurement.</a>

<div id="root-good" style="display:inline-block;"></div>
<div id="append-child-good" style="left:-10000px;position:absolute;"></div>

Consider the first example. Most modern browsers append 100 “hi”s at the
same time. The time printed in the console would be way too short. This
is because those browsers gather DOM writes in synchronous code. They
execute a layout after it’s completely evaluated. Some reads of node
properties, like `element.clientHeight`, will trigger layouts in
synchronous code. More below.

The second example executes a layout every time our simulated user
clicks the `#add-element` element. It gives us a time we can compare to a
speed test of another implementation of the same functionality.

This isn’t to say that an event should execute only one layout should
for each user interaction. Some bad DOM performance occurs when a single
user action triggers many layouts. You’ll want to account for all those
layouts in your measurement. You’ll see many layouts executed for
certain reads of nodes. For example, calling `clientHeight` on some
element between 2 DOM writes will trigger two layouts. The first is for
the browser to figure out the client height of the element after the
first write. The second is to display the result of the second write.

### Further reading

* [How browsers
  work](http://taligarsiel.com/Projects/howbrowserswork1.htm)
* [Repaints and Reflows: Manipulating the
DOM](http://blog.letitialew.com/post/30425074101/repaints-and-reflows-manipulating-the-dom)
