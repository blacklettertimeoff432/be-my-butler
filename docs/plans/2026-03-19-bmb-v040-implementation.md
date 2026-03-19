# BMB v0.4.0 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Upgrade BMB with 6 features: OMX cross-model fix, discipline embedding, model upgrade, consultant sync, session-end prep, parallel sessions, and visual brainstorming.

**Architecture:** Each feature is independent — modify agent .md files, cross-model-run.sh, bmb.md pipeline, and monitor/consultant agents. No new files except parallel-manifest schema.

**Tech Stack:** Bash (cross-model-run.sh), Markdown (agent definitions), tmux (pipeline orchestration)

**Design Doc:** `docs/plans/2026-03-19-bmb-v040-design.md`

---

## File Structure

| File | Responsibility | Changes |
|------|---------------|---------|
| `bmb-system/scripts/cross-model-run.sh` | Cross-model invocation | Add omx cleanup + MCP disable |
| `agents/bmb-executor.md` | Executor agent | model→opus, add discipline |
| `agents/bmb-frontend.md` | Frontend agent | model→opus, add discipline |
| `agents/bmb-tester.md` | Tester agent | model→opus, add discipline |
| `agents/bmb-verifier.md` | Verifier agent | model→opus, add discipline |
| `agents/bmb-architect.md` | Architect agent | model→opus, add discipline |
| `agents/bmb-simplifier.md` | Simplifier agent | model→opus, add discipline |
| `agents/bmb-analyst.md` | Analyst agent | model→opus |
| `agents/bmb-writer.md` | Writer agent | model→opus |
| `agents/bmb-consultant.md` | Consultant agent | model→opus, add feed_update handler |
| `agents/bmb-monitor.md` | Monitor agent | add consultant heartbeat |
| `skills/bmb/bmb.md` | Pipeline orchestration | session-end prep, parallel mode, visual brainstorm, consultant event |

---

## Task 1: OMX Cross-Model Fix

**Files:**
- Modify: `bmb-system/scripts/cross-model-run.sh:17-32` (add omx cleanup)
- Modify: `bmb-system/scripts/cross-model-run.sh:203-250` (add MCP disable)

- [ ] **Step 1: Add omx cleanup before codex invocations**

In `cross-model-run.sh`, after the preflight check section (line ~200) and before the `case "$PROVIDER"` block, add:

```bash
# --- OMX cleanup: kill orphaned processes before invocation (v0.4.0) ---
if command -v omx &>/dev/null; then
  omx cleanup 2>/dev/null || true
fi
```

- [ ] **Step 2: Add MCP disable for cross-model codex exec**

In the codex provider section, modify the `_run_codex()` function to disable MCP servers. Change the `codex exec` command lines to include `-c 'mcp_servers={}'`:

```bash
# Line ~224 (with OUTPUT_FILE)
timeout "$TIMEOUT" codex exec $MODEL_ARGS -c 'mcp_servers={}' --full-auto -C "$WORKDIR" "$FULL_PROMPT" > "$output_tmp" 2>"$stderr_tmp"

# Line ~249 (without OUTPUT_FILE)
exec timeout "$TIMEOUT" codex exec $MODEL_ARGS -c 'mcp_servers={}' --full-auto -C "$WORKDIR" "$FULL_PROMPT"
```

- [ ] **Step 3: Test cross-model-run.sh syntax**

Run: `bash -n ~/Projects/bmb/bmb-system/scripts/cross-model-run.sh`
Expected: No syntax errors

- [ ] **Step 4: Verify omx cleanup works**

