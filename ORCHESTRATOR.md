# Orchestrator v4.0 — Dual-Mode Intelligence Layer

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATOR v4                                │
│                                                                        │
│  INPUT                                                                 │
│    │                                                                   │
│    ▼                                                                   │
│  ┌─────────┐                                                           │
│  │ PLANNER │ ← classifies mode + builds execution plan                │
│  └────┬────┘                                                           │
│       │                                                                │
│   ┌───┴────────────────────────────────────┐                          │
│   │ AI_MODE                                │ ALGO_MODE                 │
│   ▼                                        ▼                          │
│  ┌──────────┐                         ┌───────────┐                   │
│  │ EXECUTOR │                         │ ALGO-CORE │                   │
│  └────┬─────┘                         └─────┬─────┘                   │
│       │                                     │                         │
│       ▼                                     ▼                         │
│  ┌──────────┐   ┌──────────┐         ┌────────────┐                   │
│  │  CRITIC  │   │  DEBUGGER│         │  VERIFIER  │                   │
│  └────┬─────┘   └────┬─────┘         └─────┬──────┘                   │
│       │score<0.75    │retry                 │fail                     │
│       └──────────────┘               ┌─────┴──────┐                   │
│       │                              │ ALT-SOLVER │ ← escalate algo   │
│       ▼                              └────────────┘                   │
│  ┌──────────┐                                                          │
│  │OPTIMIZER │ ← async, end-of-session                                 │
│  └──────────┘                                                          │
│       │                                                                │
│       ▼                                                                │
│  ┌──────────┐                                                          │
│  │ LAM LAYER│ ← executes tool call sequences                          │
│  └──────────┘                                                          │
│                                                                        │
│  Storage: learnings.jsonl | session-deltas.jsonl |                    │
│           prompt-mutations.jsonl | algo-solutions.jsonl               │
└────────────────────────────────────────────────────────────────────────┘
```

**Message envelope** (all inter-module communication):
```typescript
interface OrchestratorMessage {
  from: 'planner'|'executor'|'critic'|'debugger'|'optimizer'|'lam'|'algo_core'|'verifier'
  to:   'planner'|'executor'|'critic'|'debugger'|'optimizer'|'lam'|'algo_core'|'verifier'|'user'
  session_id: string
  ts: string
  type: 'output'|'gate'|'debug_request'|'retry'|'escalate'|'lam_plan'|'algo_result'|'optimize'
  mode: 'AI_MODE' | 'ALGO_MODE'
  payload: unknown
  priority: 1|2|3
  circuit_open?: boolean
}
```

---

## Module 1: PLANNER

**Input:** raw user message + `{BRANCH, SLUG, SID, LEARNINGS, DELTAS, HACKATHON}`
**Output:** `Plan`

```typescript
interface Plan {
  mode: 'AI_MODE' | 'ALGO_MODE'

  // AI_MODE fields
  intent_type?: string
  skill_chain?: string[]
  parallel_groups?: string[][]
  gates?: { after: string; type: 'human'|'auto' }[]
  lam_required?: boolean

  // ALGO_MODE fields
  problem_type?: string          // pattern matched from ALGO_CORE
  algo_id?: string               // ALGO-XX
  n_constraint?: number          // extracted from problem
  time_limit_ms?: number
  complexity_required?: string   // O(?) needed to fit constraints

  // shared
  estimated_tokens: number
  hackathon_mode: boolean
}
```

### Classification Protocol

```
1. Extract signals from message (see CLAUDE.md Dual-Mode Router)

2. If ALGO_MODE:
   a. Extract N, Q, constraints from problem statement
   b. Compute required complexity: complexity_required = f(N, time_limit)
   c. Match problem text to Pattern Recognition Engine (ALGO_CORE.md)
   d. Select algo_id (one primary, one fallback)
   e. Flag: needs_proof=true for competitive context

3. If AI_MODE:
   a. Extract intent_type
   b. Map to skill_chain (CLAUDE.md Skill Registry)
   c. Detect compound tasks → multi-skill chains
   d. Assign parallel_groups (independent skills)
   e. Declare gates
   f. Flag lam_required

