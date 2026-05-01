# Architecture v4.0 — gstack Elite + ICPC

## System Map

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        gstack-elite v4.0                                 │
│                                                                          │
│  ENTRY POINT                                                             │
│  ───────────                                                             │
│  SKILL.md ← Claude reads this first (installable skill entrypoint)      │
│      │                                                                   │
│      ▼                                                                   │
│  CLAUDE.md ← system brain (rules, dual-mode router, registry, R1–R12)  │
│      │                                                                   │
│      ├─────────────── AI_MODE ─────────────────────────────────┐        │
│      │                                                          │        │
│      └─────────────── ALGO_MODE ───────────────────────────────┼───┐    │
│                                                                 │   │    │
│  ┌──────────────────────────────────────────────────────────┐   │   │    │
│  │                   ORCHESTRATOR v4                        │   │   │    │
│  │                                                          │   │   │    │
│  │  PLANNER ──► EXECUTOR ──► CRITIC ──► OPTIMIZER          │   │   │    │
│  │               │               │ <0.75    │ async         │   │   │    │
│  │               │           DEBUGGER       │               │   │   │    │
│  │               │               │ retry    │               │   │   │    │
│  │               └───────────────┘          │               │   │   │    │
│  │               │                          │               │   │   │    │
│  │            LAM LAYER                     │               │   │   │    │
│  │         (plan→actions→tools)             │               │   │   │    │
│  └──────────────────────────────────────────┘   ◄───────────┘   │    │   │
│                                                                   │    │   │
│  ┌──────────────────────────────────────────────────────────┐    │    │   │
│  │                    ALGO-CORE v4                          │◄───┘    │   │
│  │                                                          │         │   │
│  │  Pattern Recognition Engine                             │         │   │
│  │  ── 7 problem categories                                │         │   │
│  │  ── 25 algorithm templates (ALGO-01..25)                │         │   │
│  │  ── Complexity Verification Matrix                      │         │   │
│  │  ── Edge Case Checklist (R12)                           │         │   │
│  │  ── Competitive Boilerplate                             │         │   │
│  │  ── Auto-Solve Protocol                                 │         │   │
│  │                                                          │         │   │
│  │  VERIFIER ── checks solution against examples + edges  │◄────────┘   │
│  │  ALT-SOLVER ── fallback when primary algo fails        │             │
│  └──────────────────────────────────────────────────────────┘             │
│                                                                            │
│  SUPER-PROMPTS v4 (17 prompts)                                            │
│  SP-01..SP-12  (inherited, optimized)                                    │
│  SP-13 Mutate  SP-14 LAM   SP-15 AlgoSolve✦  SP-16 Complexity✦          │
│  SP-17 Compete✦                               (✦ = new in v4)           │
│                                                                          │
│  PIPELINE v4 (dual-mode)                                                 │
│  Stage 0: Context Load                                                   │
│  Stage 1: Decomposition + Mode Routing                                   │
│  Stage 2a/2b: Prompt Assembly | Algo Prep                                │
│  Stage 3a/3b/3c: Execution | Algo Solve | LAM                           │
│  Stage 4: Validation → 4b AI Debug | 4c Algo Debug                     │
│  Stage 5: Iteration                                                      │
│  Stage 6: Output + Learning                                              │
│                                                                          │
│  STORAGE (per-project: ~/.gstack/projects/{SLUG}/)                      │
│  ├── learnings.jsonl         SP-09 synthesizes after each session       │
│  ├── session-deltas.jsonl    OPTIMIZER reads for trend analysis         │
│  ├── optimizer-report.jsonl  OPTIMIZER writes recommendations           │
│  ├── prompt-mutations.jsonl  SP-13 generates; EXECUTOR applies         │
│  ├── low-confidence.jsonl    findings < confidence 5                    │
│  └── algo-solutions.jsonl   ✦ solved problems for recall               │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Gap Resolution Matrix — v3 → v4

