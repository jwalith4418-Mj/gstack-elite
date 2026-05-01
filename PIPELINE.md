# Pipeline v4.0 — Dual-Mode Execution + Self-Heal + Self-Improve

## Full Pipeline

```
INPUT
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 0 — CONTEXT LOAD                                              │
│   Standard: < 500ms  |  Hackathon: < 100ms                         │
│                                                                     │
│  • Run preamble bash                                                │
│  • Extract: BRANCH, SLUG, SID, LEARNINGS[:5], DELTAS[:3]           │
│  • Load prompt-mutations.jsonl (active overrides)                  │
│  • Load algo-solutions.jsonl (past solved patterns)                │
│  • Reset circuit breaker (new session = fresh start)               │
│  • Detect MODE from prior session context if task is continuation   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 1 — DECOMPOSITION + MODE ROUTING  [PLANNER]                   │
│                                                                     │
│  • Classify: AI_MODE or ALGO_MODE (dual-mode router, CLAUDE.md)    │
│  • AI_MODE:   extract intent → skill_chain → parallel_groups       │
│  • ALGO_MODE: extract N, constraints → pattern match → algo_id     │
│  • Declare gates (one-way doors → GATE: human)                     │
│  • Estimate tokens per tier                                         │
│  • Emit PLAN block → pause if GATE: human                          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ approved / auto-pass
              ┌────────────────┴──────────────────────┐
              │ AI_MODE                                │ ALGO_MODE
              ▼                                        ▼
┌─────────────────────────┐            ┌─────────────────────────────┐
│ STAGE 2a — PROMPT ASSY  │            │ STAGE 2b — ALGO PREP        │
│ • Select SP template     │            │ • Load ALGO-XX template     │
│ • Apply mutations        │            │ • Verify complexity vs N    │
│ • Fill {VARS}            │            │   (R11 gate — fail → reroute│
│ • Inject top-3 learnings │            │    to better algorithm)     │
│ • Set token_budget       │            │ • Inject past solutions for │
│ • Terse: strip prose     │            │   similar patterns          │
└────────────┬────────────┘            └──────────────┬──────────────┘
             │                                         │
             ▼                                         ▼
┌─────────────────────────┐            ┌─────────────────────────────┐
│ STAGE 3a — EXECUTION    │            │ STAGE 3b — ALGO SOLVE       │
│  [EXECUTOR]             │            │  [ALGO-CORE]                │
│                         │            │                             │
│ Sequential:             │            │ 1. PATTERN: match problem   │
│   skill_1→skill_2→...   │            │    to ALGO_CORE registry    │
│                         │            │                             │
│ Parallel group:         │            │ 2. SELECT: algo_id +        │
│   [A||B||C] → merge     │            │    fallback_algo_id         │
│                         │            │                             │
│ Per skill:              │            │ 3. INSTANTIATE: fill        │
│ • Check circuit breaker │            │    template with problem    │
│ • If lam_required:      │            │    variables               │
│   → Stage 3c (LAM)      │            │                             │
│ • Execute + track       │            │ 4. VERIFY: R12 edge cases  │
│ • On timeout: partial   │            │    (5 cases, mental trace)  │
└────────────┬────────────┘            └──────────────┬──────────────┘
             │                                         │
             │               ┌──────────────────────┐  │
             │               │ STAGE 3c — LAM LAYER  │  │
             │               │ (when lam_required)   │  │
             │               │                       │  │
             │               │ • Generate LAMPlan    │  │
             │               │   (SP-14)             │  │
             │               │ • Gate irreversible   │  │
             │               │   actions → human     │  │
             │               │ • Execute sequence    │  │
             │               │ • Validate expected   │  │
             │               │   output per action   │  │
             │               │ • halt|skip|retry     │  │
             └───────────────┤                       │  │
                             └──────────────────────┘  │
                                        │               │
                         ┌──────────────┴───────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 4 — VALIDATION                                                │
│                                                                     │
│  AI_MODE:  [CRITIC] scores 4 dimensions                            │
│    quality_score ≥ 0.75 → continue                                 │
│    quality_score < 0.75 → Stage 4b (AI Debug Loop)                 │
│                                                                     │
│  ALGO_MODE: [VERIFIER] checks solution                             │
│    All examples pass + edge cases pass → continue                  │
│    Any failure → Stage 4c (Algo Debug Loop)                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ validation passed
        ┌──────────────────────┤
        │                      │ AI: quality < 0.75
        │             ┌────────┴──────────────────────────────────────┐
        │             │ STAGE 4b — AI DEBUG LOOP  [DEBUGGER]          │
        │             │  • Classify failure type (schema/evidence/    │
        │             │    scope/calibration/timeout/tool)            │
        │             │  • Inject debug template (SP-12)             │
        │             │  • Re-execute attempt 2                       │
        │             │  • Re-score [CRITIC]                          │
        │             │  • Still failing → attempt 3:                │
        │             │    /investigate on failure itself             │
        │             │  • attempt 4 → ESCALATE                      │
        │             │  • Increment session_retry_count              │
        │             │  • Check R8 circuit breaker                   │
        │             └───────────────────────────────────────────────┘
        │
        │                      │ ALGO: verification failed
        │             ┌────────┴──────────────────────────────────────┐
        │             │ STAGE 4c — ALGO DEBUG LOOP                    │
        │             │                                               │
        │             │  Failure classification:                      │
        │             │                                               │
        │             │  WRONG_ANSWER:                                │
        │             │    • Identify failing test case               │
        │             │    • Trace algorithm step-by-step             │
        │             │    • Find invariant violation                 │
        │             │    • Fix specific bug (not full rewrite)      │
        │             │    • Re-verify all 5 edge cases              │
        │             │                                               │
        │             │  TIME_LIMIT_EXCEEDED:                         │
        │             │    • Profile: which loop is O(N²)?            │
        │             │    • Escalate complexity: try next            │
        │             │      algorithm tier                           │
        │             │    • Example: BFS→Dijkstra, O(N²)→O(N log N) │
        │             │                                               │
        │             │  MEMORY_LIMIT_EXCEEDED:                       │
        │             │    • Find O(N²) space structure               │
        │             │    • Apply space optimization:                │
        │             │      rolling array / compressed DP            │
        │             │    • Reduce from 2D to 1D where possible      │
        │             │                                               │
        │             │  OVERFLOW:                                    │
        │             │    • Locate the overflowing expression        │
        │             │    • Apply ll cast or intermediate mod        │
        │             │    • Re-verify                                │
        │             │                                               │
        │             │  EDGE_CASE_FAIL:                              │
        │             │    • Add special case handler                 │
        │             │    • Or generalize algorithm to handle it     │
        │             │                                               │
        │             │  After fix: re-run VERIFIER                   │
        │             │  Max 3 attempts → then ALT-SOLVER             │
        │             │  ALT-SOLVER: try fallback_algo_id             │
        │             │  Still failing → ESCALATE with analysis       │
        │             └───────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 5 — ITERATION  (if next[] non-empty)                          │
│  • Queue skill invocations from output.next[]                       │
│  • Surface user-required decisions                                  │
│  • Max 3 auto-iterations / session (prevent loops)                  │
│  • Detect circular chains: A→B→A → BREAK at 2nd visit              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 6 — FINAL OUTPUT + LEARNING                                   │
│                                                                     │
│  AI_MODE:                                                           │
│  • Merge skill outputs (dedup findings by location)                 │
│  • Critic self-eval (SP-11) on merged output                        │
│  • Present: findings + verdict + next actions                       │
│                                                                     │
│  ALGO_MODE:                                                         │
│  • Present: solution + complexity + edge cases + confidence         │
│  • Append to algo-solutions.jsonl (for future similar problems)     │
│                                                                     │
│  BOTH MODES:                                                        │
│  • SP-09: synthesize learnings                                      │
│  • Write session-deltas.jsonl                                       │
│  • OPTIMIZER async: check mutation thresholds (R10)                 │
│  • Write optimizer-report.jsonl                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Self-Heal Pipeline

Activated when Stage 4 validation fails. Three layers of healing in order:

### Layer 1: Targeted Fix (attempt 2)

```
Classify failure → inject exact repair template → re-execute