4. Token estimate:
   AI_MODE:   sum(budget[skill]) × 1.2
   ALGO_MODE: budget['Algo-Solve'] + budget['Algo-Verify']

5. Emit PLAN block. Pause if GATE: human.
```

### Dynamic Routing (AI vs Algo)

The PLANNER uses this decision tree inline:

```python
def route(message: str, context: dict) -> str:
    ALGO_SIGNALS = [
        r'\d\s*[≤<]=?\s*[nNkK]\s*[≤<]=?\s*10',  # 1 ≤ N ≤ 10^6
        r'(time limit|memory limit)',
        r'(test case|test input)',
        r'\bO\([Nn]',                              # O(N log N) etc
        r'(graph|tree|shortest path|longest|subsequence|subarray)',
        r'(dijkstra|bfs|dfs|dp|greedy|segment tree)',
        r'(codeforces|leetcode|icpc|atcoder|competitive)',
    ]
    if any(re.search(p, message, re.I) for p in ALGO_SIGNALS):
        return 'ALGO_MODE'
    return 'AI_MODE'
```

---

## Module 2: EXECUTOR (AI Mode)

**Input:** `Plan` + `SessionState`
**Output:** `SkillOutput` per skill

```typescript
interface SessionState {
  session_id: string; slug: string; branch: string; mode: 'AI_MODE'|'ALGO_MODE'
  plan: Plan
  skill_outputs: Record<string, SkillOutput>
  cumulative_findings: Finding[]
  total_tokens_used: number; total_retries: number
  user_corrections: number; circuit_open: boolean; elapsed_s: number
}

interface SkillOutput {
  skill: string; verdict: 'pass'|'warn'|'fail'|'needs_input'
  confidence: number; quality_score: number
  findings: Finding[]; next: string[]
  tokens_used: number; duration_s: number
  truncated: boolean; attempt: number
  lam_actions?: LAMAction[]
}
```

### Parallel Micro-Agent Spawn

```typescript
async function runParallelGroup(
  group: string[], ctx: SessionState
): Promise<Record<string, SkillOutput>> {
  const promises = group.map(skill =>
    runSkillWithBudget(skill, ctx, ctx.plan.estimated_tokens / group.length)
  );
  const results = await Promise.allSettled(promises);
  const outputs: Record<string, SkillOutput> = {};
  for (let i = 0; i < group.length; i++) {
    const r = results[i];
    outputs[group[i]] = r.status === 'fulfilled'
      ? r.value
      : { verdict: 'fail', findings: [], error: r.reason, skill: group[i] };
  }
  return outputs;
}

function mergeParallelOutputs(outputs: Record<string, SkillOutput>): SkillOutput {
  const all = Object.values(outputs);
  return {
    skill: 'parallel_merge',
    verdict: worstVerdict(all.map(o => o.verdict)),
    confidence: Math.min(...all.map(o => o.confidence)),
    findings: deduplicateFindings(all.flatMap(o => o.findings)),
    next: [...new Set(all.flatMap(o => o.next))],
    tokens_used: all.reduce((s, o) => s + o.tokens_used, 0),
    quality_score: all.reduce((s, o) => s + o.quality_score, 0) / all.length,
    duration_s: Math.max(...all.map(o => o.duration_s)), // parallel time = max
    truncated: all.some(o => o.truncated),
    attempt: 1,
  };
}
```

---

## Module 3: ALGO-CORE (Algo Mode)

**Input:** `Plan` (ALGO_MODE) + problem statement
**Output:** `AlgoResult`

```typescript
interface AlgoResult {
  algo_id: string
  algorithm_name: string
  complexity_time: string
  complexity_space: string
  fits_constraints: boolean
  pattern_matched: string
  edge_cases_verified: string[]
  overflow_safe: boolean
  solution_code: string
  confidence: number              // 1-10, must be ≥ 9 for competitive submit
  fallback_algo_id?: string       // if primary fails verification
  proof_sketch?: string           // brief correctness argument
}
```

### Solve Protocol

```
1. PATTERN MATCH
   Input: problem statement
   Action: scan ALGO_CORE.md Pattern Recognition Engine
   Output: {pattern, algo_id, complexity_required}

2. COMPLEXITY CHECK (R11)
   Check: O(algo) ≤ complexity_required
   If fail: escalate to next-best algorithm
   If no algorithm fits: flag as research problem, escalate to user