| v3 Gap | Severity | v4 Solution | File |
|---|---|---|---|
| **No algorithmic intelligence** | P0 | ALGO_CORE.md: 25 templates, pattern engine, complexity gate | ALGO_CORE.md |
| **No competitive problem solving** | P0 | SP-15 (Algo Solver), SP-16 (Complexity), SP-17 (Scorer) | SUPER_PROMPTS.md |
| **No dual-mode routing** | P0 | Mode router in CLAUDE.md + PLANNER + PIPELINE Stage 1 | CLAUDE.md, ORCHESTRATOR.md |
| **No complexity gate** | P0 | R11: O(?) check before every solution, reroute if fails | CLAUDE.md |
| **No edge case enforcement** | P0 | R12: mandatory 5-case checklist per algo solution | CLAUDE.md, ALGO_CORE.md |
| **No overflow protection** | P1 | R12 item + SP-15/16 overflow scan + `ll` in boilerplate | ALGO_CORE.md |
| **No algo debug loop** | P1 | Stage 4c: WA/TLE/MLE/OVF/EDGE specific handlers | PIPELINE.md |
| **No algo solution memory** | P1 | algo-solutions.jsonl + Stage 0 load + PLANNER cache | PIPELINE.md |
| **No competitive scoring** | P2 | SP-17 + competitive scoring in SP-15 output | SUPER_PROMPTS.md |
| **No alt-solver fallback** | P1 | ALT-SOLVER module + fallback_algo_id in AlgoResult | ORCHESTRATOR.md |
| **No pattern recall from history** | P2 | algo-solutions.jsonl injected into SP-15 as priors | PIPELINE.md |
| **Verifier not defined** | P1 | VERIFIER module with full verification protocol | ORCHESTRATOR.md |

---

## Component Dependency Graph

```
SKILL.md
  └─ CLAUDE.md
       ├─ ORCHESTRATOR.md
       │    ├─ PLANNER        (standalone, reads CLAUDE.md dual-mode router)
       │    ├─ EXECUTOR       (AI_MODE, requires PLANNER output)
       │    │    └─ LAM       (requires EXECUTOR context + ACTION_REGISTRY)
       │    ├─ ALGO-CORE      (ALGO_MODE, requires PLANNER pattern match)
       │    │    └─ VERIFIER  (requires ALGO-CORE AlgoResult)
       │    │    └─ ALT-SOLVER(requires VERIFIER failure)
       │    ├─ CRITIC         (AI_MODE, requires EXECUTOR SkillOutput)
       │    ├─ DEBUGGER       (AI+ALGO, requires CRITIC/VERIFIER failure)
       │    └─ OPTIMIZER      (async, requires session-deltas.jsonl)
       │         └─ SP-13     (requires OPTIMIZER analysis)
       ├─ PIPELINE.md         (requires ORCHESTRATOR)
       │    └─ SUPER_PROMPTS.md
       │         ├─ SP-01..12 (base)
       │         ├─ SP-13..14 (v3: mutation, LAM)
       │         └─ SP-15..17 ✦ (v4: algo solver, complexity, competitive)
       └─ ALGO_CORE.md        ✦ (v4: new module, required by ALGO-CORE orchestrator module)
```

---

## Data Flow (AI Mode)

```
User input
  → PLANNER: intent=debug → skill_chain=[/investigate, /review]
  → EXECUTOR: run /investigate (parallel group A if applicable)
    → SP-02 filled with {ERROR, STACK, FILES, LEARNINGS}
    → LLM call → SkillOutput
  → CRITIC: quality_score=0.82 ✓
  → EXECUTOR: run /review (with investigate output as context)
    → SP-03 filled with {DIFF, LEARNINGS}
    → LLM call → SkillOutput
  → CRITIC: quality_score=0.58 ✗
  → DEBUGGER: weak_dim=evidence → inject SP evidence template
  → EXECUTOR: retry /review (attempt 2)
  → CRITIC: quality_score=0.79 ✓
  → [GATE: human] → user approves
  → STAGE 6: merge → SP-09 → learnings → delta → optimizer
```

## Data Flow (Algo Mode)

```
User input: "Find shortest path in weighted graph, N=10^5 nodes"
  → PLANNER: ALGO_MODE detected
    complexity_required = O(N log N)  [from N=10^5, time=1s]
    pattern = "shortest path, weighted" → ALGO-01 (Dijkstra)
    complexity_check: O((V+E) log V) ≤ O(N log N) ✓
  → ALGO-CORE:
    → Load ALGO-01 template
    → Instantiate with problem variables
    → R12: 5 edge cases (single node, disconnected, all same weight, max N, neg-weight absent)
    → Overflow: distances use ll ✓
    → AlgoResult {confidence: 9, solution: <code>}
  → VERIFIER:
    → Trace example inputs → correct ✓
    → Complexity verify: O((V+E) log V) ✓
    → Overflow: ll confirmed ✓
    → VerificationResult {passed: true}
  → SP-17: score → {correctness:10, efficiency:8, elegance:7, edge_coverage:10, total:35/40}
  → STAGE 6: append algo-solutions.jsonl → SP-09 → learnings → delta
```

