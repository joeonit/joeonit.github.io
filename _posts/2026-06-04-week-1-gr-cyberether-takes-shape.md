---
layout: post
title: "Week 1: gr-cyberether takes shape"
date: 2026-06-04 10:00:00 +0000
categories: gsoc
permalink: /gsoc/week1
---

This is the first weekly update on my Google Summer of Code project with GNU
Radio: **gr-cyberether**, an out-of-tree (OOT) module that brings CyberEther's
GPU-accelerated visualization (Superluminal) into GNU Radio flowgraphs as
native sinks.

Week 1 was deliberately about scaffolding, not features: get the build chain right, prove that the headers and the linker work, set up CI, and get data out of GNU Radio into CyberEther (implementing a minimal `cyber_lineplot_sink`). Here's how it went.

## Building against CyberEther

Starting from v1.4.0, CyberEther ships distributed binaries for all major
operating systems plus a Superluminal Python wheel. That sounds like it should
make the OOT build trivial: link the lib, ship the module. The catch is that
**none of the distributed artifacts expose the C++ development surface an OOT
needs**: the `<jetstream/superluminal.hh>` header, the rest of the
`jetstream/` include tree, and the `jetstream.pc` pkg-config file. The Python
wheel only helps Python consumers, and gr-cyberether is a C++ module.

So we have two paths: have the user build CyberEther from source so the
headers + `.pc` show up in their prefix, or vendor CyberEther as a git
submodule inside the OOT. For now I'm going with the first, and pinning a minimum version through pkg-config:

```cmake
set(CYBERETHER_MIN_VERSION "1.4.0"
    CACHE STRING "Minimum required CyberEther version")

find_package(PkgConfig REQUIRED)
pkg_check_modules(CYBERETHER REQUIRED IMPORTED_TARGET GLOBAL
    "jetstream>=${CYBERETHER_MIN_VERSION}")
```

In my last meeting with my mentors, Luigi said that in the coming weeks CyberEther would have a new major release, possibly with some minor API changes. As for the build, he's still considering adding another release channel targeted at development, one that would expose the linkable libs and the `jetstream` headers. But for now, I'll depend on building from source.

## The null sink: proving the chain end-to-end

Before writing any real block I wanted to prove the build chain works. So the
first thing in the repo is a **null sink**: one complex input, `work()`
returns immediately, no plotting. The point isn't to do anything useful; it's
to answer two questions:

- Do the `<jetstream/...>` headers actually compile under C++20 inside a GR block?
- Does `libjetstream` actually link?

The constructor does two cheap things as proof:

```cpp
// compile-time proof the headers are visible
JST_INFO("[gr-cyberether] D0 build test: built against CyberEther v{}.",
         JETSTREAM_VERSION_STR);

// link-time proof (this symbol must resolve against libjetstream)
JST_LOG_SET_DEBUG_LEVEL(JST_LOG_DEBUG_DEFAULT_LEVEL);
```

If it compiles and links, the CMake + pkg-config work above is correct and we
can move on to a block that actually does something.

## Dual-branch CI: 3.10 and main

GNU Radio is mid-stream between 3.10 and main, so the OOT has to keep working
on both. I added `.github/workflows/build.yml` so every PR builds against
**both GR 3.10 and GR main** before it can merge, as a matrix with
`fail-fast: false` so one leg breaking still shows the other.

CyberEther itself is built from source at the pinned v1.4.0 once per CI run
and cached; both GR legs reuse the cache. The two GR legs differ in setup:

- **3.10** ships in apt on Ubuntu 24.04, so it's just `gnuradio-dev`.
- **main** isn't packaged, so it's built from source and cached too.

The last step is a sanity check that the version we pinned is the one
pkg-config actually picked up:

```yaml
- name: Confirm the pinned CyberEther version was found
  run: pkg-config --modversion jetstream
```

## The line plot sink: first working prototype

With the chain proven, the first real block is the **line plot sink**. The
path is: complex stream in → ring buffer → a `CF32` Jetstream tensor →
registered with Superluminal as a time-domain `Line` plot showing the real
part of each sample.

```cpp
Superluminal::Plot(d_name, layout, {
    .buffer    = d_tensor,
    .type      = Superluminal::Type::Line,
    .source    = Superluminal::Domain::Time,
    .display   = Superluminal::Domain::Time,
    .operation = Superluminal::Operation::Real,
});
```

`work()` is just a ring-buffer write into the tensor, no rendering happens
on the GR scheduler thread.

### Lazy init — the teardown crash

A GR block has two lives: construction and running. My first version did
`Superluminal::Initialize()` and `Plot()` in the constructor, so the whole
Metal instance booted the moment the block was instantiated. Problem is the
only thing that tears the Superluminal instance down is `Show()`, which only
runs from `present()`. A typical flowgraph does `start()` and `wait()` and
never calls `present()`, so the instance booted but **never** shut down.

Fix: make the whole Superluminal lifecycle **lazy**. The constructor now only
allocates the buffer. `Initialize`, `Plot`, and `Show` all move into
`present()`, gated on a first-call flag:

```cpp
void present() {
    if (!d_initialized) {           // first call only
        Superluminal::Initialize();
        Superluminal::Plot(...);
        d_initialized = true;
    }
    Superluminal::Show();            // opens window, runs loop, tears down on close
}
```

So if you never open a window, nothing boots and there's nothing to crash
on exit. If you do open one, the same call that powers it on also powers it
off.

## What's next

- **Buffer model.** The current ring-buffer write into a single persistent
  tensor accepts a benign writer/reader race for D1. Week 2 starts the proper
  buffer research: lock-free per-sink ring, what Superluminal expects, and
  what the minimum-copy path actually looks like.
- **Threading model.** D1 uses one `present()` per sink owning the main
  thread. Week 2 sketches a shared `cyber_context` render thread so multiple
  sinks can share one window/event loop, which is the realistic shape for any
  flowgraph that wants more than one plot.
- Research the distribution plan for `gr-cyberether` as the mentor suggested.

Repo: [gr-cyberether](https://github.com/joeonit/gr-cyberether)  
Mentor: [Luigi Cruz](https://github.com/luigifcruz) (CyberEther). [Håkon Vågsether](https://github.com/haakov) (GNU Radio).