AI failures → DEBUGGER (5 failure types, 5 templates)
Algo failures → inline fix (WA/TLE/MLE/OVF/EDGE handlers above)

Principle: minimal intervention. Fix the weakest dimension only.
Do not rewrite. Do not change algorithm unless complexity mismatch.
```

### Layer 2: Context Reduction (attempt 3)

```
Strip everything except the failing unit:
  AI_MODE:   run /investigate on the failure itself
  ALGO_MODE: reduce to minimal failing test case, re-trace

Inject:
  AI_MODE:   stripped context + only weak_dimension + SP-12
  ALGO_MODE: problem constraints + failing case + algorithm state at failure
```

### Layer 3: Escalation (attempt 4 / ALT-SOLVER)

```
AI_MODE:   ESCALATE to user with evidence bundle
           {skill, attempts[], quality_scores[], weak_dimensions[], session_id}

ALGO_MODE: Switch to fallback_algo_id
           If fallback also fails → ESCALATE:
           {problem, primary_algo, fallback_algo, failing_cases, analysis}
```

---

## Self-Improve Pipeline

Runs asynchronously at end of every session.

### Phase 1: Learning Synthesis (SP-09)

```
Input:  session findings + prior learnings + retry count + confidence outcomes
Output: new_learnings[], confirmed[], refuted[], calibration_delta

