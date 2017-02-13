---
layout: post
title: "How does Node.js manage timers internally"
date: 2017-02-11T00:00:00+02:00
comments: true
tags: [node-js]
---
                                                                                                                      
Timers are an essential part in the Javascript developer tool set. The timers API has been with us a long time, dating back to the days when Javascript was limited to the browser.
This API provides us with the `setTimeout`, `clearTimeout`, `setInterval` and `clearInterval` methods which allows us to schedule code for later execution, either once or repeatedly. 
<!-- more -->
Thanks to Node.js [timer module]( https://nodejs.org/api/timers.html), these methods (along with a few others, such as `setImmediate` and `clearImmediate`) are also available natively 
in node code. On top of user code and 3rd-party libraries using timers, timers are also used internally by the Node.js platform itself. For example, a dedicated 
timer is used with each TCP connection to detect a possible connection timeout. It's very possible that Node.js will have to manage a large number of timers, so it's important that 
the implementation will be highly efficient. In this post I will look at the way Node.js manages these timers under the hood. 

Before tackling the timers implementation, it's worthwhile to examine Node.js implementation of linked lists; they play a large role in the timers implementation, and have some interesting 
parts to them as well. 


Linked Lists
------------

The [relevant code](https://github.com/nodejs/node/blob/master/lib/internal/linkedlist.js) is quite short, and contain all the
methods you'd expect to find in a linked list implementation, such as `create`, `append`, `remove`, `peek`, `shift`, `isEmpty`.

One thing to note is that this is not a "Class", and there is no constructor. Instead, these methods are in fact utility functions 
that operate on an existing object. Two fields are used to manage the links between the nodes: `_idleNext` and `_idlePrev`. 
Perhaps unintuitively, `_idleNext` points to the **older** item, while `_idlePrev` points to the **newer** item.

Let's look at the `init` function:

{% highlight javascript %}
function init(list) {
  list._idleNext = list;
  list._idlePrev = list;
}
{% endhighlight %}

In the root object, `_idleNext` points to the *head* of the list (the oldest element) and `_idlePrev` points to the *tail* of the list (the newest element).
Initially, both point to the list root itself. 

Let's continue and look at the `append` and `remove` functions. Notice that `append` first calls `remove` to ensure the list is unique.

{% highlight JavaScript %}
function append(list, item) {
  if (item._idleNext || item._idlePrev) {
    remove(item);
  }

  item._idleNext = list._idleNext;
  item._idlePrev = list;

  list._idleNext._idlePrev = item;
  list._idleNext = item;
}

function remove(item) {
  if (item._idleNext) {
    item._idleNext._idlePrev = item._idlePrev;
  }

  if (item._idlePrev) {
    item._idlePrev._idleNext = item._idleNext;
  }

  item._idleNext = null;
  item._idlePrev = null;
}
{% endhighlight %}

Notice that at no point we actually traverse the list. Thus, `append` and `remove` are very efficient, $$O(1)$$ operations.

To make this a bit more concrete, lets initialize a new list and append two items. We'll show the objects graph after initialization and 
after each append.

{% highlight JavaScript %}
const list = { name: 'list'};
const A    = { name: 'A' };
const B    = { name: 'B' };

init(list);
init(A);
init(B);

append(list, A);
append(list, B);
{% endhighlight %}

Here's a diagram of the list right after initialization:

![Linked list after initialization](/public/img/node_js_timers/linked_list_1.png)

Here's how it looks after a first append:

![Linked list after first append](/public/img/node_js_timers/linked_list_2.png)

Here's how it looks after a second append:

![Linked list after second append](/public/img/node_js_timers/linked_list_3.png)


From this point, I think it's quite clear how later appends will look.
It is now time to look at timers in more detail.

Timers
------

Each Timer instance is initialized with `msec` argument which determines the delay (in millisecond) until timeout.
It's quite intuitive that if we initialized two timers with the same `msec` argument, then the second timer will timeout either with or after the first.
Node.js uses this and organizes the Timers by indexing them according to their `msec`: all Timers with the same `msec` value will form a linked list, ordered
by creation time (there are actually two indexes, one for `refed` timers and one for `unrefed` timers, but we'll ignore this fact for now. See 
[here](https://nodejs.org/api/timers.html#timers_class_timeout) to read about the difference). 

![Indexing timers by msec](/public/img/node_js_timers/refed_timers.png)

In this example, we initialized 6 timers. `T1`,`T2`,`T3` were initialized with 50 msec, `T4` with 10 msec and `T5,T6` with 200 msec.

Each Timers list is backed up by a `TimerWrap`, which is a wrapper over a `uv_timer_t`, a [native libuv timer type](http://docs.libuv.org/en/v1.x/timer.html). 
Since we know the timers in each list are ordered by non-decreasing timeout, a single `TimerWrap` is enough, as we can reuse it between timeouts.

In pseudo code, the (a bit simplified) strategy on timeout is as follows:

{% highlight python %}
for each timer t in the list L(msec):
  diff := now() - t.start_time
  if diff < msec:
    remaining := max(msec - diff, 0)
    t.start(remaining)
    return
  L.remove(t)
{% endhighlight %}

Once the last timer in a list timeouts, we can remove the list's `msec` key and free the backing native timer.
Same as the `append` and `remove` operations on linked lists, the timeout operation runs in constant time, regardless the number of scheduled timers. 

It follows that less than constant-time operations are limited to 2 places:

1. The native libuv timers implementation
2. The lookup for certain list of timers based on a `msec`

However, [according to the source code](https://github.com/asafdav2/node/blob/master/lib/timers.js#L84), these operations combined have shown to be trivial 
in comparison to other alternative timers architectures.
  
