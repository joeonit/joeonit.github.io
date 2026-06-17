---
layout: post
title: "Week 2 & 3: buffers, threading, and more"
date: 2026-06-16 10:00:00 +0000
categories: gsoc
permalink: /gsoc/week2-3
---

Week 2 was reduced work due to my finals, no new deliverable just research and tying off the minimal line-plot sink from last week. Week 3 was applying that research. So this post covers both, and the more interesting story is the part of the plan that didn't survive contact with the code.

To recap on last week's line-plot sink problems:

1. **Single-plot ceiling** : each sink owned the whole Superluminal lifecycle,
   so a second sink couldn't register without conflicting with the first.
2. **Data race** : GNU Radio's `work()` and Superluminal's compute
   thread were reading and writing the same `CF32` display tensor with no
   synchronization. Visually fine at one sink, but a real race.

The plan was a per-sink lock-free ring buffer for the first problem and a shared `cyber_context` wrapper for the second.

## Week 2: research week

Most of Week 2 was reading, not coding. I had two problems to solve and zero
real background in either of them, so before I touched the codebase I
needed to get comfortable with the concepts: lock-free data structures, the
C++ memory model (`std::atomic`, `memory_order_relaxed/acquire/release/seq_cst`),
the producer/consumer pattern, ThreadSanitizer, and the general shape of
"how do two threads share state without locking each other out." A lot of
this was new vocabulary for me; by the end of the week I had enough to
read the SPSC ring patterns in the wild and understand why the
acquire/release pair gives you the happens-before edge that makes the
whole thing safe.

Alongside that, I read CyberEther's own renderer (`cyberether/src/superluminal/base.cc`)
to understand what Superluminal actually expects from a buffer producer.
Two things were load-bearing for everything that came later:

- `Superluminal::Start()` already spawns a *compute* thread and a *present*
  thread. The main thread becomes the GLFW/AppKit event loop the moment
  you call `Block()`. So there is no hidden render thread we need to
  spawn, Superluminal owns rendering end-to-end, and we're a data source
  feeding a renderer that already runs itself.
- External tensors are imported via *Dynamic Memory Import* (DMI), whose
  CPU back-end is a no-op: Superluminal reads the tensor pointer you pass
  to `Plot()` live during every `compute()` pass. There is no hidden copy
  inside CyberEther.

That second point looked harmless during the research week. It quietly
became the reason for Week 3's u-turn.

The last Week 2 deliverable was the distribution plan for the OOT, the question Luigi, my mentor, raised in the Week 1 meeting. I thought we can wait on this decision until we have a couple of working sinks and a robust threading/buffer system. Also, as Luigi mentioned, he will decide whether it's possible to have another release channel targeted toward development that would expose the linkable libraries.

For now building from source is the only path, none of CyberEther's
distributed v1.4.0 artifacts expose the C++ dev surface an OOT needs
(`<jetstream/superluminal.hh>`, the rest of `jetstream/`, `jetstream.pc`).
Once we have a stable surface I want the user-facing channel to be
**Homebrew on macOS and conda-forge cross-platform**, not hand-rolled
installers, conda-forge in particular plays well with the
rest of the GNU Radio ecosystem (radioconda).

## Week 3: building the planned thing

Two pieces shipped first.