3. TEMPLATE INSTANTIATION
   Load: ALGO-XX template from ALGO_CORE.md
   Fill: problem-specific variable names, array sizes, constraints
   Output: complete solution code

4. EDGE CASE VERIFICATION (R12)
   Generate: 5 edge cases from R12 checklist
   Execute: mental trace on each
   If any fail: fix before proceeding

5. EMIT AlgoResult
   Confidence ≥ 9 or do not emit for competitive context
```

### Fallback Strategy

```
Primary algorithm fails verification:
  → Try next pattern match (second-best algo)
  → If complexity still fails → escalate complexity one tier
    (e.g., O(N log N) not possible → O(N √N) → flag tradeoff)
  → If no solution fits constraints → emit RESEARCH_NEEDED with analysis
```

---

## Module 4: VERIFIER (Algo Mode)

**Input:** `AlgoResult`
**Output:** `VerificationResult`

```typescript
interface VerificationResult {
  passed: boolean
  example_inputs_correct: boolean
  edge_cases_passed: string[]
  edge_cases_failed: string[]
  complexity_verified: boolean
  overflow_checked: boolean
  issues: string[]
  confidence_adjustment: number  // delta applied to AlgoResult.confidence
}
```

### Verification Protocol

```
1. Run algorithm on all provided example inputs → verify output matches expected
2. Run R12 edge case checklist
3. Verify complexity formula matches implementation (e.g., nested loops = O(N²))
4. Check every integer operation for overflow potential
5. If issues found:
   - Minor (off-by-one, wrong index): fix inline, note in issues
   - Major (wrong algorithm, wrong complexity): route to ALT-SOLVER
6. Emit VerificationResult
```

---

## Module 5: CRITIC (AI Mode)

**Scoring rubric:**

| Dimension | Weight | Measurement |
|---|---|---|
| Correctness | 40% | findings with file:line / total |
| Completeness | 25% | scope touched / scope declared |
| Calibration | 20% | Pearson(self-reported confidence, evidence) |
| Schema | 15% | output matches contract (0 or 1) |

```
quality_score = correctness×0.40 + completeness×0.25 + calibration×0.20 + schema×0.15

≥ 0.75 → pass to user
0.50–0.74 → pass with [REVIEW] flag
< 0.50 → route to DEBUGGER
```

**Evidence adjustment:**
```
adjusted_confidence = self_reported
  - 2.0  (no file:line in location)
  - 1.0  (file not actually read via Read tool)
  - 0.5  (no cause→effect in description)
  + 1.0  (historical calibration confirms pattern from learnings)
