# car-reel-editor

Loomelic Media's Claude skill for editing dealership car reels: vertical 4K, ~18 s, 3D motion graphics, every clip stabilized and slowed to exactly 50%, beat-cut to Epidemic Sound, HyperFrames by default.

The skill lives at [`skills/car-reel-editor/SKILL.md`](skills/car-reel-editor/SKILL.md).

## How this repo is kept current

The canonical copy of the skill is the one installed in Denham's Claude account (`/car-reel-editor`). Every Monday a scheduled Claude task:

1. reads the installed skill,
2. compares it to `skills/car-reel-editor/SKILL.md` here,
3. commits and pushes any change with a dated message, and
4. mirrors the same text to the "Car Reel Editor skill" doc in the Claude *Skills* project.

New rules are added to the installed skill when Denham approves or corrects a reel; this repo and the project doc follow it, not the other way around. If you change `SKILL.md` here by hand, the next sync overwrites it unless the installed skill was updated too.

## Installing elsewhere

Copy `skills/car-reel-editor/` into any Claude skills directory (Claude Code: `.claude/skills/` in a repo, or `~/.claude/skills/` for all projects), or add it as a skill in claude.ai.

## Changelog

See `CHANGELOG.md`; each weekly sync appends a line when the skill changed.