---

## Token Budget Model

```
Standard session — AI_MODE (4 skills, no retries):
  PLANNER:   ~300 tokens
  EXECUTOR:  ~2000 × 4 skills = ~8000
  CRITIC:    ~200 × 4 = ~800
  OUTPUT:    ~500
  Total:     ~9600 tokens

With 20% retry rate:
  +1600 tokens → ~11200 total

Standard session — ALGO_MODE (1 problem):
  PLANNER:   ~200 tokens
  ALGO-CORE (SP-15): ~1500
  VERIFIER (SP-16):  ~500
  SCORER (SP-17):    ~300
  Total:     ~2500 tokens  ← 4× cheaper than AI_MODE

Hackathon (parallel AI + 3 algo problems):
  AI pipeline (parallel):  ~6000 tokens
  3 × algo pipeline:        ~7500 tokens
  Total:                   ~13500 tokens / 4h session

Token efficiency target:
  quality_score / total_tokens ≥ 0.001
  Algo mode: 0.9 quality / 2500 tokens = 0.00036 (better than AI_MODE)
  Terse mode: reduces AI by ~40% → better efficiency
```

---

## Competitive Mode Time Budget

```
Problem difficulty mapping:
  Easy   (N ≤ 1000):    5 min (direct template, minimal verification)
  Medium (N ≤ 10^5):   10 min (careful template, full R12)
  Hard   (N ≤ 10^6):   20 min (may need composition of 2 templates)
  Expert (open-ended): 30 min (research + full proof)

Hackathon competitive (30s):
  00-05s: constraints → complexity → pattern match
  05-10s: template select + complexity gate
  10-20s: instantiate + critical edge cases only (3, not 5)
  20-25s: overflow check
  25-30s: emit with confidence ≥ 8 (relaxed from 9 in hackathon)
```

---

## Deployment

```bash
# Install
cp -r gstack-elite-v4 ~/.claude/skills/gstack-elite/

# Initialize
mkdir -p ~/.gstack/projects/$(basename $(git rev-parse --show-toplevel))

# AI mode
claude -p "$(cat ~/.claude/skills/gstack-elite/SKILL.md)" --task "review my auth module"

# Algo mode (auto-detected)
claude -p "$(cat ~/.claude/skills/gstack-elite/SKILL.md)" \
  --task "Given a weighted graph with N≤10^5 nodes, find shortest path from 1 to N"

# Hackathon
export GSTACK_HACKATHON=1
gstack-orchestrate "build auth, test, ship"

# Competitive (algo batch)
gstack-orchestrate --mode algo --problems problems.txt

# Score last session
gstack-score --session last

# Analytics
gstack-analytics --slug $(basename $(git rev-parse --show-toplevel))

# Check mutations
cat ~/.gstack/projects/*/prompt-mutations.jsonl | jq 'select(.status=="active")'

# Check algo solutions cache
cat ~/.gstack/projects/*/algo-solutions.jsonl | jq '{pattern, algo_id, confidence}'
```

---

## Quality Targets

| Metric | Floor | Target | Elite |
|---|---|---|---|
| AI quality_score | 0.65 | 0.75 | 0.90 |
| AI retry rate | 40% | 20% | 5% |
| AI token efficiency | 0.0005 | 0.001 | 0.002 |
| AI confidence Pearson | 0.65 | 0.80 | 0.92 |
| AI user corrections | 5/session | 2/session | 0/session |
| AI learning capture | 1/session | 3/session | 5/session |
| LAM accuracy | 0.80 | 0.90 | 0.98 |
| Circuit breaker trips | 0 | 0 | 0 |
| Algo accuracy | 0.70 | 0.85 | 0.99 |
| Algo edge cases / solution | 3 | 5 | 5+ custom |
| Complexity gate pass rate | 0.90 | 1.00 | 1.00 |
| Competitive score / problem | 25/40 | 32/40 | 38/40 |
| Mutation acceptance rate | — | 50% | 80% |
