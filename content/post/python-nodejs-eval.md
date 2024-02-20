---
title: "Running JavaScript from Python using NodeJS"
date: "2024-02-19"
lead: "It's RCE, but local!"
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

I wanted to call a pile of JavaScript (actually transpiled TypeScript) from
Python, so I could run reinforcement learning on
[this](/post/typescript-asteroids).

I was dissatisfied with the existing options for calling JS from Python, so I
made a _new_ thing!

<!--more-->

```python
    # Let's run some JavaScript from Python!
    async with evaluator() as e:
        result = await e.run("return Math.pow(7, 3);")
        assert result == 343
```

```goat
+-----------------------+            +--------------------+
| Python                |            | NodeJS             |
| +------------------+  |            |   +--------------+ |
| |    nodejs-eval   |------------------>|  http-eval   | |
| +---------+--------+  |    Unix    |   +------+-------+ |
|           |           |   domain   |          |         |
|           v           |   socket   |  o   o   v   o   o |
| +------------------+  |            |     Arbitrary  o   |
| |    nodejs-bin    +-------------->|  o  JavaScript   o |
| +---------+--------+  | fork/exec  |    o   o   o   o   |
|                       |            |                    |
+-----------------------+            +--------------------+

```

TL;DR: From NodeJS, we run a sidecar HTTP server on a Unix domain socket. We
launch the NodeJS server from Python and hit it with requests.

## Part A: NodeJS as a sidecar using `http-eval`

So while it should be possible, and relatively _nice_, to run JavaScript
directly within the CPython process by simply embedding V8,
[attempts to do so](#alternatives-considered) are somewhat fraught in
practice—_I couldn't get any to work._

This is a shame! Running an embedded in-process V8 would particularly be nice
for tightly coupled Python/JS code; you could pass control and data back and
forth freely and efficiently, in theory, and enable all kinds of fun workflows.

But what if our code isn't so coupled; what if we just want to evaluate a blob
of JavaScript and get a blob of data back, without much worry about performance?
Can we run V8 (or other JS engine) out-of-process instead? For that, NodeJS is
obviously a perfectly fine self-contained V8 JS engine, available on most modern
systems. Let's use that! We just need to teach it to run code from a parent
process...

But how do we send it code? I figured, as a low-dependency option, we can use
HTTP over a Unix domain socket as our IPC.

Obviously, an IPC which effectively sends arbitrary code for execution raises
security concerns. If done improperly, we're building a
[Remote Code Execution (RCE) vulnerability](https://en.wikipedia.org/wiki/Arbitrary_code_execution).
Using a Unix domain socket alleviates some security concern (including local
privilege escalation vulnerability). With some extra configuration footgun
checks (blocking binding to a TCP/IP port, and checking for a good
[`umask`](https://en.wikipedia.org/wiki/Umask)), using a Unix domain socket
_should_ make this safe. I think. :) More on that
[here](https://www.npmjs.com/package/http-eval#security-stance).

Anyway, I coded this up in TypeScript as an `npx`-executable binary script,
using [`express`](https://www.npmjs.com/package/express),
[`tsx`](https://www.npmjs.com/package/tsx) and
[`pkgroll`](https://github.com/privatenumber/pkgroll). See `http-eval` on
[Github](https://github.com/bpcreech/http-eval) and
[npm](https://www.npmjs.com/package/http-eval).

`http-eval` could be used to provide JavaScript evaluation to other languages
(or just to the shell, as an awkward REPL). But we're going to call it from
Python...

## Part B: bundling and launching NodeJS via `nodejs-bin`

This, thankfully, [already exists](https://pypi.org/project/nodejs-bin/), thanks
to Sam Willis!

In summary: [`nodejs-bin`](https://pypi.org/project/nodejs-bin/) is a Python
PyPI package which bundles NodeJS, so Python programmers don't have to figure
out how to install it, and Python packages can just depend on it using
Python-language dependency systems, and take NodeJS for granted. `nodejs-bin`
has a helper to launch NodeJS. In our case, we want `npx`, to just run a
script...

## Bringing it together: `nodejs-eval`

Now we have the parts we need to build a simple Python module which:

- Launches a NodeJS sidecar (specifically `npx`, from `nodejs-bin`),
- Which launches a JavaScript evaluation server on a Unix domain socket
  (`http-eval`), and
- Exposes helpers to send arbitrary JS to that server, and then decode and
  return the results. This is the `nodejs-eval` Python module.

I coded this up using [`hatch`](https://github.com/pypa/hatch),
[`aiohttp`](https://docs.aiohttp.org/en/stable/),
[`mypy`](https://mypy-lang.org/), and
[`pytest`](https://docs.pytest.org/en/8.0.x/). See `nodejs-eval` on
[Github](https://github.com/bpcreech/nodejs-eval) and
[PyPI](https://pypi.org/project/nodejs-eval/).

## Alternatives considered

Python and JavaScript are two of the most popular programming languages in the
world. Surely there should be many options for connecting A to B already?

Here's what I found:

- [`v8eval`](https://github.com/sony/v8eval): Python bindings for (Chrome's) v8
  JS engine, created by Sony.
  - _Abandoned, stuck on V8 v7.1 from 2018._
- [`pyv8`](https://code.google.com/archive/p/pyv8/): Python bindings for v8,
  created by Google.
  - _Abandoned and doesn't work anymore._
- [`PythonMonkey`](https://github.com/Distributive-Network/PythonMonkey): Python
  bindings for (Mozilla's) Spidermonkey JS engine, created by Distributive
  Networks, LLC.
  - _Still in development. I hit and then worked around a couple segfaults
    before giving up._
- [`python-spidermonkey`](https://github.com/davisp/python-spidermonkey): A
  different set of Python bindings for Spidermonkey, from Paul J. Davis.
  - _Abandoned for 14 years!_
- [`Js2Py`](https://github.com/PiotrDabkowski/Js2Py): Natively executes
  JavaScript in Python using a custom Python-native JS engine.
  - _Still in development. Doesn't support ES6 including arrow functions, so I
    moved on._
- [Selenium WebDriver](https://www.selenium.dev/documentation/webdriver/): Just
  run the whole Chromium browser as a subprocess from Python.
  - _Obviously works and used by zillions of folks, but seemed heavier-weight
    than what I want._
