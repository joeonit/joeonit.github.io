---
layout: post
title: "Week 5: Waterfall support, refactoring and fixing bugs"
date: 2026-06-29 10:00:00 +0000
categories: gsoc
permalink: /gsoc/week5
---

Week 4 ended with the minimal waterfall sink. This week it landed on `main` and
caught up to the line sink having complex and float inputs, a Display Domain
dropdown, the same `cyber_context` lifecycle. Then I removed a GRC option that turned out to do nothing, fixed a binding-order bug.

## Waterfall sink

The minimal waterfall from week 4 merged via [PR #4](https://github.com/joeonit/gr-cyberether/pull/4).
Once it was on `main`, bringing it up to the line sink's level was mostly
applying patterns that already existed.

Input types first. The same recipe the line sink got in week 4, a class
template with per-type explicit instantiations and a GRC dropdown mapping:

```cpp
typedef cyber_waterfall_sink<gr_complex> cyber_waterfall_sink_c;
typedef cyber_waterfall_sink<float>      cyber_waterfall_sink_f;
```

Float inputs get `imag = 0` on the way into the CF32 display tensor,
exactly like the line sink. The only thing waterfall-specific is the `height` option and that
Superluminal runs the FFT internally when `display = Frequency`.

## Removing the Operation dropdown

Same day I added two GRC options to both sinks: **Display Domain**
(Time / Frequency) and **Operation** (Real / Imaginary / Amplitude /
Phase). Display Domain is real, it picks whether Superluminal shows the
raw time series or inserts an FFT and shows the spectrum. It works.

Operation does not. Reading Superluminal's line and waterfall pipeline
builders, the enum never reaches the render path both collapse a
complex sample to `|z|` regardless of what you pass. So the
`operation` argument is gone from both `make()` signatures, the GRC
blocks, and the bindings.

```cpp
// before
static sptr make(size_t buffer_size, const std::string& name,
                 Domain display = Domain::Time,
                 Operation operation = Operation::Real);   // ← ignored downstream

// after
static sptr make(size_t buffer_size, const std::string& name,
                 Domain display = Domain::Time);
```

## A pybind ordering bug

Splitting the enums out surfaced a subtle one. The sink bindings use
Superluminal's enums (`Domain`, `DeviceType`) as default argument
values in their pybind signatures. pybind has to know those enum types
at the moment it builds the sink's function signature. If the enum
isn't registered yet, the import throws.

The fix is just call order in `python_bindings.cc`:
`bind_cyber_context` (which registers the enums) has to run before any
sink binding that names them.

```cpp
bind_cyber_context(m);        // registers DeviceType, Domain first
bind_cyber_lineplot_sink(m);  // so these can use them as defaults
bind_cyber_waterfall_sink(m);
```

## Removing the null sink

The last cleanup was the oldest code in the repo. `cyber_null_sink` was
the very first block I wrote just to prove the build worked correcly and it has no use now.

## Next week

- Understand the mosaic and implement a kind of translation unit to better support multi sink visualliztion.
- Improve the documentation and add building from source guide.
- Add QA tests.

Repo: [gr-cyberether](https://github.com/joeonit/gr-cyberether) — `main`  
Mentors: [Luigi Cruz](https://github.com/luigifcruz) (CyberEther),
[Håkon Vågsether](https://github.com/haakov) (GNU Radio).
