---
name: bmb
description: "Be-my-butler (BMB) full A-to-Z multi-agent pipeline — 12 steps with cross-model council, blind verification, simplification, and session continuity. Keywords: butler, pipeline, agent team, cross-model, blind verification, council debate."
---

# /BMB

Read and follow `~/.claude/skills/bmb/bmb.md` for the full 12-step pipeline.

Related skills (invoke separately):
- `/BMB-setup` — configure project settings
- `/BMB-brainstorm` — consulting session with Lead + Consultant
- `/BMB-refactoring` — parallel refactoring with cross-model review
- `/BMB-status` — project/idea dashboard and nudge system

## Parallel Sessions (v0.4.0)

BMB supports parallel execution via SESSION_MODE:

- **standalone** (default): Full 12-step pipeline. 100% backward compatible.
- **sub**: Parallel track worker. Reduced pipeline (skip brainstorm/approval). Code only, docs/ direct edit forbidden. Uses own worktree.
- **consolidation**: Merge-only recipe. No code writing. Merges worktree changes + staging docs.

### Usage
1. During brainstorming, Lead assesses if work can be split into independent tracks
2. If yes, generates `parallel-manifest.json` + per-track prompts
3. User starts each track in a separate terminal: `BMB sub: {track description}`
4. After all tracks complete: `BMB consolidate: {manifest description}`