**A lock-free single-producer / single-consumer ring buffer** —
[`lib/spsc_ring.h`](https://github.com/joeonit/gr-cyberether/blob/main/lib/spsc_ring.h),
introduced in commit
[`14e5b9c`](https://github.com/joeonit/gr-cyberether/commit/14e5b9c)
and wired into the sink in
[`2aa3d61`](https://github.com/joeonit/gr-cyberether/commit/2aa3d61).
One producer (GR's `work()`), one consumer (a per-sink ticker thread),
two atomic counters, no mutex. Here's the producer side, verbatim:

```cpp
// Producer side, called only from the GR scheduler thread.
// Writes up to `n` samples, returns the number actually written.
size_t push(const T* src, size_t n) {
    const size_t head = d_head.load(std::memory_order_relaxed);
    const size_t tail = d_tail.load(std::memory_order_acquire);
    const size_t free_space = d_capacity - (head - tail);
    const size_t to_write = std::min(n, free_space);

    for (size_t i = 0; i < to_write; ++i) {
        d_buffer[(head + i) % d_capacity] = src[i];
    }

    d_head.store(head + to_write, std::memory_order_release);
    return to_write;
}
```

The `tail` is loaded with `acquire` so the producer observes how far the
consumer has drained, samples are then written into the buffer. Finally
`head` is published with a `release` store, which is the commit. The
consumer does the mirror image with `tail` (`pop()`). The acquire/release
pair is what gives the happens-before edge: the samples written *before*
the release are guaranteed visible *after* the acquire, without a lock,
without `work()` ever blocking. Overflow policy is drop-on-full —
`push()` writes `min(n, free)` and returns the count, which is the right
call for a visualisation sink: keep the newest samples, never
back-pressure the flowgraph.

Tested with 8 unit tests including drop-on-full under load and CF32
torn-write detection, all ThreadSanitizer-clean.

**A `cyber_context` singleton + registry** — header at
[`include/gnuradio/cyberether/cyber_context.h`](https://github.com/joeonit/gr-cyberether/blob/main/include/gnuradio/cyberether/cyber_context.h),
implementation at
[`lib/cyber_context.cc`](https://github.com/joeonit/gr-cyberether/blob/main/lib/cyber_context.cc),
the initial lifecycle move landed in commit
[`865bd66`](https://github.com/joeonit/gr-cyberether/commit/865bd66).

This is the piece I want to spend the most time on, because it solves
more than one problem and the why matters.

**Why have a context at all.** Week 1 sinks each owned the whole
Superluminal lifecycle (`Initialize → Plot → Show → Terminate`). That
shape can't be made to scale past one sink:

1. Two sinks each calling `Initialize()` race against the global
   Superluminal instance.
2. Two sinks each calling `Show()` fight for the main thread.
3. Two sinks each calling `Plot()` with their own mosaic grid get
   rejected, Superluminal enforces that every plot share one grid,
   and a sink at construction time has no way of knowing how many
   siblings will exist.
4. A sink that calls `Initialize()` in its constructor and never calls
   `Show()` (the common case for a normal flowgraph) leaves Superluminal
   booted but unterminated, which crashes at process exit. (This was the
   Week 1 "mutex lock failed" bug.)

The context fixes all four by being the single place that talks to
Superluminal. Sinks never call `Initialize`, `Plot`, `Show`, or
`Terminate` directly, they just **register** a plot with the context
at `start()` and **unregister** it at `stop()`. The context does the
talking, on the main thread, once, when the user opens the window with
`cyberether.present()`.

Registration is cheap and safe to call from a GR worker thread because
it doesn't touch Superluminal at all:

```cpp
void
cyber_context::register_plot(const plot_request& request)
{
    std::lock_guard<std::mutex> lock(d_mutex);
    auto it = std::find_if(d_plots.begin(), d_plots.end(),
                           [&](const plot_request& p) { return p.owner == request.owner; });
    if (it != d_plots.end()) {
        *it = request;            // replace if a sink re-registers
    } else {
        d_plots.push_back(request);
    }
}
```

The `owner` field is a stable per-sink key (each sink passes `this`) so
the same sink can re-register cleanly and two sinks that happen to share
a display name are still tracked separately.

`present()` is where all the Superluminal talking happens. It's also
where the multi-plot grid gets decided, because by the time `present()`
runs, every sink has already registered and the count of plots is known.
The grid layout is then a one-liner:

```cpp
// square grid: cols = ceil(sqrt(n)), rows = ceil(n / cols).
//   1 -> 1x1   2 -> 1x2   3,4 -> 2x2   5,6 -> 2x3   ...
const size_t n    = plots.size();
const U8          cols = std::ceil(std::sqrt(double(n)));
const U8          rows = (n + cols - 1) / cols;

for (std::size_t i = 0; i < n; ++i) {
    const U8 row = i / cols;
    const U8 col = i % cols;
    const auto mosaic = Superluminal::MosaicLayout(rows, cols, 1, 1, col, row);
    Superluminal::Plot(plots[i].name, mosaic, plots[i].config);
}
```

Every plot gets the *same* grid (which Superluminal demands) but a
different `(col, row)` cell, so they tile instead of stacking. This is
the actual multi-plot fix, and it has nothing to do with the ring
buffer.

The rest of `present()` is the lifecycle pump: `Start()` → spawn one
update thread that calls `Update()` at display cadence → `Block()` on
the main thread until the user closes the window → join the update
thread → `Stop()` + `Terminate()`. Re-entry is guarded with an atomic
exchange so a stray second call to `present()` returns immediately
instead of opening a second window. With this much in place, the
multi-sink showcase opens with four tiles in a 2×2 grid, no overlaps,
no lifecycle conflicts. The Week 2 plan worked.

## The twist

Then I went back and read `base.cc` one more time, more carefully,
specifically looking at how the buffer wiring I'd built compared to how
`Show()` does it. Three things landed in close succession.

**`Update()` is global, not per-plot.** Every sink's ticker was calling
`Superluminal::Update(d_name)`, but the implementation ignores the name
argument and wakes the *one* compute thread for the whole window. Four
sinks meant four threads competing to do the same wake-up.

**The ring didn't fix the race I thought it fixed.** This was the
load-bearing finding from Week 2 catching up with me. Superluminal reads
the display tensor in place via DMI, there is no copy. So the ring
decoupled `work()` from the ticker (which is real), but the ticker still
wrote the tensor while Superluminal's compute thread read it. The race
had moved one layer down, not gone away.

**`work()` runs at sample rate.** An earlier draft of the no-ring
version had each sink call `Update()` straight from `work()`. At a 48 kHz
source that means 48 000 `Update()` calls per second into a renderer
that wants 60. Wrong shape.

The right shape, sitting in plain sight in `Show()`, was: write the
tensor directly from `work()`, run *one* thread that calls `Update()`
every 16 ms for the whole window, and put it in `present()` next to
`Block()` so its lifetime matches the window's. The refactor that did
all of this is commit
[`fe9e757`](https://github.com/joeonit/gr-cyberether/commit/fe9e757).

The sink shrank dramatically — `work()` now writes straight into the
rolling tensor, no ring, no ticker (full file:
[`lib/cyber_lineplot_sink_impl.cc`](https://github.com/joeonit/gr-cyberether/blob/cyber-context-threading/lib/cyber_lineplot_sink_impl.cc)):

```cpp
CF32* display = d_tensor.data<CF32>();
for (size_t i = 0; i < n; ++i) {
    display[d_display_write_ptr] = in[i];
    d_display_write_ptr = (d_display_write_ptr + 1) % d_buffer_size;
}
// no Update() here — present() drives that on its own thread
```

And the update loop moved into `cyber_context::present()`, one thread for
the whole window, matching `Show()` byte-for-byte:

```cpp
update_thread = std::thread([&update_running]() {
    while (update_running.load(std::memory_order_acquire) &&
           Superluminal::Presenting()) {
        Superluminal::Update();
        std::this_thread::sleep_for(std::chrono::milliseconds(16));
    }
});

Superluminal::Block();   // main-thread event loop until window closes
```

### What's parked vs what's gone

- The per-sink ticker thread is gone, deleted from the sink, not coming back.
- The SPSC ring is still in the tree, marked unused.

So the Week 3 score on the ring is honest: I built a thing that didn't
fit the contract I was producing for, learned the contract properly in
the process, and kept the piece that earns its keep for the iteration
that does need it. The atomics + memory-ordering work was foundational
for everything else in the file.

## Where this leaves the repo

After three weeks the working tree is:

- One CyberEther line-plot sink, multi-instance, sharing a window.
- A `cyber_context` that owns the Superluminal lifecycle, lays plots out
  in a shared mosaic grid, and runs the single Update thread.
- A 4-tile multi-sink showcase flowgraph (`examples/lineplot_showcase.grc`)
  that exercises the grid layout, the duplicate-name disambiguation, and
  the lifecycle on close.
- A parked SPSC ring buffer.
- Dual-branch CI (GR 3.10 + GR main) still green.

> TO-BE-FIXED next week: Currently, the design still relies on the GRC Python snippet as an entry point. This is because macOS strictly demands that all UI windows and event loops run on the main thread, and GRC doesn't allow us to launch a window from the main thread unless we inject it via a code snippet.

Repo: [gr-cyberether](https://github.com/joeonit/gr-cyberether)  
Mentors: [Luigi Cruz](https://github.com/luigifcruz) (CyberEther),
[Håkon Vågsether](https://github.com/haakov) (GNU Radio).