Store in learnings.jsonl:
  new_learnings → add with initial confidence
  confirmed     → boost confidence +0.1
  refuted       → mark stale (don't delete, just deprioritize)
  calibration_delta → apply to confidence scoring formula
```

### Phase 2: Pattern Promotion (OPTIMIZER)

```
After 3+ sessions:
  retry_count ≥ 2 consistently → promote failure pattern to calibration_anchor
  same error type in same skill → add to DEBUGGER's known-failure index
  same algo pattern solved correctly → add to algo-solutions.jsonl for instant recall

Thresholds:
  session_retry_count > 6 in 2 sessions → circuit breaker trips → review skill
  user_corrections ≥ 3 on same skill   → flag degraded → SP-13 mutation
  algo_accuracy < 0.80 over 5 sessions  → expand Pattern Recognition Engine
```

### Phase 3: Prompt Mutation (SP-13 → R10)

```
1. OPTIMIZER flags degraded skill (quality < 0.75, 3+ sessions)
2. SP-13 generates mutation_proposal:
   • Identify weakest structural element in prompt
   • Propose ONE targeted change (not a rewrite)
   • Define success: quality_delta ≥ +0.05 over 3 sessions
   • Define revert: quality_delta < 0 over 3 sessions
3. Write to prompt-mutations.jsonl with status: 'pending'
4. Stage 0 (next session): load active mutations → apply as override
5. Track quality_delta session by session
6. ACCEPT if mean_delta > 0.05 (status: 'accepted', write to learnings as CONFIRMED_MUTATION)
7. REVERT if mean_delta < 0 (status: 'reverted', note in learnings as REVERT_MUTATION)
```

### Phase 4: Algo Library Expansion

```
Every successfully solved and verified algo problem:
  Append to algo-solutions.jsonl:
  {
    "ts": "ISO",
    "pattern": "shortest path weighted",
    "algo_id": "ALGO-01",
    "n_constraint": 100000,
    "problem_hash": "...",
    "solution_confidence": 9,
    "edge_cases": ["single node", "disconnected graph", ...],
    "complexity_time": "O((V+E) log V)",
    "complexity_space": "O(V+E)"
  }

Stage 0 loads recent algo-solutions. PLANNER uses them to:
  • Speed up pattern matching (cache recent hits)
  • Calibrate confidence (similar problems solved before → higher initial confidence)
  • Reuse verified edge case sets for similar problems
```

---

## Competitive Mode Pipeline

Activated when ICPC / competitive context detected.

```
Time budget: 30s per problem (hackathon) | 5 min (normal)

SPEED PROTOCOL:
  00–05s: read constraints → complexity required → pattern match
  05–10s: select template → check complexity gate (R11)
  10–20s: instantiate template → fill variables
  20–25s: R12 edge cases (5 cases, 10s)
  25–30s: emit solution

CONFIDENCE GATE (R12):
  Must achieve confidence ≥ 9/10 before emitting
  If stuck at 7–8: one more targeted verification
  If stuck at < 7: escalate to fallback algorithm

SCORING LAYER:
  Correct solution:   +10 pts
  Optimal complexity: +5 pts  (beats stated constraint × 10)
  Edge cases clean:   +3 pts
  Code elegance:      +2 pts  (terse, no redundant vars, clear variable names)
  Total possible:     20 pts per problem

Emit scoring with every competitive solution:
  {"correctness":10,"efficiency":5,"elegance":2,"edge_cases":3,"total":20,"notes":"..."}
```

---

## Preamble (Standard vs Hackathon)

```bash
# Standard (< 500ms)
_B=$(git branch --show-current 2>/dev/null || echo "detached")
_S=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")
_ID=$(date +%s%3N)
_L=$(cat ~/.gstack/projects/$_S/learnings.jsonl 2>/dev/null | tail -5 | jq -sc '.' 2>/dev/null || echo "[]")
_D=$(cat ~/.gstack/projects/$_S/session-deltas.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
_A=$(cat ~/.gstack/projects/$_S/algo-solutions.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
_M=$(cat ~/.gstack/projects/$_S/prompt-mutations.jsonl 2>/dev/null | jq -sc '[.[]|select(.status=="active")]' 2>/dev/null || echo "[]")
_XL=$([ "${EXPLAIN_LEVEL:-}" = "terse" ] && echo 1 || echo 0)
_HM=$([ "${GSTACK_HACKATHON:-0}" = "1" ] && echo 1 || echo 0)
echo "BRANCH:$_B SLUG:$_S SID:$_ID HACKATHON:$_HM TERSE:$_XL"
echo "LEARNINGS:$_L DELTAS:$_D ALGO_SOLUTIONS:$_A MUTATIONS:$_M"
```

```bash
# Hackathon override (< 100ms)
_B=$(git branch --show-current 2>/dev/null || echo "?")
_S=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "?")
_L=$(cat ~/.gstack/projects/$_S/learnings.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
echo "B:$_B S:$_S L:$_L"
```

---

## Error Recovery Runbook

| Error | Detection | Recovery | Escalate if |
|---|---|---|---|
| bash cmd not found | exit 127 | `which <cmd>`, install or skip | cmd critical to skill |
| Read: file not found | empty output | glob alternatives, report | file in scope |
| Schema invalid | JSON parse error | schema template, retry | attempt ≥ 3 |
| Confidence too low | score < 5 | evidence template, retry | attempt ≥ 3 |
| Timeout (AI) | elapsed > budget | partial + TRUNCATED | scope > 50% done |
| Tool disabled | 403 / disabled | report, suggest manual | always |
| Stale ref | grep fails | re-scan repo | P0/P1 finding |
| Circular skill chain | A→B→A | break at 2nd visit | always |
| Circuit open | retry_count > 6 | HALT, report session | always |
| Algo WA | output ≠ expected | trace + fix specific line | attempt ≥ 3 |
| Algo TLE | O(?) too slow | escalate algorithm tier | no better algo exists |
| Algo MLE | space too high | rolling array / compress | attempt ≥ 2 |
| Algo overflow | result wraps | add ll cast + mod | pervasive overflow |

---

## Output Scoring (Both Modes)

```
AI_MODE quality_score:
  ≥ 0.90 → 🟢 Elite  — present immediately
  0.75–0.89 → 🟡 Good  — present with metadata
  0.50–0.74 → 🟠 Weak  — present with [REVIEW] flag
  < 0.50  → 🔴 Bad   — debug first, never present

ALGO_MODE confidence:
  9–10 → 🟢 Submit — high confidence, all edges clean
  7–8  → 🟡 Review — flag uncertain cases, add comments
  5–6  → 🟠 Unsafe — do not submit in competitive context
  < 5  → 🔴 Wrong  — escalate, try fallback algorithm

Token efficiency:
  > 0.002 → Excellent
  0.001–0.002 → Good
  < 0.001 → Expensive → flag for OPTIMIZER
```
