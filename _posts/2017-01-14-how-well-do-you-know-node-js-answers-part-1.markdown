---
layout: post
title: "How Well Do You Know Node.js? - Answers (Part 1)"
date: 2017-01-14T17:48:05+02:00
comments: true
tags: [node-js]
---

Recently I came across a blog post titled [How well do you know Node.js?](https://edgecoders.com/how-well-do-you-know-node-js-36b1473c01c8#.sxqlki3ar). In it, Samer Buna lists 48
questions which be expects a Node.js developer to be able to answer. This post is part 1 of my attempt to provide answers to these questions. 
<!-- more -->
You can check out [part 2]({% post_url 2017-01-18-how-well-do-you-know-node-js-answers-part-2 %}).

### What is the relationship between Node.js and V8? Can Node work without V8?
V8 is a JavaScript engine developed by The Chromium Project, first for the Google Chrome web browser and later for other projects, including Node.js.
It allows to compile, optimize and run JavaScript code and is the base for code execution inside Node.js.
However, V8 is not essential for Node.js; There are attempts to use other javascript engines, such as [node-chakracore](https://github.com/nodejs/node-chakracore)
(Node.js on ChakraCore) or [spidernode](https://github.com/mozilla/spidernode) (Node.js on top of SpiderMonkey).

### How come when you declare a global variable in any Node.js file it’s not really global to all modules?
A module's code is wrapped by a function wrapper that looks like the following:
{% highlight javascript %}
(function (exports, require, module, __filename, __dirname) {
  // module code
});
{% endhighlight %}
This wrapping allows to keeps top-level variables (defined with var, const or let) scoped to the module, rather than to
the global object.

Read more on the [module wrapper](https://nodejs.org/api/modules.html#modules_the_module_wrapper).

### When exporting the API of a Node module, why can we sometimes use `exports` and other times we have to use `module.exports`?
To understand the difference, we can look at this simplified view of a JavaScript file in Node.js:
{% highlight javascript %}
var module = { exports: {} };
var exports = module.exports;

// your code

return module.exports;
{% endhighlight %}

so, `exports` is initially an alias to `module.exports`. if you want to simply export an object with
named fields, you can use the `exports` shortcut. for example, had we written `exports.a = 9`, we'd
actually export this object: `{a: 9}`.

However, if you want to export a function or another object, you have to use the `module.exports` object.
For example: `module.exports = function bar() {}`. Once you do that, `exports` and `module.exports`
no longer reference the same object.

### Can we require local files without using relative paths?
There are several options, as described [here](https://gist.github.com/branneman/8048520).

### Can different versions of the same package be used in the same application?
No, this is currently prevented by NPM. see [this issue](https://github.com/npm/npm/issues/2943) for more details.

### What is the Event Loop? Is it part of V8?
In event-driven programming, an application expresses interest in certain events and respond to them when they occur.
This is the way Node.js can handle asynchronous execution while running the code in a single thread. When an
asynchronous operation starts (for example, when we call `setTimeout`, `http.get` or `fs.readFile`), Node.js sends
these operations to a different thread allowing V8 to keep executing our code. Node also calls the callback when
the counter has run down or the IO / http operation has finished. In Node.js, the responsibility of gathering events
from the operating system or monitoring other sources of events is handled by [libuv](https://github.com/libuv/libuv),
and the user can register callbacks to be invoked when an event occurs. The event-loop usually keeps running forever.

### What is the Call Stack? Is it part of V8?
The call stack is the basic mechanism for javascript code execution. When we call a function, we push the function parameters and
the return address to the stack. This allows to runtime to know where to continue code execution once the function ends.  In Node.js,
the Call Stack is handled by V8.

### What is the difference between `setImmediate` and `process.nextTick`?
`setImmediate` queues a function *behind* whatever I/O event callbacks that are already in the event queue.
`process.nextTick` queues a function *at the head* of the event queue so that it executes immediately after
the currently running function completes.

### How do you make an asynchronous function return a value?
You could return a promise resolving to that value, for example `return Promise.resolve(true)`.

### Can callbacks be used with promises or is it one way or the other?
Callbacks and promises can be used together. For example, the following method calls a callback and returns a promise:
{% highlight javascript %}
function foo(cb) {
  // do some processing
  if (cb) {
    cb();
  }
  return Promise.resolve(true);
}
{% endhighlight %}

### What are the major differences between spawn, exec, and fork?
  * `exec` methods spawns a shell and then executes a command within that shell, buffering any generated output
  * `spawn` works similarly to `exec`. The main difference is that `spawn` returns the process output as a stream while `exec` returns it as a buffer
  * `fork` is a special case of `spawn` that also creates a new V8 engine instance. This is useful to create additional
workers of the same Node.js code base. (for example, in the [cluster module](https://nodejs.org/api/cluster.html)).

### How does the cluster module work? How is it different than using a load balancer?
The [cluster module](https://nodejs.org/api/cluster.html) works by forking the server into several worker processes (all run
inside the same host). The master process listens and accepts new connections and distributes them across the worker processes in a
round-robin fashion (with some built-in smarts to avoid overloading a worker process).

A load balancer, in contrast, is used to distribute incoming connections across *multiple hosts*.

### What are the `--harmony-*` flags?
These are flags that one can pass to the Node.js runtime to enable **Staged** features. 
Staged features are almost-completed features that are not considered stable by the V8 team.

### How can you read and inspect the memory usage of a Node.js process?
You can invoke the [`process.memoryUsage()`](https://nodejs.org/api/process.html#process_process_memoryusage<Paste>) method which returns an object describing the memory usage of the Node.js process, measured in bytes.

### Can reverse-search in commands history be used inside Node’s REPL?
Currently it seems like its not possible. The Node REPL does allow to persist the history into a file and later load it, but doesn't allow
to reverse-search it.

### What are V8 object and function templates?
V8 object is a native, C++ representation of a JavaScript object. It is a essentially the way the V8 engine views javascript objects.
A function template is the blueprint for a single function. You create a JavaScript instance of the template by calling the template's

`GetFunction` method from within the context in which you wish to instantiate the JavaScript function. You can also associate a C++
callback with a function template which is called when the JavaScript function instance is invoked.

### What is libuv and how does Node.js use it?
[libuv](https://github.com/libuv/libuv) is a multi-platform support library with a focus on asynchronous I/O. Its core job is to provide an event loop and
callback based notifications of I/O and other activities. In addition, libuv offers core utilities such as timers, non-blocking networking
support, asynchronous file system access, child processes and more.

### How can you make Node’s REPL always use JavaScript strict mode?
You can run Node.js with the `--use_strict` which will open the REPL in strict mode.

### What is process.argv? What type of data does it hold?
The `process.argv` property returns an array containing the command line arguments passed when the Node.js process
was launched. The first element will be `process.execPath`. The second element will be the path to the JavaScript file
being executed. The remaining elements will be any additional command line arguments.

### How can we do one final operation before a Node process exits? Can that operation be done asynchronously?
By registering a handler for `process.on('exit')`:

{% highlight javascript %}
function exitHandler(options, err) {
    console.log('clean');
}

process.on('exit', exitHandler.bind(null));
{% endhighlight %}

Listener functions to the `exit` event must only perform synchronous operations. To perform asynchronous operations,
one can register a handler for `process.on('beforeExit')`.

### What are some of the built-in dot commands you can use in Node’s REPL?
The following dot commands can be used:

* `.break` - When in the process of inputting a multi-line expression, entering the `.break` command (or pressing the `<ctrl>-C` key combination) will abort further input or processing of that expression.
* `.clear` - Resets the REPL 'context' to an empty object and clears any multi-line expression currently being input.
* `.exit` - Close the I/O stream, causing the REPL to exit.
* `.help` - Show this list of special commands.
* `.save` - Save the current REPL session to a file: `> .save ./file/to/save.js`
* `.load` - Load a file into the current REPL session. `> .load ./file/to/load.js`
* `.editor` - Enter editor mode (`<ctrl>-D` to finish, `<ctrl>-C` to cancel)

### Besides V8 and libuv, what other external dependencies does Node have?
Beside V8 and libuv, node has several other dependencies:

* [http-parser](https://github.com/joyent/http-parser/): a lightweight C library which handles HTTP parsing
* [c-areas](http://c-ares.haxx.se/docs.html): used for some asynchronous DNS requests
* [OpenSSL](https://www.openssl.org/docs/): used extensively in both the `tls` and `crypto` modules
* [zlib](http://www.zlib.net/manual.html): used for fast compression and decompression

Read more about [node dependencies](https://nodejs.org/en/docs/meta/topics/dependencies/).

---
Well, that's it for part 1. You can check out the rest at [part 2]({% post_url 2017-01-18-how-well-do-you-know-node-js-answers-part-2 %})
