---
title: "PyMiniRacer v0.12.2"
date: "2024-05-24"
lead: "Call your Python from your JavaScript from your Python"
disable_comments: false # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: true # Optional, enable MathJax for specific post
categories:
  - "programming"
tags:
  - "javascript"
  - "typescript"
  - "python"
---

The dynamic language embedding
[turducken](https://en.wikipedia.org/wiki/Turducken) is complete: you can now
call Python from JavaScript from Python!

By writing logic in Python and exposing it to V8's JavaScript sandbox, you can
now in theory, while writing _only Python and JavaScript_, create your own JS
extension library similar to
[NodeJS's standard library](https://nodejs.org/docs/latest/api/).Obviously,
anything created this way this will almost certainly be less extensive, less
standardized, and less efficient than the NodeJS standard library; but it will
be _more tailored to your use case_.

This post follows up on prior content related to
[reviving PyMiniRacer](https://github.com/bpcreech/PyMiniRacer) and then
[giving Python code the power to directly poke JS Objects, call JS Functions, and await JS Promises](https://bpcreech.com/post/mini-racer-v0.11.1/).

<!--more-->

## Adding Python extensions to V8 without writing any C++ code

PyMiniRacer runs a pretty vanilla V8 sandbox, and thus it doesn't have any
[Web APIs](https://developer.mozilla.org/en-US/docs/Web/API),
[Web Workers APIs](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API),
or any of [the NodeJS SDK](https://nodejs.org/docs/latest/api/). This provides,
by default, both simplicity and
[security](https://bpcreech.com/PyMiniRacer/architecture/#security-goals).

Thus, until now, you couldn't, say, go and grab a random URL off the Internet,
or even log to stdout (e.g., using `console.log`). Neither is bundled with V8.

Well, now you can extend PyMiniRacer to add that functionality yourself!

```python
import aiohttp
import asyncio
from py_mini_racer import MiniRacer

mr = MiniRacer()

async def demo():
  async def log(content):
    print(content)

  async def antigravity():
    import antigravity

  async with aiohttp.ClientSession() as session:
    async def get_url(url):
      async with session.get(url) as resp:
         return await resp.text()

    async with (
         mr.wrap_py_function(log) as log_js,
         mr.wrap_py_function(get_url) as get_url_js,
         mr.wrap_py_function(antigravity) as antigravity_js,
      ):
      # Add a basic log-to-stdout capability to JavaScript:
      mr.eval('this')['log'] = log_js

      # Add a basic url fetch capability to JavaScript:
      mr.eval('this')['get_url'] = get_url_js

      # Add antigravity:
      mr.eval('this')['antigravity'] = antigravity_js

      await mr.eval("""
async () => {
   const content = await get_url("https://xkcd.com/353/");
   await log(content);
   await antigravity();
}
""")()

# prints the contents of https://xkcd.com/353/ to stdout... and then also loads it in
# your browser:
asyncio.run(demo())
```

### Security note

It is possible (_modulo open-source disclaimers of warranty, etc_) to use
PyMiniRacer to run untrusted JS code; this is a
[stated security goal](https://bpcreech.com/PyMiniRacer/architecture/#security-goals).

However, exposing your Python functions to JavaScript of course breaches the
hermetic V8 sandbox. If you extend PyMiniRacer by exposing Python functions, you
are obviously taking security matters into your own hands. The above demo, by
exposing an arbitrary `get_url` function, would expose an obvious data
exfiltration and denial-of-service vector if we were running untrusted JS code
with it.

### About `async`

You will note that the above demo is using `async` heavily, both on the Python
and JavaScript side. PyMiniRacer only lets you expose `async` Python functions
to V8. These are represented as `async` on the JavaScript side, meaning you need
to `await` them, meaning in turn that the easiest way to even call them is from
`async` JavaScript code. Everything gets
[red](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
very fast!

It turns out this is the only way to reliably expose Python functionality to V8.
V8 runs JavaScript in a single-threaded fashion, and doing anything synchronous
in a callback to Python would block the entire V8 isolate. Worse, things would
likely deadlock very quickly—if your Python extension function tries to call
back into V8 it will freeze. The only thing we can reasonably do, when
JavaScript calls outside the V8 sandbox, is create a `Promise` and do the work
to fulfill that `Promise` out of band.

## Internal changes to PyMiniRacer

### Implementing `wrap_py_function`

I put some general design ideas about `wrap_py_function`
[here](https://github.com/bpcreech/PyMiniRacer/issues/39). What landed in
PyMiniRacer is basically
["Alternate implementation idea 3"](https://github.com/bpcreech/PyMiniRacer/issues/39#issuecomment-2043984826).
Generally speaking:

#### Generalizing PyMiniRacer callbacks and sharing allocated objects with V8

PyMiniRacer already had a C++-to-Python callback mechanism; this was used to
await `Promise` objects. We just had to generalize it!

... Which specifically means allowing V8 to _reuse_ a callback. With `Promises`,
callbacks are only used 0.5 times on average (either the `resolve` or `reject`
is used exactly once). PyMiniRacer handled cleanup of these on its own; the
first time either `resolve` or `reject` was called, PyMiniRacer could destroy
both callbacks. This technique was borrowed from
[V8's d8 tool](https://github.com/v8/v8/blob/0f719663da59ee690f3b72c520f8f9ea2328071a/src/d8/d8.cc#L1493).

But now we want to expose arbitary-use callbacks, which need to live longer...
We just need to attach a tiny bit of C++ state (the Python address of the
callback function, and a pointer to our V8 value wrapper) to a callback object,
and hand that off to V8.

Unfortunately,
[V8 is pretty apathetic about telling us when it's _done with_ an external C++ object we give it](https://stackoverflow.com/questions/24107280/v8-weakcallback-never-gets-called).
I couldn't make it work at all, and all commentary I can find on the matter says
trying to get V8 `MakeWeak` and finalizers work reliably is a fool's errand.
This creates a memory management conundrum: we need to create a small bit of
per-callback C++ state and hand it to V8, but V8 won't reliably tell us when it
has finally dropped all references to that state.

So I moved to a model of creating _one such callback object per MiniRacer
context_. We can tear _that_ object down when tearing down the whole MiniRacer
context, rather than relying on V8 to tell us when it's done with the data we
gave it. In order to multiplex many callbacks into that object, we can just give
JavaScript an ID number (because V8 _does_ manage to reliably destruct
numbers!). This new logic lives in
[`MiniRacer::JSCallbackMaker`](https://github.com/bpcreech/PyMiniRacer/blob/release/v0.12.2/src/v8_py_frontend/js_callback_maker.h).

Meanwhile the Python side of things can manage its own map of ID number to
callback, and remove entries from that map when it wants to. (This is what the
`wrap_py_function` context manager does on `__exit__`). Since Python is tearing
down the callback and its ID on its own schedule, in total ignorance of
JavaScript's references to that callback ID, it's possible for JS to keep trying
to call the callback after Python already tore it down. This is working as
designed; such calls can be easily spotted because the `callback_caller_id` or
`callback_id` don't reference an active `CallbackCaller` or callback (see
diagram below), and they can thus be easily ignored.

I think this could be turned into a generalized strategy for safely sharing
allocated C++ memory with V8:

1. Assume any _raw_ pointers and references to C++ objects which you directly
   share with V8 will need to live at least as long as the `V8::Isolate`.
2. If you want to be able to delete any objects _before_ the `v8::Isolate`
   exits, don't share _raw_ pointers and references. Instead:
   1. On the C++ side, create an ID-number-to-pointer map, and give V8 only ID
      numbers.
   2. Create an API, whether in C++ or JavaScript (or Python which calls C++ in
      PyMiniRacer's case) which authoratively tears down the object and the
      ID-number-to-pointer map entry. (Don't rely on V8 to tell you when all
      references to the object have dropped; it won't do this reliably.)
   3. Be prepared for JavaScript code to try and use the ID after you've torn
      down the object. The C++ code can detect this (because the ID is not in
      the map), and safely reject such attempts.

#### System diagram

Here's roughly what the system looks like:

<table border="0"><tr><td width="50%">

<div style="min-width:200px">

```goat
+----------------------------------------+
|                                        |
|           MiniRacer user code          |
|                                        |
|               +----------------------+ |
|               |                      | |
|               |   my_callback_func   | |
|               |                      | |
|               +----------+-----------+ |
|                        3 |     ^       |
+----------------+---------|-----|-------+
              1  |         |     |
                 v         |     |
+--------------------------|-----|-------+
|                          |     |       |
|               MiniRacer  |     |       |
|                          |     |       |
| +------------------------|-----|-----+ |
| |                        |     |     | |
| |             _Context   |     |     | |
| |                        v     | 10  | |
| | +----------------------------+---+ | |
| | |                                | | |
| | |      _CallbackRegistry         | | |
| | |                                | | |
| | +----------------------+---------+ | |
| |                      4 |     ^     | |
| +---------------+--------|-----|-----+ |
|               2 |        |     |       |
+-----------------|--------|-----|-------+
                  |        |     |
 Python space     |        |     |
··················|········|·····|································
 C++ space        |        |     |
                  v        |     |
+--------------------------|-----|-----+
| MiniRacer::Context       |     |     |
|                          |     |     |
|                          v     |     |
| +------------------------------|---+ |
| |                              |   | |   +---------------------+
| | MiniRacer::JSCallbackMaker   |   | |   |                     |
| |                            9 |   | |   |      (C++)          |
| | +----------------------------+-+ | |   | MiniRacer           |
| | |                              | | |   |   ::JSCallbackMaker |
| | |  MiniRacer::CallbackCaller   |<------+   ::OnCalledStatic  |
| | |                              | | | 8 |                     |
| | +------------------------------+ | |   +---------------------+
| |                                  | |              ^
| +------------------------+---------+ +--------------|----------+
|                        5 |                          |          |
| +------------------------|--------------------------|--------+ |
| |                        |                          |        | |
| |  v8::Isolate           |                        7 |        | |
| |                        |                          |        | |
| |                        |  +-----------------------+------+ | |
| |                        |  |                              | | |
| |                        '->| v8::Function                 | | |
| |                           |                              | | |
| |                           | data:                        | | |
| |                           | +--------------------------+ | | |
| | +-----------------+       | |                          | | | |
| | |                 |  6    | |  v8::Array               | | | |
| | | JavaScript code +------>| | [callback_id,            | | | |
| | |                 |       | |  callback_caller_id]     | | | |
| | +-----------------+       | |                          | | | |
| |                           | +--------------------------+ | | |
| |                           |                              | | |
| |                           +------------------------------+ | |
| |                                                            | |
| +------------------------------------------------------------+ |
|                                                                |
+----------------------------------------------------------------+
```

</div>

</td><td width="50%">

1. MiniRacer Python user code instantiates a `py_mini_racer.MiniRacer` object
   which contains a `py_mini_racer._Context` object.

2. The `py_mini_racer._Context` Python object instantiates a C++
   `MiniRacer::Context`, passing in a pointer to a generic callback in the
   Python-side `_CallbackRegistry`. The `MiniRacer::Context` creates a
   `MiniRacer::CallbackCaller` which has a process-scope `callback_caller_id`
   associated with it.

3. MiniRacer Python user code passes Python function `my_callback_func` into
   `MiniRacer.wrap_py_function`. `MiniRacer` stores a wrapper of
   `my_callback_func` in its `_CallbackRegistry`, thus generating a
   `callback_id`.

4. `MiniRacer.wrap_py_function` passes this `callback_id` down to the C++ side
   of the house to generate a V8 callback function.

5. `MiniRacer::JSCallbackMaker` creates a `v8::Function` within the
   `v8::Isolate`, with data attached containing an array of
   `[callback_id, callback_caller_id]`. This data is all
   `MiniRacer::JSCallbackMaker` needs to later find the right Python-side
   callback when this function is called. `MiniRacer::JSCallbackMaker` returns a
   handle to this `v8::Function` all the way back up to Python, which can then
   take that handle (represented as a `JSFunction` in Python) and pass it around
   to JavaScript code.

6. Eventually, JavaScript code calls the callback created above.

7. V8 dispatches that callback to `MiniRacer::JSCallbackMaker::OnCalledStatic`.

8. `MiniRacer::JSCallbackMaker::OnCalledStatic` digs out the
   `[callback_id, callback_caller_id]` array to find the
   `MiniRacer::CallbackCaller`, and the `callback_id` to pass back to it.

9. `MiniRacer::CallbackCaller` converts the returned V8 value to a
   `MiniRacer::BinaryValue`, and calls back to the Python C function pointer
   with that and the `callback_id`.

10. The `MiniRacer._ContextRegistry` converts the `callback_id` to the
    destination Python function object (`my_callback_func`), and finally passes
    the function parameters back to it.

</td></tr></table>

### Making teardown more deterministic

<div style="text-align: center;" width="250px">

![All the things meme](/img/all-the-things.jpg)

</div>

In my last post, I
[talked about](https://bpcreech.com/post/mini-racer-v0.11.1/#stdshared_ptr-all-the-things)
"regressing to C++ developer phase 1" by using `std::shared_ptr` _everywhere_ to
manage object lifecycles. As of PyMiniRacer `v0.11.1` we were using a DAG of
`std::shared_ptr` references to manage lifecycle of a dozen different classes.

I discovered [a bug](https://github.com/bpcreech/PyMiniRacer/issues/62) that
this logic left behind. The problem with the laissez-faire
`std::shared_ptr`-all-the-things memory management pattern is that we leak
memory when we have reference cycles, and unfortunately we do have reference
cycles in PyMiniRacer. In particular, we create and throw function closures onto
the `v8::Isolate` message loop which contain `shared_ptr` references to
objects... which eventually contain `shared_ptr` references to the `v8::Isolate`
itself: a reference cycle! (We also had some bare references _not_ managed by a
`shared_ptr`, which would result in a use-after-free in hard-to-reach situations
during teardown of PyMiniRacer.)

A simple way to resolve that in _another_ system might be to explicitly clear
out the `v8::Isolate`'s message queue on shutdown. Basically, by wiping out all
the not-yet-executed closures, we can "cut" all the reference cycles on exit.
However, a sticky point with PyMiniRacer's design is that it uses that same
message queue even for teardown: we put tasks on the message queue which delete
C++ objects, because it's the easiest way to ensure they're deleted under the
`v8::Isolate` lock (discussion of that
[here](https://groups.google.com/g/v8-users/c/glG3-3pufCo)). So PyMiniRacer
can't simply clear out the message, or it will leak C++ objects on exit.

After playing with this some and failing to get the lifecycles just right, and
struggling to even understand the effective teardown order of dozens of
different reference-counted objects, I realized we could switch to a different,
easier-to-think-about pattern: every object putting tasks onto the `v8::Isolate`
simply needs to ensure those tasks complete before it _anything it puts into
those tasks_ is torn down.

I'll restate that rule: _if you call `MiniRacer::IsolateManager::Run(xyz)`, you
had better ensure that task is done **before** you destruct any objects you
bound into the function closure `xyz`._

This rule seems obvious in restrospect! But it's hard to implement. I:

1. Modified
   [`MiniRacer::IsolateManager::Run`](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/isolate_manager.h#L87)
   to always return a future, to make it easier to wait on these tasks. This is
   used
   [in](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/isolate_memory_monitor.cc#L49)
   [various](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/context_holder.cc#L28)
   [places](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/context.cc#L51)
   to ensure the above rule is honored.

2. Refactored `CancelableTaskManager` completely to make it _explicitly track_
   all the tasks it makes, just so it can
   [reliably cancel _and await_ them all upon teardown](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/cancelable_task_runner.cc#L26)
   (where before it would just sort of "fire and forget" tasks, not at all
   caring if they ever actually finished).

3. Added a simple new garbage-collection algorithm in
   [`IsolateObjectCollector`](https://github.com/bpcreech/PyMiniRacer/blob/99591c962b219251d309e0f5899b3cf2a676ab9e/src/v8_py_frontend/isolate_object_collector.h#L13)
   to ensure all the garbage is gone before we move on to tearing down the
   `IsolateManager` and its message pump loop.
