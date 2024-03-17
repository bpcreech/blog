---
title: "Reviving PyMiniRacer"
date: "2024-03-16"
lead: "JS-in-Python, in-process, redux!"
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

In [this last blog post](https://bpcreech.com/post/python-nodejs-eval/), I
created a [helper](https://pypi.org/project/nodejs-eval/) to call JavaScript
from Python using a NodeJS sidecar process. In the post I
[commented](https://bpcreech.com/post/python-nodejs-eval/#alternatives-considered)
that *in-*process JS evaluation might be nicer. The old
[Sqreen `PyMiniRacer` project](https://github.com/sqreen/PyMiniRacer) had only
_recently_ fallen into disrepair. Can we revive it?

**TL;DR: Yes!** It just took a couple weeks of elbow grease, and new ownership
(it me). I now own the _two_ best ways to run JavaScript from a Python program:
[`PyMiniRacer`](https://github.com/bpcreech/PyMiniRacer) and
[`nodejs-eval`](https://github.com/bpcreech/nodejs-eval).

<!--more-->

```python
from py_mini_racer import MiniRacer

mr = MiniRacer()

# Let's run some JavaScript from Python!
>>> mr.run("Math.pow(7, 3);")
343

# Updated, for the first time since 2021!
>>> mr.v8_version
"b'12.2.281.23'"

# Now supported: the Intl API!
>>> mr.eval('Intl.DateTimeFormat(["ban", "id"]).format(new Date())')
'16/3/2024'

# As of the v0.9.0 release, async execution (as a side effect, Control+C works):
>>> mr.eval('while (1) {}')
^CTraceback (most recent call last):
...
KeyboardInterrupt
```

_Other new features can be found
[on the relnotes page](https://bpcreech.com/PyMiniRacer/history/), where v0.7.0
is the first new version since 2021._

## A lineage

1. **[`therubyracer`](https://github.com/rubyjs/therubyracer) (2009-2018)**
   [Charles Lowell](https://github.com/cowboyd) of
   [Frontside Software](https://frontside.com/) created
   [The Ruby Racer](https://github.com/rubyjs/therubyracer) to embed
   [V8](https://v8.dev/) (the JavaScript engine used by Chrome, NodeJS, etc)
   into [Ruby](https://www.ruby-lang.org/) for direct JS execution from Ruby
   programs.
   - Unfortunately, the rich integration between Ruby Racer and V8 became a pain
     point for Ruby Racer, because upgrading V8 often mean revamping Ruby Racer
     to fit interface changes. So this project was eventually archived, and
     replaced with...
2. **[`mini_racer`](https://github.com/rubyjs/mini_racer) (2016-)**
   [Sam Saffron](https://github.com/SamSaffron) and others created
   [`mini_racer`](https://github.com/rubyjs/mini_racer), a new Ruby / V8
   integration, stripped down relative to Ruby Racer. This version is still
   maintained.
3. **[`sqreen/PyMiniRacer`](https://github.com/sqreen/PyMiniRacer) (2016-2021)**
   [Sqreen](https://github.com/sqreen), a wep app security startup, created
   [`PyMiniRacer`](https://github.com/sqreen/PyMiniRacer), a Python module
   modeled after Ruby's `mini_racer`. This followed the same model of minimizing
   the interface with V8, and also used a Python `ctypes` integration (as
   opposed to a Python extension module) which furthermore minimized the
   interface with _Python_, resulting in a JS/Python integration with relatively
   little support burden.
   - Unfortunately, `PyMiniRacer` wasn't updated after 2021 when
     [Sqreen was acquired by DataDog](https://www.datadoghq.com/blog/datadog-acquires-sqreen/).
4. **[`bpcreech/PyMiniRacer`](https://github.com/bpcreech/PyMiniRacer) (2024-)**
   This is what you're reading about now. :)

After discussion with the Sqreen (now DataDog) folks, we decided to host my
revival of their `PyMiniRacer` project as a fork, which lives here:

- [Github](https://github.com/bpcreech/PyMiniRacer)
- [Docs](https://bpcreech.com/PyMiniRacer/), and
- [PyPI](https://pypi.org/project/mini-racer/).

## General updates

Other than upgrading V8—which has its own section below—I took the opportunity
to dust off various parts of this project.

### Python ecosystem updates

In particular, lots of things have happened in the Python world!

#### Python versions (drop Python 2, add up to 3.12)

First, we can drop Python 2 which was
[globally EOL'd in 2020](https://www.python.org/doc/sunset-python-2/) (after a
deprecation plan over a decade long!). Because the world is big, folks are still
using Python 2 out there, but we don't need to maintain an _up-to-date_ V8
integration for them.

Meanwhile, as of this writing, Python is up to 3.12, which for `PyMiniRacer`
added some minor breakage here and there. For example, some change to the Python
`memoryview` system
[now requires](https://github.com/bpcreech/PyMiniRacer/blob/d1b6f4d83306bf7b94a7827d45476017b6ec5382/src/py_mini_racer/py_mini_racer.py#L508)
[explicit `memoryview` casting](https://github.com/python/cpython/issues/60148).

I also added support for the fancy new Python
[`importlib.resources`](https://docs.python.org/3/library/importlib.resources.html)
specification which lets Python directly run modules _and load their
dependencies, e.g., `PyMiniRacer`'s DLL file_ out of unusual non-filesystem
places (such as from within `zip` files). This system was added in Python 3.7
and revamped in Python 3.9; `PyMiniRacer` now supports both incantations of
loading-data-files-from-the-package.

#### Packaging with Hatch

`PyMiniRacer` originally built binary distributions using `setuptools`, and
managed its various bits of automation using a hand-written `Makefile`. Python
now has a standardized pluggable packaging system for building binary
distributions, and [Hatch](https://github.com/pypa/hatch) is the most popular
implementation of it.
[_"Hatch is trying to be the Cargo or Go CLI equivalent for Python"_](https://github.com/pypa/hatch/discussions/1117#discussioncomment-7827378)
per [its author](https://github.com/ofek). By using it (and accepting its
various opinions) we can drop a _lot_ of configuration from `PyMiniRacer`.

Hatch includes a bundled an opinionated linter and code formatter in
[Ruff](https://github.com/astral-sh/ruff), which lets us drop `flake8` and
`isort` and their config files as development dependencies. The only default
setting I changed was the line length, from 120 to Black's default of 88. I
thought this pointless debate was
[finally settled by Black](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html#line-length)
when it won the formatting war, but for some reason Hatch
[overrides this setting to 120](https://github.com/pypa/hatch/discussions/1117),
so I put it back where Black (and Ruff) default to.

Hatch also includes built-in support for Python version matrix testing. It works
super well (modulo, not on [Alpine](https://www.alpinelinux.org/) for
[reasons](https://github.com/indygreg/python-build-standalone/issues/86)) and
lets us drop `tox` as a development dependency.

### Docs!

Inspired by
[this post](https://matklad.github.io//2021/02/06/ARCHITECTURE.md.html), I
figured we should have an `ARCHITECTURE.md`, so
[I wrote one](https://github.com/bpcreech/PyMiniRacer/blob/main/ARCHITECTURE.md)
([same on the `mkdocs` site](https://bpcreech.com/PyMiniRacer/architecture/)).

I also sprinkled in a ton of comments. `PyMiniRacer`'s V8 build (see below) is
full of workarounds and config changes. These do _not_ help in forwards
compatibility, because each little config tweak is a potential source of future
breakage when that config parameter stops working. Now, we at least have a paper
trail of where those tweaks came from!

Finally, and most dramatically from a cosmetic perspective, I migrated the
`PyMiniRacer` docs from [Sphinx](https://www.sphinx-doc.org/en/master/) to
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) and created
a docs build pipeline (AFAICT there wasn't one!). Sphinx has been around and
working forever, but the current ecosystem mindshare seems to be pouring into
`mkdocs-material` lately.

I am a little worried about maintainability since `mkdocs-material` is a
complicated and load-bearing _plugin_ for the `mkdocs`, and itself only works
with
[_other_ plugins for `mkdocs-material`](https://squidfunk.github.io/mkdocs-material/setup/extensions/).
it's a setup ripe for [this situation](https://xkcd.com/2347/). But, I went with
peer pressure, and the new docs look great and have a very simple configuration,
because `mkdocs-material` is indeed fantastic. The new docs live
[here](https://bpcreech.com/PyMiniRacer).

## Actually building V8

Okay, the main work here is updating V8. The last Sqreen version of
`PyMiniRacer`, from 2021, used V8 8.9, and no longer builds. V8 is up to 12.2
today.

### General challenges in building V8

There is no official binary distribution of V8. The only way to use V8 is to
build it yourself. _Unless, perhaps, you use the NodeJS binary, which brings us
back to
[my last blog post](https://bpcreech.com/post/python-nodejs-eval/)—maybe, after
all, the best way to use V8 is via a server running in NodeJS?_

But building V8 is hard! Fun challenges in building V8:

1. **V8 is enormous and building it is slow:** V8 contains _or dynamically
   downloads_ 6.6 GB of _source_! (Okay, not all "source", actually: this
   includes some vendored copies of operating system roots, like `/usr` from
   flavors of Debian.) There are about 2.4k build steps, which include building
   tons of code generated by [the Torque compiler](https://v8.dev/docs/torque).
   It takes over an hour to build from scratch on a
   [free GitHub Actions runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)
   (currently, a 4-CPU machine for Linux, etc). To build for Linux `aarch64`,
   GitHub doesn't provide any free hosted runners, so we run via
   [emulation](https://github.com/uraimo/run-on-arch-action), and it takes
   _several days_. This far exceeds
   [the maximum 6 hours provided by GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration#usage-limits),
   meaning builds fail due to the time limit. We can work around _that_
   limitation by using [`sccache`](https://github.com/mozilla/sccache) to cache
   and catch up on builds; after enough retries our GitHub Actions builds _do_
   eventually succeed. (And hopefully, one day, GitHub will provide free
   `aarch64` Linux runners!)

2. **V8 wants to set up its own build ecosystem:** To
   [build V8](https://v8.dev/docs/build), you first
   [download another set of utilities](https://v8.dev/docs/source-code) called
   [`depot_tools`](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up).
   `depot_tools` includes its very own binaries built for some but not all our
   target platforms, for things like Python,
   [Goma](https://chromium.googlesource.com/infra/goma/client/) (a build cache
   we're not using), [Ninja](https://ninja-build.org/) (a build system we do
   use), [GN](https://github.com/o-lim/generate-ninja) (a meta-build system we
   also use), etc.

3. **That build ecosystem, and the build in general, doesn't actually work _on_
   Alpine or Linux-on-Arm:** For `PyMiniRacer` we want to target at least
   `{ Windows, Mac, Linux [glibc], Linux [musl] } × { x86_64, aarch64 }`
   (`aarch64`
   [by popular demand](https://github.com/sqreen/PyMiniRacer/issues/154)).
   However, V8 doesn't support building on Linux-on-arm64, although it does
   support cross-compiling for it. V8 doesn't support `musl` (Alpine's `libc`)
   in either on-host building _or_ cross-compiling. So we need to do various fun
   config tweaks to make it actually work.

4. **V8 and the build system change all the time:** V8 is under very heavy
   development at Google, for a variety of products (Chromium of course, but
   also ChromeOS, etc). The available and default config options change over
   time, meaning any intricate build setup we do in `PyMiniRacer` is likely to
   break in with newer V8 verions. So we want to _minimize_ the amount of build
   configuration we do in `PyMiniRacer`, to future-proof it as best we can.

5. **V8 needs a bleeding-edge LLVM (particularly, clang) and wants its own
   libstdc++:** V8 uses brand new features of `clang`, including an ML-driven
   optimization model. It comes with a build of the llvm toolchain, but only for
   supported platforms (thus excluding Alpine, and excluding building _on_ Linux
   `aarch64`). We must mitigate this by installing a bleeding-edge LLVM from
   [the LLVM project](https://llvm.org/) where the binaries vendored into V8
   itself don't work.

Meanwhile, we impose another challenge by sticking to the free GitHub Actions
runners: unfortunately, GitHub Actions has no native `aarch64` hosted runners,
and no native Alpine runners. We work around this using Umberto Raimondi's
fantastic [`run-on-arch-action`](https://github.com/uraimo/run-on-arch-action).
This GitHub Action plug-in helps us build Docker containers for Linux
distributions and architectures, and then build `PyMiniRacer` there.

### Extra features added while updating V8

Aside from
[all the V8 updates from v8.9 to v12.2](https://chromium.googlesource.com/v8/v8/+log/branch-heads/8.9..branch-heads/12.2/?n=1000),
I plumbed in the following which had been disabled in prior `PyMiniRacer`
builds:

- Support for the [ECMAScript internalization API](https://v8.dev/docs/i18n) and
  thus [the ECMA `Intl` API](https://tc39.es/ecma402/)
- V8 [fast startup snapshots](https://v8.dev/blog/custom-startup-snapshots)

### Potential future work in simplifying the V8 build

The Ruby [`mini_racer` project](https://github.com/rubyjs/mini_racer) mentioned
above actually split the V8 build out into a separate project,
[`libv8-node`](https://github.com/rubyjs/libv8-node): "_A project for
distributing the v8 runtime libraries and headers in both source and binary
form, packaged as a language-independent zip and as a Ruby gem._". This project
takes a different tack on the problem by reusing NodeJS's opinionated vendored
copy and build of V8 instead of trying to build V8 from
[Google's directions](https://v8.dev/docs/build). We _might_ be able to simplify
`PyMiniRacer` by rebasing it upon the `libv8-node` build of V8. We'd need to
ensure `libv8-node` is up-to-date and stable (it's not totally clear to me that
it is) and, because we're dropping the V8 build entirely, move the compilation
of
[`PyMiniRacer`'s custom C++ code](https://github.com/bpcreech/PyMiniRacer/tree/main/src/v8_py_frontend)
out of `GN`+`ninja` and into another to-be-determined build system.

Alternatively, it would be nice if V8 lived within a common C/C++ package
system. The winning multi-platform C/C++ package system today seems to be
<https://conan.io>. Making V8 work with Conan (well enough for official upload
to conan central) would be tough because V8 loves to download its own
dependencies in violation of Conan's common-sense One-Definition Rule (ODR).

## Future work in `PyMiniRacer`

Future work may include:

- Support for
  [Python `asyncio`](https://docs.python.org/3/library/asyncio.html).
- Other stuff from
  [the old GitHub issues](https://github.com/sqreen/PyMiniRacer/issues) list.
- Updates to new V8 builds which we can assume will appear unabated.
