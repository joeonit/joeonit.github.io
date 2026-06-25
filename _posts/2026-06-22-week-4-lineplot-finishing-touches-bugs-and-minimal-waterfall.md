---
layout: post
title: "Week 4: lineplot finishing touches, bugs and minimal waterfall"
date: 2026-06-22 10:00:00 +0000
categories: gsoc
permalink: /gsoc/week4
---

Last week ended with a working multi-sink line plot, but with one inconvenient problem: the macOS main-thread constraint still forced every flowgraph to include a GRC Snippet block calling `cyberether.present()`. That block worked, it's inconvenient for users to have to remember the block on every. I also added input-type variants (complex / float)
and exposed Superluminal's renderer-device choice to GRC. and Finished with a minimal waterfall sink implementation that is yet to be pushed.

## Custom GRC workflow

The Snippet was a workaround, not an integration. I wanted the GR-side
solution. I Asked on Matrix and Marcus from the core dev has pointed at the fix:
looked at `grc/blocks/python_qt_gui.workflow.yml`.

Reading that file unlocked the model. GRC's `generate_options` values
(`no_gui`, `qt_gui`, `python_bokeh_gui`, …) are each backed by a
`*.workflow.yml` descriptor plus a Python generator module under
`gnuradio.grc.workflows.<name>`. The YAML is just a regular block file
that GRC discovers via its block-path scan, exactly the way an OOT
installs `.block.yml`. And `generator_module` inside the YAML points at
any Python module. So OOTs can register their own `generate_options` value with zero upstream changes to GR.

The actual implementation is just four files in the OOT:

- A workflow descriptor:
  `grc/cyberether_standalone_gui.workflow.yml`. The key lines are
  `generate_options: cyberether_gui` (the new dropdown value) and
  `generator_module: gnuradio.cyberether.workflows.cyberether_standalone_gui`
  (the Python module we ship).
- A generator class
  (`python/cyberether/workflows/cyberether_standalone_gui/top_block.py`)
  that inherits from `gnuradio.grc.workflows.common.PythonGeneratorBase`
  and points at our `.mako` template.
- The Mako template itself, modelled on the in-tree
  `flow_graph_nogui.py.mako` with one change: the bottom of
  `main()` calls `cyberether.present(tb, device=...)` instead of
  `tb.wait()`.
- CMake glue so the YAML lands in `share/gnuradio/grc/blocks/` and the
  Python module ships under the OOT's package.

The result, from `examples/lineplot_demo.py`:

```python
def main(top_block_cls=lineplot_demo, options=None):
    tb = top_block_cls()
    def sig_handler(...): tb.stop(); tb.wait(); sys.exit(0)
    signal.signal(signal.SIGINT, sig_handler)
    signal.signal(signal.SIGTERM, sig_handler)
    cyberether.present(tb, device=cyberether.DeviceType.Auto)
```

No Snippet, no `tb.wait()`, no qtgui imports. `cyberether.present()` is
the main-thread loop, exactly the way `qapp.exec_()` is for Qt
flowgraphs.

## Line sink input types

Letting the user pick the input type from GRC. Same pattern `gr-blocks`
uses for `add_const_xxx`, `multiply_const_xxx`, etc. — C++ class
template + per-type explicit instantiations + GRC `option_attributes`
mapping a dropdown to the factory name.

```cpp
template <typename T>
class cyber_lineplot_sink : virtual public gr::sync_block { ... };

typedef cyber_lineplot_sink<gr_complex> cyber_lineplot_sink_c;
typedef cyber_lineplot_sink<float>      cyber_lineplot_sink_f;
```

The only type-aware code is the per-sample write in `work()`:

```cpp
if constexpr (std::is_same_v<T, gr_complex>) {
    display[d_display_write_ptr] = in[i];
} else {
    display[d_display_write_ptr] = CF32{static_cast<float>(in[i]), 0.0f};
}
```

Display tensor stays CF32; non-complex inputs get `imag=0`. Explicit
instantiations for `gr_complex` and `float` go at the bottom of the
`.cc`. Back-compat is a Python alias —
`cyber_lineplot_sink = cyber_lineplot_sink_c` — so pre-W4 flowgraphs
keep working untouched. `int`/`short`/`byte` skipped; a
`blocks.int_to_float` in front covers the rare case.

## Renderer device on the window

Same week, different surface. Superluminal's `InstanceConfig` has a
`device` field that selects which GPU API to render with (Metal, Vulkan,
CUDA, WebGPU) or leaves it on auto. I exposed it all the way through:
`cyber_context::present()` takes a `Jetstream::DeviceType` parameter,
pybind11 binds the enum as `cyberether.DeviceType`, the workflow YAML
adds a "Renderer Device" dropdown, and the Mako template emits
`cyberether.present(tb, device=cyberether.DeviceType.${device})`.

```cpp
Superluminal::InstanceConfig config;
config.device = device;
if (Superluminal::Initialize(config) != Result::SUCCESS) { ... }

// ... existing snapshot + for-loop calling Superluminal::Plot() ...
```

## Minimal waterfall sink

What's minimal:

- Complex input only.
- Same lifecycle pattern as the line sink, register with
  `cyber_context` in `start()`, write tensor in `work()`, unregister
  in `stop()`. The context handles the main-thread lifecycle.
- `source=Time, display=Frequency, operation=Amplitude, Type=Waterfall`
  Superluminal does the FFT internally.
- Single magnitude scale (Superluminal's built-in `-100..0` dB).

The first version was wrong on the buffer shape, and the way it failed taught me what
Superluminal's waterfall pipeline actually expects.

```cpp
d_tensor(DeviceType::CPU, TypeToDataType<CF32>(),
         {1, static_cast<U64>(d_fft_size)})

Superluminal::PlotConfig config = {
    .buffer    = d_tensor,
    .type      = Superluminal::Type::Waterfall,
    .source    = Superluminal::Domain::Time,
    .display   = Superluminal::Domain::Frequency,
    .operation = Superluminal::Operation::Amplitude,
};
config.options["height"] = static_cast<I32>(d_height);
cyber_context::instance().register_plot({ this, d_name, config });
```

## Where this leaves the repo

After four weeks:

- The OOT ships its own GRC workflow (the Snippet workaround is gone).
- The line sink supports complex and float inputs picked from a GRC dropdown.
- The window picks its renderer device through the same GRC options
  block (default Auto, deterministic priority Metal → Vulkan).
- A minimal complex-input waterfall sink works end-to-end. The
  multi-sink design (one CyberEther window, multiple sinks tiled in a
  shared mosaic grid) now has something visually distinct from a line plot.
- All three sink types share one `cyber_context` lifecycle, one
  registration model, one main-thread entry point.

Repo: [gr-cyberether](https://github.com/joeonit/gr-cyberether) — main
branch has the workflow + types + device work; the waterfall sink
lives on the `minimal-waterfall-sink` branch and will merge after the
meeting feedback.  
Mentors: [Luigi Cruz](https://github.com/luigifcruz) (CyberEther),
[Håkon Vågsether](https://github.com/haakov) (GNU Radio). Thanks also
to [Marcus Müller](https://github.com/marcusmueller) for pointing at `grc/workflows/` on Matrix.
