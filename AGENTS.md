# Working on Ian's agent skills

Every public skill lives at `skills/<name>/`. Keep each installed folder
self-contained. Supporting tests, examples and private work belong outside
the skill folder.

## png2svg

This repo's product is the **skill** in `skills/png2svg/`. Everything else
exists to build, test and document it.

## The design principle

There is no single algorithm that reconstructs every logo. Decomposition —
how many shapes, what covers what, which coincidences are real design
constraints — needs eyes on the image. So:

- **the library makes every measurement deterministic and exact**
- **the agent decides what to measure, what overlaps what, and when to stop**

Do not try to collapse that into a one-shot `reconstruct` command. Effort is
better spent making each primitive sharper, or moving a decision the agent
keeps re-deriving into a call it can make once.

## Layout gotcha

The Python package lives at `skills/png2svg/scripts/png2svg/` — inside the
skill, so the skill is self-contained. `pyproject.toml` points the build
backend there via `tool.uv.build-backend.module-root`. **There is exactly
one copy.** Never vendor a second one into `src/`; a drifting duplicate is
worse than either copy alone.

```
skills/png2svg/          the deliverable
  SKILL.md               workflow; keep under 500 lines / 5000 tokens
  references/            loaded on demand — conventions, model, examples
  scripts/png2svg/       the engine
  scripts/*_template.py  what an agent copies and edits
examples/                real per-image reconstruction scripts
tests/                   ground-truth tests
```

## Commands

```bash
uv sync
uv run pytest                    # must stay green
uv run png2svg --help
uvx --from skills-ref agentskills validate ./skills/png2svg
```

The bundled entry point must also work with nothing installed, since that is
how the skill runs on someone else's machine:

```bash
uv run --no-project skills/png2svg/scripts/png2svg_cli.py --help
```

## Testing: validate against ground truth

Synthesise something whose answer is known, then check the code recovers it.
Render a gradient with known stops and refit it; fit a known cubic and
compare control points; build a shape with known geometry and measure it
back.

**The recurring trap in this codebase is a wrong verification harness.**
Three separate times a fitter looked broken when the checker was at fault —
each time by comparing a curve against a fixed-resolution sampling, which
puts a floor under the reported error equal to half the sample spacing.
Symptoms: an error that will not go below some value no matter how tight the
tolerance, or a straight line that reports 1.26px of deviation.

So: when a result looks wrong, **check the instrument before changing the
code**. `references/conventions.md` §12 lists the known ones — renderer
quarter-pixel quantisation, the mask's weakness on light-against-dark.

## Determinism is a hard requirement

Same input, same output, byte for byte — measurement, model and SVG. No RNG,
no iteration-order dependence, no wall-clock. `validate` checks the SVG
regenerates identically; verify the measurement path yourself when you touch
it:

```bash
uv run python examples/build_p_model.py && cp work/p/project.json /tmp/a.json
uv run python examples/build_p_model.py && cmp /tmp/a.json work/p/project.json
```

## Keep the docs with the code

The skill's docs are the interface. An agent that installs this reads
`SKILL.md` and the `references/`, not the source — so a primitive that is not
documented there effectively does not exist, and it will go and write 300
lines by hand instead. When you add or change a primitive, update:

- `SKILL.md` — the workflow and the primitives table
- `references/conventions.md` — any new rule that cost you time
- `references/model.md` — schema changes
- `references/examples.md` — if a reconstruction taught something general
- `scripts/measure_template.py` — it must reflect the current pipeline

Write down what went wrong, not just what works. The conventions file is
valuable precisely because it is a list of mistakes already paid for.

## Source artwork

`work/` is gitignored: it holds private test artwork, per-image scripts,
generated SVGs and large comparison renders. Only Keep, IN and P Auto are
published under `examples/`, with their approved source images in
`examples/assets/`. Do not move any other artwork or reconstruction out of
`work/` without confirming its redistribution rights.
