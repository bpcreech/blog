---
title: "PyMiniRacer v0.11.1"
date: "2024-04-09"
lead: "Poke objects! Call functions! Await promises!"
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

In [this last blog post](https://bpcreech.com/post/mini-racer/), I discussed my
revival of [PyMiniRacer](https://github.com/bpcreech/PyMiniRacer), a neat
project created by [Sqreen](https://github.com/sqreen) to embed the V8
JavaScript into Python. As of that post (`v0.8.0`), I hadn't touched the C++
code yet. Here we talk about some extensions to PyMiniRacer, rolling up the
changes up to `v0.11.1`: JS `Object` and `Array` manipulation, directly calling
JS functions from Python, `async` support, and a discussion of the C++ changes
needed to make all that work.

<!--more-->

_**Note: As of this writing on 2024-04-09, `v0.11.0` is
[up on PyPI](https://pypi.org/project/mini-racer/) and I'm building the
`v0.11.1` release, which has a bug fix for long-running microtask, and a lot of
the C++ revamp explained below. PyMiniRacer builds take a few days because we
build on aarch64 using emulation, so `v0.11.1` should be ready on PyPI
approximately 2024-04-11. Check out the full docs for PyMiniRacer
[here](https://bpcreech.com/PyMiniRacer/).**_

```python
from py_mini_racer import MiniRacer

mr = MiniRacer()

# Direct object and array access!
>>> obj = mr.eval('let obj = {"foo": "bar"}; obj')
>>> obj["foo"]
'bar'
>>> obj["baz"] = mr.eval('[]')
>>> obj["baz"].append(42)
>>> mr.eval('JSON.stringify(obj)')
'{"foo":"bar","baz":[42]}'

# Call JS functions directly!
>>> func = mr.eval('(a) => a*7')
>>> func(6)
42

# Promise await support, so you can wait in two languages at once!
>>> async def will_you_wait_just_one_second_please():
...    promise = mr.eval('new Promise((res, rej) => setTimeout(res, 1000))')
...    await promise
...
>>> import asyncio
>>> asyncio.run(will_you_wait_just_one_second_please())  # does exactly as requested!
>>>
```

_Other new features can be found
[on the relnotes page](https://bpcreech.com/PyMiniRacer/history/), where v0.8.0
was the subject of the [last blog post](https://bpcreech.com/post/mini-racer/)._

## New feature rundown

First, I'll discuss new features for PyMiniRacer. These require an incremental
overhaul of the C++ side of PyMiniRacer, which is discussed below.

### Manipulating objects and arrays

As of `v0.8.0` and earlier, PyMiniRacer could create and manipulate objects and
arrays only at a distance: you could create them in the JS context via
`MiniRacer.eval` statements, and poke at them via _more_ `MiniRacer.eval`
statements to either pull out individual values or `JSON.stringify` them in
bulk. `MiniRacer.eval` could convert primitives like numbers and strings
directly to Python objects, but to get a member of an object, you had to run an
evaluation of some JS code which would extract that member, like
`mr.eval(my_obj["my_property"])` instead of simply writing
`my_obj["my_property"]` in Python.

It feels like programming with
[waldos](https://en.wikipedia.org/wiki/Remote_manipulator):

<div style="text-align: center;">
  
![A "waldo" or "remote manipulator"](/img/waldos.jpg) </p>

_Working with Objects and Arrays in PyMiniRacer v0.8.0.
[source](https://en.wikipedia.org/wiki/Remote_manipulator)_

</div>

Well, now you can directly mess with objects and arrays! I added a
[`MutableMapping`](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableMapping)
(`dict`-like) interface for all derivatives of JS Objects, and a
[`MutableSequence`](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableSequence)
(`list`-like) interface for JS Arrays. You can now use Pythonic idioms to read
and write `Object` properties and `Array` elements in Python, including
recursively (i.e., you can read `Object`s embedded in other `Object`s, and embed
your own).

This required tracking v8 object handles within Python (instead of reading and
disposing them at the end of every `MiniRacer.eval` call), which in turn
required revamping the C++ memory management model. More on that
[below](#relieving-python-of-the-duty-of-managing-c-memory).

### Direct function calls

As of `v0.8.0` and earlier, PyMiniRacer couldn't directly call a function. You
could only evaluate some JS code which would call your function. Passing in
function parameters was likewise awkward; you had to serialize them into JS
code. So instead of doing `foo(my_str, my_int)` in Python, you had to do
something like `mr.eval(f'foo("{my_str}", {my_int})')`. The `MiniRacer.call`
method helped with this by JSON-serializing your data for you, but the wrapper
it puts around your code isn't always quite right (as reported on the original
[sqreen/PyMiniRacer](https://github.com/sqreen/PyMiniRacer) GitHub issue
tracker).

Well, now you can retrieve a function from JS, and then... just call it:

```python
>>> reverseString = mr.eval("""
function reverseString(str) {
    return str.split("").reverse().join("");
}
reverseString  // return the function
""")
>>> reverseString
<py_mini_racer.py_mini_racer.JSFunction object at 0x7bb3dc5739a0>
>>> reverseString("reviled diaper")
'repaid deliver'
```

You can also specify `this` as a keyword argument, because JavaScript:

```python
>>> get_whatever = mr.eval("""
function get_whatever(str) {
    return this.whatever;
}
get_whatever  // return the function
""")
>>> obj = mr.eval("let obj = {whatever: 42}; obj")
>>> get_whatever(this=obj)
42
```

As with direct `Object` and `Array` manipulation, aside from a straightforward
exposure of C++ APIs to Python, to make this work we have to revamp the C++
object lifecyle model; more
[below](#relieving-python-of-the-duty-of-managing-c-memory).

### Async and timeouts

It seemed like a big gap that PyMiniRacer, as of `v0.8.0` and earlier, could
create JS Promises, but couldn't do anything to asynchronously await them.
[I wasn't the only one with this feeling](https://github.com/bpcreech/PyMiniRacer/issues/7).
One of the PyMiniRacer tests did work with promises, but only by
[polling for completion](https://github.com/bpcreech/PyMiniRacer/blob/5161d99a39076c36209519c9601d32546c8d276b/tests/test_eval.py#L215).

Both Python and JavaScript have a concept of `async` code. Can we hook them up?
Yes!

Now you can create a Promise on the JS side and await it either using `asyncio`
or using a blocking `.get` call:

```python
>>> promise = mr.eval('new Promise((res, rej) => setTimeout(() => res(42), 1000))')
>>> await promise  # only works within Python "async def" functions
42
>>> promise.get()  # works outside of python "async def" functions and not recommended
42                 # within async functions (because it would block up the asyncio loop)
```

To make this demo work, I actually also had to write a `setTimeout` function,
which is funny because it's so ubiquitous you might forget that it's not part of
the ECMA standard, and thus not part of V8's standard library. (`setTimeout` is
[a _web_ standard](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout),
and also
[exists in NodeJS](https://nodejs.org/api/timers.html#settimeoutcallback-delay-args).
E.g., in browser scripts, `setTimeout` lives on the `window` object, but
PyMiniRacer has no `window` object.) Turns out we can write a pure-JS
`setTimeout` using only the ECMA standard libraries using
[a somewhat hacky wrapper](https://github.com/bpcreech/PyMiniRacer/blob/560d5ac7de6b0b92a12d7a4a8062b9392a28a1b4/src/py_mini_racer/py_mini_racer.py#L684)
of
[`Atomics.waitAsync`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/waitAsync).
I stole this insight from the PyMiniRacer unit tests and spun it into a
glorified `settTimeout` / `clearTimeout` wrapper in what felt like one of those
silly improbable interview questions ("Please build thing A using only
half-broken tools B and C!").

Moreover, this required making PyMiniRacer better at running code indefinitely
so it would actually process the async work reliably—more on that below.

## Changes to the PyMiniRacer C++ backend

So, the above features involve overhauling PyMiniRacer. Let's talk about how I
did that!

### `clang-tidy`

First, before getting into C++ changes, I wanted to inherit the best of
automated wisdom about how to write C++. I personally have been writing C++ for
over two decades, but having taken a break from it for 5 years, I missed out on
the fun of both C++17 and C++20!

So I added
[a `clang-tidy` pre-commit](https://github.com/bpcreech/PyMiniRacer/blob/560d5ac7de6b0b92a12d7a4a8062b9392a28a1b4/.pre-commit-config.yaml#L36).
As well as a `clang-format` pre-commit, because duh. Some observations:

1. `clang-tidy` has taught me lots of things I didn't know, e.g., when to
   `std::move` stuff and when to rely on
   [guaranteed copy elision](https://en.cppreference.com/w/cpp/language/copy_elision),
   and how to annoy others by using
   [trailing return types](https://en.wikipedia.org/wiki/Trailing_return_type)
   _everywhere, absolutely everywhere_.
2. It continually catches my mistakes, like extra copies, extra lambda
   parameters, and whatnot!
3. There is unfortunately no good set of recommended `clang-tidy` checks
   everyone should enable (as compared to ESLint for JavaScript, Ruff for
   Python, etc, which work pretty well out of the box). Some bundled checks are
   broadly useful for most everyone, and some are are clearly only intended for
   a limited audience. E.g., the llvm checks are doomed to fail if you're not
   _writing llvm itself_. The `altera` checks are intended for people writing
   code for FGPAs. The `fuschia` checks have some very unusual opinions that I'm
   sure make sense for the Fuschia project but I cannot imagine there is
   consenus that, e.g., defaults in function parameters are bad. So everyone
   using `clang-tidy` has to figure out, by trial and error, which checks have
   weird non-applicable opinions and thus have to be disabled.
4. The memory management checks seem unhelpful in that I, like most C++
   programmers, use smart pointers _everywhere_, so when the checks fail it's
   just noise 100% of the time so far. It seems like these complicated
   memory-tracking checks could almost be simplified into "uh you used new or
   delete without a shared pointer", and then would only _usefully_ trigger for
   novice C++ programmers.
5. `clang-tidy` is slow; something like 100 times slower than `clang` itself. It
   takes about 20 minutes on my old laptop to run over PyMiniRacer, which is
   only 29 small files.

Anyway, `clang-tidy` is super useful; would recommend!

### Inverting PyMiniRacer's threading model

So, to start off, for `async` code to work correctly in PyMiniRacer (and also,
to run code off the Python thread, thus enabling `KeyboardInterrupt` of
PyMiniRacer JS code), we need V8 to execute code _continually_, e.g., to process
delayed callbacks from `setTimeout`. In other words, if want to be able to use
`setTimeout` to schedule work for `N` seconds from now, and have it actually,
you know, _run_ that delayed work, we need to convince V8 to actually run,
continually, until explicitly shut down.

However, PyMiniRacer was set up like _both_ of V8's
[extensive list of two samples](https://v8.github.io/api/head/examples.html). It
ran a thing once, and pumped the V8 message loop a bit, and quit, never to call
V8 again (until the next user input). This seems odd: how do you know there is
no delayed work? I guess you just assume there's no delayed work. But at the
same time, programs like Chrome, which
[a few people use](https://backlinko.com/chrome-users), and NodeJS,
[likewise](https://radixweb.com/blog/nodejs-usage-statistics), obviously use V8
in continually-operating form. How do we do it?

A couple facts make "how do we do it" tricky to answer:

1. The `v8::Isolate`, the environment in which all your JS code runs, is not
   inherently thread-safe. You need to grab a `v8::Locker` first to use it
   safely.
1. `v8::platform::PumpMessageLoop`, the thing that powers all work beyond an
   initial code evaluation in V8, needs the `v8::Isolate` lock. However, it does
   not actually _ask for_ the lock. It does not release the lock either,
   apparently. And yet we need it to run, continually and without returning
   control to us. We have to use its wait-for-work mode, (unless we want to
   [use a lot of electricity](https://en.wikipedia.org/wiki/Busy_waiting)),
   which means the message pump is doomed to sit around a lot, doing nothing but
   hogging the `v8::Isolate` lock.

So you need to get the lock to use the `Isolate`, but you also need to spend a
lot of time calling this thing (`PumpMessageLoop`) hogs that lock. How do you
reconcile these?

My solution was inspired by [the `d8` tool](https://v8.dev/docs/d8), which ships
with V8: all code which interacts with the `Isolate` is
["posted" as a "task"](https://github.com/v8/v8/blob/2ce051bb21edff5e66c4e87180c9d90b18fdf526/include/v8-platform.h#L82)
on the `Isolate`'s `TaskRunner`. Then it will run _under_ the `PumpMessageLoop`
call, where _it already has that `Isolate` lock which `PumpMessageLoop` has been
hogging_. Nobody needs to grab the `Isolate` lock, because _they already have
it_, having been sequenced into the `PumpMessageLoop` thread as tasks.

This seems to work, but involved reorganizing all of PyMiniRacer, inverting
control such that a thread running `PumpMessageLoop` is the center of the
universe, and everything else just asks it to do work. Even things like "hey I'd
like to delete this object handle" need to be put onto the `v8::Isolate`'s task
queue.

The resulting setup looks roughly like this:

```goat
+--------------------------------------------------------------------+
|                      MiniRacer::IsolateManager                     |
|                                                                    |
|                    .--------------------------------------------.  |
|                   | MiniRacer::IsolateMessagePump thread         | |
|                   |                                              | |
|                   |                    +-------------+           | |
|                   |  1. creates, ----->|             |           | |
|                   |                    | v8::Isolate |           | |
|                   |  2. exposes,       |             |           | |
| v8::Isolate* <----+--------------------+             |           | |
|     ^             |                    +--+----------+           | |
|     | 6. enqueues |                       |   ^                  | |
|     |             |                       |   |                  | |
|     |             |  3. … then runs        '-'                   | |
|     |             |     v8::Platform::PumpMessages               | |
|     |             |     (looping until shutdown)                 | |
|     |              '---------------------------------------------' |
|     |                                                              |
+-----+--------------------------------------------------------------+
      | 5. MiniRacer::IsolateManager
      |     ::Run(task)
      |
+-----+--------------------+            +-----------------------+
|                          |            |                       |
| MiniRacer::CodeEvaluator +----------->| MiniRacer::AdHocTask  |
|         (etc)            | 4. creates |                       |
+--------------------------+            +-----------------------+
```

Reading that diagram in order:

1. The `MiniRacer::IsolateMessagePump` runs a thread which creates a
   `v8::Isolate`,
2. ... exposes it to the `MiniRacer::IsolateManager`,
3. ... and loops on `v8::platform::PumpMessageLoop` until shutdown.
4. Then, any code which wants to _use_ the `Isolate`, such as
   `MiniRacer::CodeEvaluator` (the class which implements the `MiniRacer.eval`
   function to run arbitrary JS code) can package up tasks into
   `MiniRacer::AdHocTask` and
5. ... throw them onto the `Isolate` work queue to actually run, on the
   message-pumping thread.

#### `std::shared_ptr` all the things!

Our control inversion to an event-driven design (explained above) is complicated
to pull off safely in C++, where object lifecycle is up to the developer. Since
everything is event-driven, we have to be very careful to control the lifecycle
of every object, ensuring objects outlive all the tasks which reference them.
After trying to explicitly control everything, I gave up and
[settled on](https://github.com/bpcreech/PyMiniRacer/pull/38) basically using
`std::shared_ptr` to manage lifecycle for just about everything (regressing to
"C++ developer phase 1" as described
[here](https://www.reddit.com/r/cpp_questions/comments/17tl7oh/best_practice_smart_pointer_use/)).
If a task has a `std::shared_ptr` pointing to all the objects it needs to run,
the objects are guaranteed to still be around when the task runs. This in turn
involves some refactoring of classes, to ensure there are no references cycles
when we implement this strategy. Reference cycles and `std::shared_ptr` do not
get along.

#### Threading, in summary

The above story seems like it should be common to most use of V8, yet all seems
underdocumented in V8. The solution I landed on involved some educated guesses
about thread safety of V8 components (like, can you safely add a task to the
`Isolate`'s foreground task runner without the `Isolate` lock? The
implementation seems to say so, but the docs... don't exist! Does
`v8::platform::PumpMessageLoop` need the lock? It seems to crash when I don't
have it; core files are a form of library documentation I guess, but maybe I was
simply holding it wrong when it crashed?) I have
[put a question to the v8-users group](https://groups.google.com/g/v8-users/c/glG3-3pufCo/m/-rSSMTLEAAAJ)
to see if I can get any confirmation of my assumptions here.

### Relieving Python of the duty of managing C++ memory

There are 5 places where Python and C++ need to talk about objects in memory:

1. The overall MiniRacer context object which creates and owns everything
   (including the `v8::Isolate` discussed above).

2. JS values created by MiniRacer and passed back to Python (and, when we start
   doing mutable objects or function calls, _JS values created in Python and
   passed into JS_!).

3. Callback function addresses from C++ to Python.

4. Callback context to go along with the above, so that when Python receives a
   callback it understands what the callback is about. (E.g., we typically use a
   Python-side `Future` as the callback context here; the callback sends both
   _data_ and _context_. The callback function just has to take the _data_,
   i.e., a result value, and stuff it into a `Future`, which it conveniently can
   find given the callback's _context_.)

5. Task handles for long-running `eval` tasks, so we can cancel them.

While [Python's garbage collector](https://docs.python.org/3/library/gc.html)
tracks references and automatically deletes unreachable _Python_ objects, if
you're extending Python with C (or C++) code, you have to explicitly delete the
C/C++ objects. There are kinda two approaches for systematically ensuring
deletion happens:

1. **`with` statements:**

   You can treat C++ objects as external resources and use
   [`with statements and context managers](https://peps.python.org/pep-0343/) to
   explicitly manage object lifecycle, in user code. There is
   [an absolutist view](https://stackoverflow.com/questions/1481488/what-is-the-del-method-and-how-do-i-call-it#answer-1481512)
   (previously held by me after being burned by finalizers before; see below)
   that this is the only way proper way to manage external resources (_but are
   in-process C++ objects really "external resources"?_). Code using context
   managers to explicitly deallocate all allocated objects would look like, in
   absolutist form:

   ```python
   with MiniRacer() as mr:
     with mr.eval("[]") as array:
        array.append(42)
        with mr.eval('JSON.stringify') as stringify_function:
           with stringify_function(array) as stringified:
              print(stringified)  # prints '[42]'
              # at this point, the value behind "stringified" is freed by calling a C API.
           # at this point, the value behind "stringify_function" is freed by calling a C API.
        # at this point, the value behind "array" is freed by calling a C API.
     # at this point, the MiniRacer context is freed by calling a C API.
   ```

   _(Astute Pythonistas may note that two of those `with` statements could be
   collapsed into one to make it "simpler". Yay.)_

   This is nicely explicit, but extremely annoying to use. (I actually
   implemented this, but undid it when I started updating the PyMiniRacer
   `README` and saw how annoying it is.)

2. **Finalizers, aka
   [the `__del__` method](https://docs.python.org/3/reference/datamodel.html#object.__del__):**

   You can create a Python-native wrapper class whose `__del__` method frees the
   underlying C++ resource. This looks like, e.g.:

   ```python
   class _Context:
     def __init__(self, dll):
        self.dll = dll  # ("dll" here comes from the ctypes API)
        self._c_object = self.dll.mr_init_context()

     def __del__(self):
        self.dll.mr_free_context(self._c_object)

     ...
   ```

   Then user code doesn't have to remember to free things at all! _What could go
   possibly wrong?_

   <center><div style="max-width: 300px;">

   ```goat
   +--------------+    +--------------+
   |              +--->|              |
   |  A (Python)  |    |  B (Python)  |
   |              |<---+              |
   +--------------+    +--------------+
   ```

   </div></center>

#### The trouble with finalizers

_What could possibly go wrong_ is that finalizers are called somewhat lazily by
Python, and in kind of unpredictable order. This problem is shared by Java,
[C#](https://stackoverflow.com/questions/30368849/destructor-execution-order)
and I assume every other garbage-collected language which supports finalizers:
if you have two objects `A` and `B` which refer to each other (aka a reference
cycle), but which are obviously otherwise unreachable, obviously you should
garbage-collect them.

But if _both_ `A` and `B` have finalizers, which do you call first? If you call
`A`'s finalizer, great, but later `B`'s finalizer might try and reach out to
`A`, _which has already been finalized!_ Likewise, if you finalize `B` first,
then `A`'s finalizer might try and reach out to `B` which is already gone. You
can't win! So if you're the Python garbage collector, you just call the
finalizers for both objects, in _whatever_ order you like, and you declare it to
be programmer error if these objects refer to each other in their finalizers,
and you generally _declare that finalization order doesn't matter_.

<center><div style="text-align: center; max-width: 400px;">

```goat
      +--------------+    +--------------+
      |              +--->|              |
      |       A      |    |       B      |
      |              |<---+              |
      +-------+------+    +-------+------+
Python space  | `__del__`         |  `__del__`
··············|···················|·············
C++ space     |                   |
              v                   v
      +--------------+    +--------------+
      |              |    |              |
      |       C      +--->|       D      |
      |              |    |              |
      +--------------+    +--------------+
```

</div></center>

Unfortunately, _order sometimes matters_. Let's say those objects `A` and `B`
are each managing C++ objects, `C` and `D`, respectively, as depicted above.
Obviously, in the above picture, we should tear down `C` before `D` so we don't
leave a dangling reference from `C` to `D`. The best teardown order here is:
`A`, `C`, `B`, _then_ `D`. But _Python doesn't know that_! It has no idea about
the link between C++ objects `C` and `D`. It is likely to call `B`'s finalizer
first, tearing down `D` before `C`, thus leaving a dangling reference on the C++
side.

<center><div style="text-align: center; max-width: 600px;">

```goat
   +------------------------------+    +--------------------------+
   |                              +--->|                          |
   |        _ValueHandle          |    |         _Context         |
   |                              |<---+                          |
   +----------+-------------------+    +---+----------------------+
              |                            |
Python space  | `__del__`                  | `__del__`
··············|····························|····························
C++ space     | calls mr_value_free        | calls mr_context_free
              | with a pointer             | with a pointer
              v                            v
   +------------------------------+    +--------------------------+
   |                              |    |                          |
   |    MiniRacer::BinaryValue    |    |    MiniRacer::Context    |
   |                              |    |                          |
   +----------------+-------------+    +------------+-------------+
                    | points into                   | owns
                    |                               v
                    |                  +--------------------------+
                    |                  |                          |
                    +----------------->|        v8::Isolate       |
                                       |                          |
                                       +--------------------------+
```

</div></center>

This happens in practice in MiniRacer: a Python `_Context` wraps a
`MiniRacer::Context`, and a Python `_ValueHandle` wraps a
`MiniRacer::BinaryValue`. But within C++ land, that `MiniRacer::Context` is what
created those `MiniRacer::BinaryValue` objects, and they each know have pointers
into the `v8::Isolate` object which the `Context` owns. If you free the
`MiniRacer::Context` before you free the _values pointing into its
`v8::Isolate`_, things get very crashy, fast. We _want_ Python to free all the
`_ValueHandle` objects _before_ the `_Context`, but there's no way to tell it
that, and worse, the garbage collection ordering is nondeterministic, so it will
get it wrong, and crash, _randomly_.

The same situation arises for task handles (`MiniRacer::CancelableTaskHandle` in
C++), and we have other more mundane risky use of pointers and Python-owned
memory with callback function pointers and callback context data.

When I started work on it, MiniRacer tracked raw C++ pointers for the MiniRacer
context _only_, using `__del__` to clean it up. It only used values ephemerally,
converting and cleaning them up immediately (except for `memoryview` objects
into
[JS `ArrayBuffers`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)
but that was a rare case). If you're only using finalizers with one type of
object, and instances of that object don't interact, the reference cycle problem
explained above doesn't exist.

This worked fine until... I introduced more object types (async tasks, callback
functions, callback contexts, and persistent values to track `Object`s and
`Array`s). More object types means more opportunities for Python to call
finalizers _in the wrong order_.

#### Making finalizers safe

So the principle I derive from the above: _**we must refactor things so that
finalizer order doesn't matter**_. But how do we do it? And otherwise make our
Python/C++ boundary safe from memory management bugs?

I developed the following strategy:

1. Avoid passing pointers at all. Instead, pass integer identifiers
   (specifically `uint64_t`) between C++ and Python. The identifiers are all
   created and registered in maps on the C++ side. The map lets the C++ side
   safely validate, _then_ convert from the identifier to actual object
   pointers.

   - Two exceptions:

     1. For JS values, for performance reasons, we pass a special
        `BinaryValueHandle` which doubles as an identifier _and_ an address
        which can peek into data for primitive types. (In that case, the map key
        is the `BinaryValueHandle` pointer instead of an ID. The mapped value is
        the full `BinaryValue` object pointer, which is _not_ directly
        accessible to Python.) Note that the C++ side still validates any
        `BinaryValueHandle` values passed in from Python.

     2. For callback addresses from C++ to Python, we _have_ to use a pointer.
        Which then means we are at risk of
        [bugs](https://docs.python.org/3/library/ctypes.html#callback-functions)
        wherein PyMiniRacer accidentally disposes the callback before the C++
        side is totally done with it. C'est la vie.)

2. Thus the C++ side can _check for validity of any references it receives from
   Python_, eliminating any bugs related to use-after-free or Python freeing
   things in the wrong order (i.e., freeing a whole `v8::Isolate` and only
   _then_ freeing a JS value which lived in that isolate).

3. ... And the C++ side can also avoid _memory leaks_, in that when an "owning
   container" (like the context object) is torn down, the container (on the C++
   side) has a holistic view of _all_ objects created within that container, and
   can free any stragglers itself, on the spot.

In this way, we _still use_ `__del__` on the Python side, but _only_ as an
opportunistic memory-usage-reducer. The C++ side is actually tracking all memory
usage deterministically and doesn't rely on Python to get it right—especially
not the order of operations when freeing things.

As a cute bonus trick, since we're tracking all living objects on the C++ side,
can expose methods which count them,
[for use in our unit tests](https://github.com/bpcreech/PyMiniRacer/blob/560d5ac7de6b0b92a12d7a4a8062b9392a28a1b4/tests/conftest.py#L7),
to ensure we don't have any designed memory leaks! (I
[caught](https://github.com/bpcreech/PyMiniRacer/blob/main/HISTORY.md#0111-2024-04-08)
one pre-existing leak this way!)

The resulting setup, just looking at Contexts and JS Values, sort of looks like
the following diagram (and similar goes for async task handles):

<center><div style="text-align: center; max-width: 600px;">

```goat
   +---------------------------------+    +------------------------------+
   |                                 +--->|                              |
   |           _ValueHandle          |    |           _Context           |
   |                                 |<---+                              |
   +------------------------------+--+    +---+--------------------------+
                                  |           |
Python space   `__del__`          |           | `__del__`
··································|···········|······························
C++ space    calls mr_value_free  |           | calls mr_context_free with
             with a context ID +  |           | a uint64 id
             value handle pointer |           |
                                  |           v
                                  |       +-----------------------------+
                                  +------>|                             |
                                          |  MiniRacer::ContextFactory  |
                                          |    validates context IDs…   |
                                          +-----------+-----------------+
                                                      | … and passes valid
                                                      v calls to …
                                          +------------------------------+
                                          |                              |
              +---------------------------+      MiniRacer::Context      |
              | "passes individual value  |                              |
              | deletions to, and/or      +----------+-------------------+
              | deletes in bulk upon                 |
              v context teardown"                    |
   +---------------------------------+               |
   |                                 |               |
   |  MiniRacer::BinaryValueFactory  |               |
   |     validates handle ptrs       |               |
   +----+----------------------------+               |
        | delete (if handle is valid, or upon        |
        v factory teardown if never deleted)         |
   +---------------------------------+               |
   |                                 |               |
   |      MiniRacer::BinaryValue     |               |
   |                                 |               |
   +----------------+----------------+               | owns
                    | points into                    v
                    |                  +--------------------------+
                    |                  |                          |
                    +----------------->|        v8::Isolate       |
                                       |                          |
                                       +--------------------------+
```

</div></center>

With this new setup, if Python finalizes ("calls `__del__` on") the `_Context`
_before_ a `_ValueHandle` that belonged to that context, aka "the wrong order",
what happens now is:

1. `_Context.__del__` calls `MiniRacer::ContextFactory` to destroy its
   `MiniRacer::Context`. `MiniRacer::Context` destroys the
   `MiniRacer::BinaryValueFactory`, which in turn notices some C++
   `MiniRacer::BinaryValue` objects were left alive (before this change, the
   `BinaryValueFactory` wasn't tracking them and thus didn't even know they
   still existed). `BinaryValueFactory` goes ahead and tears all those leftover
   `BinaryValue` objects down. This avoids a memory leak, and avoids dangling
   pointers on the C++ side.

2. When Python later calls `_ValueHandle.__del__`, it passes a context ID to
   `MiniRacer::ContextFactory`. This context ID is no longer valid because the
   context was torn down already. Thus, this call can be safely ignored.

I think the design strategy is generalizable to most Python/C++ integrations,
and potentially likewise Java/C++ and C#/C++ integrations. (_TL;DR: use integer
identifiers instead of passing pointers, track object lifecycle in maps on the
C++ side, validate all incoming identifiers from Python, and refactor so that
out-of-order finalization doesn't hurt anything_). I wonder if it's written down
anywhere.

## Contribute!

That wraps up this overly detailed changelog for now!

If you're reading this and want to contribute, go for it! See
[the contribution guide](https://bpcreech.com/PyMiniRacer/contributing/).

One thing I wish PyMiniRacer really had, and don't have time to build right now,
is extension via user-supplied Python callback functions. A more detailed
descrition and implementation ideas can be found
[here](https://github.com/bpcreech/PyMiniRacer/issues/39).