Run: `omx cleanup 2>&1`
Expected: Cleanup output (may say "no orphaned processes" — that's fine)

- [ ] **Step 5: Commit**

```bash
git add bmb-system/scripts/cross-model-run.sh
git commit -m "fix(v0.4.0): add omx cleanup + disable MCP for cross-model invocations"
```

---

## Task 2: Agent Model Upgrade (all 9 non-Monitor agents → opus)

**Files:**
- Modify: `agents/bmb-analyst.md:4` (model: sonnet → opus)
- Modify: `agents/bmb-writer.md:4` (model: sonnet → opus)
- Modify: `agents/bmb-consultant.md:4` (model: sonnet → opus)

- [ ] **Step 1: Update model frontmatter in analyst, writer, consultant**

For each of these 3 files, change `model: sonnet` to `model: opus`:

`agents/bmb-analyst.md` line 4:
```yaml
model: opus
```

`agents/bmb-writer.md` line 4:
```yaml
model: opus
```

`agents/bmb-consultant.md` line 4:
```yaml
model: opus
```

Note: The other 6 agents (architect, executor, verifier, tester, frontend, simplifier) already have `model: opus`.

- [ ] **Step 2: Commit**

```bash
git add agents/bmb-analyst.md agents/bmb-writer.md agents/bmb-consultant.md
git commit -m "feat(v0.4.0): upgrade analyst, writer, consultant to opus (1M context)"
```

---

## Task 3: Discipline Embedding — Executor + Frontend

**Files:**
- Modify: `agents/bmb-executor.md` (append discipline section)
- Modify: `agents/bmb-frontend.md` (append discipline section)

- [ ] **Step 1: Add Discipline Rules to bmb-executor.md**

Append the following section after the "Context Efficiency Protocol" section (end of file):

```markdown

## Discipline Rules (Superpowers v5.0)

### Verification Gate
Before ANY completion claim in exec-result.md:
1. IDENTIFY: What command proves this claim?
2. RUN: Execute it fresh (not from cache or previous run)
3. READ: Full output, check exit code and failure count
4. VERIFY: Does output actually confirm the claim?
5. ONLY THEN: Write the completion status

RED FLAGS — STOP if you catch yourself:
- Using "should work", "probably passes", "seems fine"
- Expressing satisfaction ("Great!", "Done!") before running verification
- Trusting a previous test run without re-running
- Claiming "tests pass" without showing test output in this session

### Debugging Discipline
When a fix attempt fails:
- Phase 1 (root cause investigation) MUST complete before ANY fix attempt
- Trace the bug backwards: where does the bad value originate?
- One variable at a time — never bundle multiple fixes
- **3-fix limit**: If 3 consecutive fix attempts fail → STOP
  - Report architectural concern to Lead via exec-result.md
  - Do NOT attempt a 4th fix without Lead guidance
```

- [ ] **Step 2: Add identical Discipline Rules to bmb-frontend.md**

Append the same section to `agents/bmb-frontend.md` after the "Context Efficiency Protocol" section.

- [ ] **Step 3: Commit**

```bash
git add agents/bmb-executor.md agents/bmb-frontend.md
git commit -m "feat(v0.4.0): add verification gate + debugging discipline to executor/frontend"
```

---

## Task 4: Discipline Embedding — Tester

**Files:**
- Modify: `agents/bmb-tester.md` (append discipline section)

- [ ] **Step 1: Add TDD Discipline to bmb-tester.md**

Append after the "Context Efficiency Protocol" section:

```markdown

## Discipline Rules (Superpowers v5.0)

### TDD Red-Green Discipline
For each test:
1. **RED**: Write the failing test first
2. **Verify RED**: Run it — must fail for the expected reason (feature missing, NOT typo)
3. **GREEN**: Write minimal code to pass
4. **Verify GREEN**: Run it — must pass, all other tests still pass
5. Show RED failure and GREEN pass evidence in test-result-claude.md

### Testing Anti-Patterns — AVOID
- **Testing mock behavior**: Asserting a mock exists instead of testing real component behavior
- **Test-only methods in production**: If a method (destroy, cleanup) is only used by tests, move it to test utilities
- **Mocking without understanding**: Don't mock methods that have required side effects
- **Incomplete mocks**: Partial mocks that miss fields the real API returns → false confidence

### Verification Gate
Before writing test-result-claude.md status as PASS:
1. Run full test suite — show output with pass/fail counts
2. Run coverage analysis — show actual percentage
3. Check business invariant coverage — show per-rule status
4. All three must pass before writing PASS status
```

- [ ] **Step 2: Commit**

```bash
git add agents/bmb-tester.md
git commit -m "feat(v0.4.0): add TDD discipline + anti-patterns to tester"
```

---

## Task 5: Discipline Embedding — Verifier, Architect, Simplifier

**Files:**
- Modify: `agents/bmb-verifier.md` (append discipline section)
- Modify: `agents/bmb-architect.md` (append discipline section)
- Modify: `agents/bmb-simplifier.md` (append discipline section)

- [ ] **Step 1: Add Verification Gate to bmb-verifier.md**

Append after the "Context Efficiency Protocol" section:

```markdown

## Discipline Rules (Superpowers v5.0)

### Verification Gate (Strengthened)
This agent's entire purpose is verification. Apply these additional checks:
- NEVER use "should", "probably", "likely" in verification results
- Every checklist item MUST have actual command output as evidence
- If a check cannot be run (no tooling), mark as SKIPPED with reason — never assume PASS
- Treat agent success reports (exec-result.md) as CLAIMS, not facts — verify independently
```

- [ ] **Step 2: Add Scope Discipline to bmb-architect.md**

Append after the "Context Efficiency Protocol" section:

```markdown

## Discipline Rules (Superpowers v5.0)

### YAGNI Principle
- Remove unnecessary features from ALL designs — "will we need this?" → probably not
- Each design element must justify its existence with a concrete use case
- Prefer simpler alternatives unless complexity is explicitly required

### Scope Check
Before writing plan-to-exec.md, assess:
- Does this design cover multiple independent subsystems?
- If YES → decompose into separate council debates + separate handoffs
- Each handoff should produce independently testable work
- Flag to Lead if scope seems too large for a single execution cycle
```

- [ ] **Step 3: Add Minimal Change Discipline to bmb-simplifier.md**

Append after the "Context Efficiency Protocol" section:

```markdown

## Discipline Rules (Superpowers v5.0)

### Minimal Viable Change
- One simplification at a time — never bundle refactors
- If a change touches more than 3 files, split it into smaller changes
- Each change must be independently revertable
- "Better" is not a reason to change — must fix a concrete issue (dead code, duplication, naming inconsistency)
```

- [ ] **Step 4: Commit**

```bash
git add agents/bmb-verifier.md agents/bmb-architect.md agents/bmb-simplifier.md
git commit -m "feat(v0.4.0): add discipline rules to verifier, architect, simplifier"
```

---

## Task 6: Consultant Sync + Monitor Watchdog

**Files:**
- Modify: `agents/bmb-monitor.md` (add consultant heartbeat + watchdog mode)
- Modify: `agents/bmb-consultant.md` (add feed_update handler)

- [ ] **Step 1: Add consultant heartbeat + watchdog to bmb-monitor.md**

After the "Monitoring Loop" section (line ~68), add two new sections:

```markdown

## Consultant Feed Heartbeat (v0.4.0)

In addition to per-watch-item monitoring, track the consultant feed file:

### Feed Monitoring
Every monitoring cycle (default 30s), check `.bmb/consultant-feed.md`:
1. `stat .bmb/consultant-feed.md` — get current mtime
2. Compare with `last_feed_mtime`
3. If changed:
   - SendMessage to Consultant: `{"type":"feed_update","source":"monitor","ts":"HH:MM"}`
   - Update `last_feed_mtime`
4. If unchanged: skip (no message)

### State Tracking Addition
Add to per-session state:
- `last_feed_mtime` — last observed mtime of consultant-feed.md

### Rules
- Feed heartbeat runs regardless of blind phase status (feed itself is filtered by Lead)
- Never read feed file content — metadata only (mtime check)
- This does NOT replace Lead's direct SendMessage — it supplements it for idle periods

## Watchdog Mode (v0.4.0)

Monitor acts as a watchdog for the entire tmux session, detecting orphaned and crashed panes.

### tmux Pane Sweep (every 60s)
Every monitoring cycle, scan all tmux panes:
```bash
tmux list-panes -s -F '#{pane_id} #{pane_pid} #{pane_dead}'
```

For each pane NOT matching known pane IDs (Lead pane, Consultant pane from `.bmb/consultant-pane-id`):
1. If `pane_dead=1` → SendMessage to Lead:
   `{"type":"watchdog","event":"pane_dead","pane":"ID","ts":"HH:MM"}`
2. If pane alive but no registered watch item for its PID → SendMessage to Lead:
   `{"type":"watchdog","event":"untracked_pane","pane":"ID","pid":N,"ts":"HH:MM"}`

### Known Pane Resolution
On startup, read:
- Own parent pane ID (Lead's pane)
- `.bmb/consultant-pane-id` (Consultant's pane)
These are excluded from watchdog sweep — they are expected long-lived panes.

### Lead Nudge Escalation
If a `stalled` or `process_died` report receives no acknowledgment from Lead within 120s:
1. Re-send with escalation flag:
   `{"type":"watchdog","event":"nudge_repeat","original_event":"stalled","agent":"NAME","nudge_count":N,"ts":"HH:MM"}`
2. Maximum 3 nudges per event — after 3rd, stop to avoid noise
3. Track per-event: `nudge_count`, `last_nudge_ts`, `acked` (boolean)

### Nudge Acknowledgment
Lead acknowledges by sending:
`{"ack":"stalled","agent":"NAME"}`
On receipt, set `acked=true` for that event and stop nudging.

### State Tracking Addition
Add to per-session state:
- `known_panes` — set of Lead + Consultant pane IDs (excluded from sweep)
- `nudge_tracker` — per-event: {event_type, agent, nudge_count, last_nudge_ts, acked}

### Rules
- Watchdog uses `tmux list-panes` only — metadata, no content
- Never kill panes — only report to Lead
- Nudge escalation is capped at 3 per event to prevent alert fatigue
- Watchdog runs independently of registered watch items
```

- [ ] **Step 2: Add feed_update handler to bmb-consultant.md**

After the "Sync Protocol" section (line ~40), add:

```markdown

### Monitor Feed Heartbeat (v0.4.0)
Monitor sends periodic `feed_update` messages when `consultant-feed.md` changes:
```json
{"type":"feed_update","source":"monitor","ts":"HH:MM"}
```
On receiving this message:
1. Re-read `.bmb/consultant-feed.md` from your last-read position
2. Update your internal context with any new events
3. If the user is actively in your pane, proactively brief them on new developments
4. If the user is not in your pane, silently update — brief them when they return
```

- [ ] **Step 3: Add feed_update + watchdog events to Consultant's SendMessage event list**

In the "Lead → Consultant message types" section, add to the JSON list:

```json
{"type":"feed_update","source":"monitor","ts":"HH:MM"}
```

- [ ] **Step 4: Add watchdog event handling to bmb.md Lead protocol**

In the CONSULTANT EVENT TEMPLATES section of `skills/bmb/bmb.md`, add:

```json
{"type":"watchdog","event":"pane_dead","pane":"ID","ts":"HH:MM"}
{"type":"watchdog","event":"untracked_pane","pane":"ID","pid":N,"ts":"HH:MM"}
{"type":"watchdog","event":"nudge_repeat","original_event":"EVENT","agent":"NAME","nudge_count":N,"ts":"HH:MM"}
```

And add Lead handling logic after receiving watchdog events:
```markdown
### Watchdog Event Handling (v0.4.0)
On receiving watchdog events from Monitor:
- `pane_dead`: Kill the dead pane (`tmux kill-pane -t {pane} 2>/dev/null`), log to session-log
- `untracked_pane`: Investigate — is this a legitimate agent? If not, kill it
- `nudge_repeat`: Re-check the stalled/died agent. If still stuck, take recovery action or log degradation
- Always acknowledge with: `{"ack":"EVENT","agent":"NAME"}` to stop further nudges
```

- [ ] **Step 5: Commit**

```bash
git add agents/bmb-monitor.md agents/bmb-consultant.md skills/bmb/bmb.md
git commit -m "feat(v0.4.0): add monitor watchdog + consultant heartbeat + lead nudge escalation"
```

---

## Task 7: Session-End Preparation System

**Files:**
- Modify: `skills/bmb/bmb.md:849-967` (Step 12 extension)

- [ ] **Step 1: Extend Step 12 with session handover**

In `skills/bmb/bmb.md`, after step 12 item 10 (carry-forward.md, around line 950), and before item 11 (worktree cleanup), insert:

```markdown

10.5. **Session Handover System (v0.4.0)**:
    Generate next-session preparation with user confirmation.
    ```bash
    # Generate next-session-plan.md
    cat > .bmb/next-session-plan.md << PLAN_EOF
    # Next Session Plan
    Generated: $(date '+%Y-%m-%d %H:%M KST')
    Previous Session: ${SESSION_ID}

    ## Completed This Session
    $(grep '| COMPLETE\|| PASS' .bmb/session-log.md 2>/dev/null | sed 's/^/- [x] /' || echo "- [x] Session completed")

    ## Next Steps
    $(grep 'Remaining\|TODO\|Unfinished' .bmb/sessions/${SESSION_ID}/carry-forward.md 2>/dev/null | sed 's/^//' || echo "- No pending items")

    ## One-Line Prompt
    > BMB: {Lead fills this with specific next task description}
    PLAN_EOF
    ```

    Present to user with AskUserQuestion:
    - **YES**: Finalize plan, display the one-line prompt prominently
    - **NO**: Delete `.bmb/next-session-plan.md`, end session normally
    - **Custom**: User modifies, Lead regenerates

    ```
    AskUserQuestion:
    question: "다음 세션 준비 계획을 확인해주세요. 수정이 필요하면 직접 입력해주세요."
    options:
      - label: "확인, 이대로 저장"
        description: "작업계획서와 한줄 프롬프트를 저장합니다."
      - label: "필요 없음"
        description: "작업계획서 없이 세션을 종료합니다."
    ```
```

- [ ] **Step 2: Commit**

```bash
git add skills/bmb/bmb.md
git commit -m "feat(v0.4.0): add session-end preparation system with user confirmation"
```

---

## Task 8: Parallel Session Native Integration

**Files:**
- Modify: `skills/bmb/bmb.md` (add SESSION_MODE routing + sub/consolidation pipelines)
- Modify: `skills/bmb/SKILL.md` (add SESSION_MODE documentation)

- [ ] **Step 1: Add SESSION_MODE detection to Step 1 (Setup)**

In `skills/bmb/bmb.md`, after the session ID generation in Step 1 (around line 99), add:

```markdown
# --- SESSION_MODE detection (v0.4.0) ---
# Check if this is a sub or consolidation session
SESSION_MODE="standalone"
if echo "$USER_PROMPT" | grep -qi 'BMB sub:'; then
  SESSION_MODE="sub"
  # Read manifest from parent session
  if [ -f ".bmb/parallel-manifest.json" ]; then
    TRACK_ID=$(echo "$USER_PROMPT" | sed 's/.*BMB sub: *//' | head -c 20 | tr ' ' '-')
  fi
elif echo "$USER_PROMPT" | grep -qi 'BMB consolidate:'; then
  SESSION_MODE="consolidation"
fi
echo "SESSION_MODE=$SESSION_MODE" >> .bmb/sessions/${SESSION_ID}/env
```

- [ ] **Step 2: Add track splitting logic to Step 2 (Brainstorm)**

After the brainstorming section in Step 2, add:

```markdown
# --- Parallel track assessment (v0.4.0) ---
# After brainstorming, assess if work can be split into independent tracks
if [ "$SESSION_MODE" = "standalone" ]; then
  # Ask Lead to assess: "Can this work be split into independent tracks?"
  # If YES:
  #   1. Generate .bmb/parallel-manifest.json with track definitions
  #   2. Generate per-track one-line prompts
  #   3. Generate consolidation prompt
  #   4. Present all prompts to user
  # If NO: continue as standalone
fi
```

- [ ] **Step 3: Add pipeline routing for sub mode**

After the SESSION_MODE detection, add routing logic:

```markdown
# --- Pipeline routing by SESSION_MODE (v0.4.0) ---
case "$SESSION_MODE" in
  standalone)
    # Normal 12-step pipeline (existing behavior, 100% backward compatible)
    ;;
  sub)
    # Reduced pipeline: skip Steps 2-3, scoped Steps 4-10
    # Step 10 Writer: staging docs ONLY — docs/ direct edit FORBIDDEN
    # Step 12: update manifest status instead of carry-forward
    ;;
  consolidation)
    # Lightest pipeline: skip Steps 2-4, 9
    # Step 5: merge worktree changes from all tracks
    # Step 6-7: integration test + verify (full scope)
    # Step 10: merge staging docs + final docs/ update
    ;;
esac
```

- [ ] **Step 4: Add consolidation recipe to recipe table**

In the RECIPE REFERENCE section of bmb.md, add:

```markdown
| consolidation | merge worktrees → integration tester(cross) → verifier(cross) → writer(merge staging) → cleanup |
```

- [ ] **Step 5: Add parallel session documentation to SKILL.md**

Add a section to `skills/bmb/SKILL.md` documenting SESSION_MODE:

```markdown
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
```

- [ ] **Step 6: Commit**

```bash
git add skills/bmb/bmb.md skills/bmb/SKILL.md
git commit -m "feat(v0.4.0): add parallel session native integration (SESSION_MODE)"
```

---

## Task 9: Visual Brainstorming Integration

**Files:**
- Modify: `skills/bmb/bmb.md` (Step 2 extension)

- [ ] **Step 1: Add visual companion gate to Step 2**

In `skills/bmb/bmb.md`, at the beginning of the Step 2 brainstorming section, add:

```markdown
# --- Visual Brainstorming Companion (v0.4.0) ---
# If upcoming questions involve visual content (mockups, diagrams, comparisons),
# offer to start the Superpowers brainstorm server.
#
# Decision: "Would the user benefit from seeing visuals during brainstorming?"
# If YES:
SUPERPOWERS_SCRIPTS=$(ls -d "$HOME/.claude/plugins/cache/superpowers-dev/superpowers"/*/skills/brainstorming/scripts 2>/dev/null | head -1)
if [ -n "$SUPERPOWERS_SCRIPTS" ] && [ -f "$SUPERPOWERS_SCRIPTS/start-server.sh" ]; then
  BRAINSTORM_SCREEN_DIR=".bmb/brainstorm-screens/${SESSION_ID}"
  mkdir -p "$BRAINSTORM_SCREEN_DIR"
  SERVER_INFO=$("$SUPERPOWERS_SCRIPTS/start-server.sh" --project-dir "$(pwd)" 2>/dev/null) || SERVER_INFO=""
  if [ -n "$SERVER_INFO" ]; then
    # Present URL to user
    echo "Visual brainstorming companion available at: $(echo "$SERVER_INFO" | grep -o 'http://[^ ]*')"
  fi
fi
# If NO: proceed with terminal-only brainstorming (existing behavior)
#
# After Step 3 approval:
if [ -n "$SUPERPOWERS_SCRIPTS" ] && [ -f "$SUPERPOWERS_SCRIPTS/stop-server.sh" ]; then
  "$SUPERPOWERS_SCRIPTS/stop-server.sh" 2>/dev/null || true
fi
```

- [ ] **Step 2: Commit**

```bash
git add skills/bmb/bmb.md
git commit -m "feat(v0.4.0): add visual brainstorming companion integration"
```

---

## Task 10: Add Consultant Event Template for v0.4.0

**Files:**
- Modify: `skills/bmb/bmb.md` (consultant event templates section)

- [ ] **Step 1: Add new event template**

In the CONSULTANT EVENT TEMPLATES section of bmb.md (around line 1005), add:

```json
{"event":"session_handover","step":"12","plan_path":".bmb/next-session-plan.md","ts":"HH:MM"}
{"event":"parallel_tracks_generated","step":"2","track_count":N,"manifest":".bmb/parallel-manifest.json","ts":"HH:MM"}
```

- [ ] **Step 2: Commit**

```bash
git add skills/bmb/bmb.md
git commit -m "feat(v0.4.0): add consultant event templates for session handover + parallel tracks"
```

---

## Task 11: Update Design Doc + Final Verification

**Files:**
- Modify: `docs/plans/2026-03-19-bmb-v040-design.md` (update OMX section with actual CLI)

- [ ] **Step 1: Update OMX design section**

Change the "Script Changes" section in Feature #3 to reflect actual `omx` capabilities:
- `omx exec` → `omx cleanup` + `codex exec -c 'mcp_servers={}'` (no --timeout-ms, no --no-mcp)
- Shell `timeout` remains for timeout management

- [ ] **Step 2: Verify all agent files have correct model**

Run: `grep -n "^model:" agents/bmb-*.md`
Expected: All non-monitor agents show `model: opus`, monitor shows `model: haiku`

- [ ] **Step 3: Verify all discipline sections exist**

Run: `grep -l "Discipline Rules" agents/bmb-*.md`
Expected: executor, frontend, tester, verifier, architect, simplifier (6 files)

- [ ] **Step 4: Verify cross-model-run.sh syntax**

Run: `bash -n bmb-system/scripts/cross-model-run.sh`
Expected: No errors

- [ ] **Step 5: Final commit**

```bash
git add docs/plans/2026-03-19-bmb-v040-design.md
git commit -m "docs(v0.4.0): update design doc with actual OMX CLI capabilities"
```

---

## Summary

| Task | Description | Files | Commits |
|------|------------|-------|---------|
| 1 | OMX cross-model fix | cross-model-run.sh | 1 |
| 2 | Model upgrade (3 agents) | analyst, writer, consultant .md | 1 |
| 3 | Discipline: executor + frontend | executor.md, frontend.md | 1 |
| 4 | Discipline: tester | tester.md | 1 |
| 5 | Discipline: verifier, architect, simplifier | verifier.md, architect.md, simplifier.md | 1 |
| 6 | Consultant sync (monitor heartbeat) | monitor.md, consultant.md | 1 |
| 7 | Session-end preparation | bmb.md | 1 |
| 8 | Parallel session native | bmb.md, SKILL.md | 1 |
| 9 | Visual brainstorming | bmb.md | 1 |
| 10 | Consultant event templates | bmb.md | 1 |
| 11 | Design doc update + verification | design doc | 1 |

**Total**: 11 tasks, 11 commits, ~15 files modified
