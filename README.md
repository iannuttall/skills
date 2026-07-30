# Ian's agent skills

Agent skills I use in my own projects. Each skill is self-contained under
`skills/`, including any scripts or references it needs.

Install one skill:

```bash
npx skills add iannuttall/skills --skill png2svg
```

Run the command without `--skill` to choose interactively:

```bash
npx skills add iannuttall/skills
```

## Available skills

| Skill | What it does |
|---|---|
| [png2svg](docs/png2svg.md) | Reconstructs geometric PNG logos as small, editable SVGs |
| [chrome-webstore-release-blueprint](skills/chrome-webstore-release-blueprint/SKILL.md) | Sets up Chrome Web Store release automation |
| [find-skills](skills/find-skills/SKILL.md) | Finds and installs agent skills through Playbooks |
| [get-url](skills/get-url/SKILL.md) | Fetches a URL as clean markdown through Playbooks |

## Working on the skills

The installable folders stay under `skills/<name>/`. Tests, examples and
private working files sit outside those folders so users only install what
the agent needs.

png2svg has the largest development harness:

```bash
uv sync
uv run pytest
uvx --from skills-ref agentskills validate ./skills/png2svg
```

Private artwork and generated comparison files belong in the ignored
`work/` directory.
