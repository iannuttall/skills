# png2svg

Agent-guided PNG to SVG reconstruction for logos, icons and monograms.

png2svg recovers the geometry and paint a designer would have used. It
measures the raster at subpixel precision, builds a small parametric model,
renders it and scores the result against the source. The finished SVG uses
native paths and gradients with no embedded raster.

The project uses an agent-guided reconstruction workflow. It does not run as
a one-shot converter. Shape decomposition, overlap and intentional design
constraints still need visual judgment.

| Included example | Result | Construction |
|---|---|---|
| Keep | IoU 0.9966, edge 0.146px, 57 nodes, 1,284 B | overlapping fitted primitives |
| IN monogram | IoU 0.9976, edge 0.099px, 26 nodes, 710 B | measured polygons and clean watermark removal |
| P Auto | IoU 0.9988, edge 0.067px, 33 nodes, 852 B | one contour with a real plug cutout |

## Install the skill

```bash
npx skills add iannuttall/skills --skill png2svg
```

The skill bundles its own measurement and SVG engine. Its scripts use `uv`
to create a cached environment on first run, so nothing is installed into
the project being worked on.

Requirements:

- Python 3.12 or newer
- [uv](https://docs.astral.sh/uv/)

Ask the agent to vectorise a logo and point it at a PNG or WebP file.

## How reconstruction works

The agent drives the visual decisions. The library makes each measurement,
model and SVG reproducible.

1. Inspect the source and decide whether it is suitable.
2. Decompose the artwork into shapes, overlaps and paint regions.
3. Trace a designed outline or fit the primitives that created it.
4. Recover flat colours, gradients and shape-following ramps.
5. Render the SVG and compare it with the source at 50% opacity.
6. Read residual clusters, adjust the model and repeat.
7. Validate every output profile and export the final SVG.

There is no universal algorithm for deciding whether two meeting edges form
a fillet, an overlap or a coincidence. That judgment is the point of the
agent loop. Once the decomposition is chosen, the measurement and export
paths are deterministic.

## Run the included examples

Only Keep, IN and P Auto are published. Their source images are under
`examples/assets/`; the [artwork notice](../examples/assets/README.md) explains
their separate licensing.

### Keep

```bash
uv run png2svg init examples/assets/keep.png --project work/keep
uv run python examples/build_keep_model.py
uv run png2svg check work/keep
uv run png2svg export work/keep -o work/keep/generated/keep.svg
```

This example fits three disconnected groups as repeated rounded rectangles,
clipped diagonal arms and a separate arrow. The soft shadow and low-level
texture are deliberately left out.

### IN monogram

```bash
uv run png2svg init examples/assets/in.webp --project work/n
uv run python examples/measure_n.py
uv run python examples/build_n_model.py
uv run png2svg check work/n
uv run png2svg export work/n -o work/n/generated/in.svg
```

This example measures straight edges and fitted corner cubics. A watermark
in the source is treated as an artifact instead of being baked into the SVG.

### P Auto

```bash
uv run png2svg init examples/assets/p-auto.png --project work/p
uv run python examples/measure_p.py
uv run python examples/build_p_model.py
uv run png2svg check work/p
uv run png2svg export work/p -o work/p/generated/p-auto.svg
```

The plug is negative space connected to the outside of the P. Keeping it in
the same contour makes the cutout work over any background. The optional
`examples/build_plug_icon.py` script writes standalone plug variants under
`work/p/generated/`.

Other artwork used during development stays in the ignored `work/`
directory. Generated SVGs belong there too, which keeps private sources and
build artifacts out of the repository.

## What the checks measure

- `silhouette_iou` measures overlap between the source and render masks.
- `edge_dist_mean/p95/max` measures symmetric boundary distance in pixels.
- `deltaE_mean/p95/max` measures CIEDE2000 over interior pixels.
- `mae_linear_rgb` reports mean absolute error in linear RGB.
- `texture_std` estimates local texture in the source.
- `deltaE_lowfreq_mean/p95` scores structure and shading after removing grain.

`check` renders at 4x and downsamples by default. This reduces the
quarter-pixel edge quantisation of the SVG renderer. A correct edge near a
mask threshold can otherwise flip a full pixel row and make the model look
worse than it is.

Each check writes a comparison folder containing the reference, render,
50/50 overlay, edge map, colour difference and metrics. The overlay remains
the main visual instrument.

## Export profiles

```bash
uv run png2svg export work/name -o work/name/generated/final.svg
uv run png2svg export work/name -o work/name/generated/semantic.svg --profile semantic
uv run png2svg export work/name -o work/name/generated/animated.svg --profile animation
uv run png2svg export work/name -o work/name/generated/cropped.svg --tight --padding 2
```

`compact` is the default. It removes authoring IDs and unnecessary
whitespace, shortens paths and colours, and leaves out fixed width and height
attributes.

`semantic` keeps stable logical shape IDs. `animation` wraps every model
shape in a stable group so its fills and stroke move together. `--tight`
calculates the smallest safe viewBox without changing `project.json`.

## Repository layout

```text
skills/png2svg/          distributable skill and bundled engine
  SKILL.md               agent workflow
  references/            conventions, model schema and worked reasoning
  scripts/png2svg/       deterministic measurement and SVG library
  scripts/*_template.py  starting points for per-image reconstruction
examples/
  assets/                the three publishable source images
  build_keep_model.py    fitted Keep construction
  measure_n.py           IN measurements
  build_n_model.py       IN model
  measure_p.py           P Auto measurements
  build_p_model.py       P Auto model
tests/                   ground-truth and rendering regressions
work/                    ignored private artwork, models and generated files
```

The Python package lives inside the skill so the installed skill remains
self-contained. `pyproject.toml` points at that single copy.

## Develop the engine

```bash
uv sync
uv run pytest
uv run png2svg --help
uvx --from skills-ref agentskills validate ./skills/png2svg
```

The 87 tests cover deterministic generation, schema validation, security,
subpixel measurement, reusable geometry, gradient fitting, export profiles
and renderer regressions.

## Artwork that works well

Clean logos with flat or gradient fills are the best fit. Lines, arcs,
Béziers, overlapping primitives, conic sweeps and smoothed corners can all be
reconstructed as editable geometry.

Textured renders can still produce useful SVGs. png2svg keeps their structure
and large-scale shading while dropping grain, brushed metal and small
photographic details. Low-frequency colour metrics are used for those cases.

Photographs, painterly artwork and tiny images under roughly 100px should use
a contour tracer such as vtracer or potrace instead.

## Current limitations

- SVG has no portable conic-gradient primitive. Conic fills compile to wedge
  fans, which cost more bytes than linear or radial gradients.
- Decomposition needs visual judgment, so there is no automatic
  `reconstruct` command.
- The agent loop acts as the editor. There is no separate interactive UI.

## License

The code is MIT licensed. Source images under `examples/assets/` have
separate ownership and reuse terms described in their artwork notice.
