# What's New in BMB v0.4.0

**6-Feature Upgrade — OMX cross-model fix, Superpowers discipline, visual brainstorming, session continuity, parallel sessions, Monitor watchdog**

---

## The Problems This Solves

1. **100% cross-model timeout** — `codex exec` loaded 8 MCP servers before executing prompts, causing every cross-model invocation to hang.
2. **No agent discipline** — agents lacked concrete verification checklists, allowing "should work" claims without evidence.
3. **Text-only brainstorming** — design discussions had no visual tools for mockups or architecture diagrams.
4. **No session continuity** — pipeline ended with no structured handover for the next session.
5. **Unsafe parallelism** — concurrent BMB sessions sharing `.bmb/` caused handoff file collisions.
6. **Silent agent stalls** — Monitor had no way to detect orphaned panes or escalate nudges.

---

## What Changed

### Feature 1: OMX Cross-Model Fix

Cross-model invocations now disable all MCP servers via `$MCP_DISABLE_ARGS`, eliminating the primary timeout cause. `omx cleanup` runs before each invocation to clear zombie processes.

- **Before**: 100% timeout rate (4/4 sessions)
- **After**: Clean execution with immediate Claude-only fallback on failure

### Feature 2: Superpowers Discipline Embedding

Battle-tested discipline skills from Superpowers v5.0 are cherry-picked and embedded directly in agent prompts (~150-300 tokens per agent):

| Agent | Disciplines |
|-------|------------|
| Executor, Frontend | Verification Gate (5-step) + Debugging Discipline (3-fix limit) |
| Tester | TDD Red-Green checklist + Testing Anti-patterns |
| Verifier | Strengthened Verification Gate |
| Architect | YAGNI principle + Scope Check |
| Simplifier | Minimal Viable Change |

All agents upgraded to **Opus 4.6 (1M context)** with effort max.

### Feature 3: Visual Brainstorming

Step 2 now supports a browser-based visual companion via the Superpowers brainstorm server:

- Architecture diagrams, trade-off matrices, UI mockups
- Per-question routing: visual topics → browser, concepts → terminal
- Auto-start at Step 2, auto-stop after Step 3

### Feature 4: Session-End Preparation

Step 12 auto-generates `.bmb/next-session-plan.md` with:

- Completed items from this session
- Discovered follow-up tasks (prioritized)
- Recommended recipe and scope for next session
- One-line start prompt ready to paste

### Feature 5: Parallel Session Native Integration

New `SESSION_MODE` enum enables safe concurrent pipelines:

| Mode | Behavior |
|------|----------|
| `standalone` | Default, 100% backward compatible |
| `sub` | Parallel track worker — code only, no `docs/` edits |
| `consolidation` | Merge only — no code, docs merge only |

Step 2 extension assesses parallelism and auto-generates track prompts + consolidation prompt.

### Feature 6: Monitor Watchdog Mode

The Haiku Monitor agent gains two capabilities:

- **Pane sweep** — periodic scan for orphaned tmux panes from crashed agents
- **Nudge escalation** — escalating warnings (info → warn → critical) for stalled agents

---

## Postfix Fixes (v0.4.0 audit)

| Fix | Description |
|-----|-------------|
| Timeout hoisting | `$CLAUDE_TIMEOUT`, `$CROSS_TIMEOUT`, `$WRITER_TIMEOUT` moved to Step 1 (were undefined or late-defined) |
| Architect timeout | Fixed from `$CROSS_TIMEOUT` → `$CLAUDE_TIMEOUT` (architect is a Claude agent) |
| Architect Write tool | Added Write to bmb-architect tools (needed for councils/, handoffs/) |
| Telegram security | Added security comment for token-in-URL visibility |
| Monitor config docs | Documented `monitor` section in configuration.md |
| ANALYST_TIMEOUT | Unified to `bmb_config_get` (was raw python3 inline) |
| Data alignment | Synced defaults.json, configuration.md, and bmb.md fallback values |

---

## Upgrade

```bash
curl -fsSL https://raw.githubusercontent.com/project820/be-my-butler/main/install.sh | bash
bmb doctor
```

No breaking changes. Existing `.bmb/config.json` files work without modification.