```

---

## Module 6: DEBUGGER (AI Mode)

| Failure Type | Detection | Remedy |
|---|---|---|
| SCHEMA_INVALID | parse error / missing field | strict_schema + schema template |
| LOW_CONFIDENCE | confidence < 5 or evidence absent | evidence_required + evidence template |
| INCOMPLETE_SCOPE | completeness < 0.7 | scope checklist injection |
| CALIBRATION_DRIFT | calibration < 0.6 | anchor from learnings |
| TIMEOUT | elapsed > budget | split task, reduce scope 50% |
| TOOL_ERROR | non-zero exit / Read fail | diagnose environment |
| STALE_REF | grep fails on location | re-scan repo |

**Retry budget:**
```
attempt 1: base prompt
attempt 2: base + debug template for weak_dimensions[0]
attempt 3: stripped context + only failing dimension + /investigate on failure
attempt 4: ESCALATE (never retry again)
```

**Compact debug templates:**
```
SCHEMA_FIX: "Schema failed. Required:{SCHEMA} Output ONLY '{...}'. No text outside JSON."
EVIDENCE:   "Re-examine:{LOCATIONS} Quote exact lines: 'file:N — text'. No quote = rejected."
SCOPE:      "Unaddressed:{MISSING} Report findings OR state 'no issues: reason' per area."
CALIBRATION:"Baseline:{ANCHOR} Rescale: 9-10=demonstrated, 7-8=strong, 5-6=suspicious, <5=omit."
TIMEOUT:    "Timeout. Address ONLY:{REDUCED} Defer:{DEFERRED}"
```

---

## Module 7: OPTIMIZER (Async)

Reads `session-deltas.jsonl` after session ends.

```typescript
interface SessionDelta {
  session_id: string; ts: string; mode: 'AI_MODE'|'ALGO_MODE'
  skill_chain?: string[]
  algo_ids?: string[]                     // for ALGO_MODE
  quality_scores: Record<string, number>
  token_efficiency: number
  retry_count: number; user_interventions: number
  lam_accuracy: number; learnings_generated: number
  mutations_applied: string[]
  algo_accuracy?: number                  // correct solutions / attempts
  edge_cases_caught_by_verifier?: number
}
```

**Trigger thresholds:**
```
token_efficiency < 0.001            → flag skill as expensive
retry_count ≥ 2 in 3+ sessions     → promote failure to calibration_anchor
user_interventions ≥ 3 same skill  → flag for human review
quality_score < 0.75 consistently  → SP-13 mutation proposal
algo_accuracy < 0.80               → expand algo pattern matching
edge_cases_caught / total > 0.30   → verifier catching too many → fix templates
```

**Prompt mutation lifecycle:**
```
1. Detect degraded skill (3+ sessions, quality < 0.75)
2. SP-13: generate mutation_proposal
3. Write to prompt-mutations.jsonl
4. EXECUTOR applies on next invocation
5. Track quality_delta over 3 sessions
6. ACCEPT if mean_delta > 0.05 | REVERT if mean_delta < 0
7. Log to learnings.jsonl as CONFIRMED_MUTATION
```

---

## Module 8: LAM (Large Action Model)

**Position:** EXECUTOR → LAM → tool_calls → EXECUTOR

```typescript
const ACTION_REGISTRY = {
  'file.read':    { reversible: true,  risk: 'none'     },
  'file.write':   { reversible: true,  risk: 'low'      },
  'file.delete':  { reversible: false, risk: 'high'     },
  'bash.test':    { reversible: true,  risk: 'none',    pattern: /^bun test|npm test|pytest/ },
  'bash.lint':    { reversible: true,  risk: 'none',    pattern: /^(bun|npm) run lint/ },
  'bash.build':   { reversible: true,  risk: 'low',     pattern: /^(bun|npm) run build/ },
  'bash.run':     { reversible: true,  risk: 'low',     pattern: /^(bun|node|python)/ },
  'git.add':      { reversible: true,  risk: 'low'      },
  'git.commit':   { reversible: true,  risk: 'low'      },
  'git.push':     { reversible: false, risk: 'high'     },
  'deploy.prod':  { reversible: false, risk: 'critical' },
} as const;
```

**Gate rule:** `reversible: false` → require explicit human approval before execution.

**LAM Action:**
```typescript
interface LAMAction {
  id: string; type: keyof typeof ACTION_REGISTRY
  tool: string; params: Record<string, unknown>
  reversible: boolean; expected_output: string
  on_failure: 'halt'|'skip'|'retry'
}
```

---

## Parallel Execution Groups

```
AI_MODE:
  Group A (plan reviews, independent): [ceo, design, devex] → merge → eng
  Group B (post-ship, independent):    [canary, retro]
  Group C (audit, independent):        [cso, review] → merge → human gate

ALGO_MODE:
  Multiple solutions in parallel (hackathon):
    [primary_algo, fallback_algo] → pick best by complexity + verification
```

---

## Quality Metrics

| Metric | Floor | Target | Elite |
|---|---|---|---|
| mean quality_score | 0.65 | 0.75 | 0.90 |
| retry rate | 40% | 20% | 5% |
| token efficiency | 0.0005 | 0.001 | 0.002 |
| confidence Pearson | 0.65 | 0.80 | 0.92 |
| user corrections / session | 5 | 2 | 0 |
| learning capture / session | 1 | 3 | 5 |
| LAM accuracy | 0.80 | 0.90 | 0.98 |
| circuit breaker trips | 0 | 0 | 0 |
| algo accuracy (correct / attempt) | 0.70 | 0.85 | 0.99 |
| edge cases verified per solution | 3 | 5 | 5+ custom |
| complexity gate pass rate | 0.90 | 1.00 | 1.00 |
