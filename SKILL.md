---
name: gstack-elite
description: >
  Dual-mode autonomous engineering agent. AI_MODE: full skill pipeline
  (debug, review, ship, audit, plan, QA, security, benchmark, retro).
  ALGO_MODE: ICPC-level algorithm solver with 25 templates, pattern recognition,
  complexity verification, and edge case enforcement.
  Triggers: any software task OR any algorithmic/competitive programming problem.
  Auto-detects mode. Self-heals, self-improves, and captures learnings across sessions.
compatibility: "Claude Code, claude.ai, Claude Desktop"
license: MIT
version: 4.0.0
---

# gstack-elite — Dual-Mode Autonomous Intelligence

## Activation

This skill activates when:
- User invokes a `/skill-name` command (AI_MODE)
- User task matches a skill trigger in CLAUDE.md Skill Registry (AI_MODE)
- User provides an algorithmic problem with constraints (ALGO_MODE)
- Orchestrator chains this skill from another skill's `next[]`

**Mode is auto-detected.** See CLAUDE.md Dual-Mode Router for routing rules.

---

## Required Reading Order

1. `CLAUDE.md`        — system brain, rules R1–R12, dual-mode router, skill + algo registries
2. `ORCHESTRATOR.md`  — 8-module orchestrator including ALGO-CORE, VERIFIER, LAM
3. `ALGO_CORE.md`     — 25 algorithm templates, pattern engine, complexity matrix, R12 checklist
4. `PIPELINE.md`      — 6-stage pipeline, self-heal, self-improve, competitive mode
5. `SUPER_PROMPTS.md` — 17 compressed prompt templates (SP-01..SP-17)

---

## Preamble (run first, always)

```bash
# Standard (< 500ms)
_B=$(git branch --show-current 2>/dev/null || echo "detached")
_S=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")
_ID=$(date +%s%3N)
_L=$(cat ~/.gstack/projects/$_S/learnings.jsonl 2>/dev/null | tail -5 | jq -sc '.' 2>/dev/null || echo "[]")
_D=$(cat ~/.gstack/projects/$_S/session-deltas.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
_A=$(cat ~/.gstack/projects/$_S/algo-solutions.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
_M=$(cat ~/.gstack/projects/$_S/prompt-mutations.jsonl 2>/dev/null | jq -sc '[.[]|select(.status=="active")]' 2>/dev/null || echo "[]")
_HM=$([ "${GSTACK_HACKATHON:-0}" = "1" ] && echo 1 || echo 0)
_XL=$([ "${EXPLAIN_LEVEL:-}" = "terse" ] && echo 1 || echo 0)
echo "BRANCH:$_B SLUG:$_S SID:$_ID HACKATHON:$_HM TERSE:$_XL"
echo "LEARNINGS:$_L DELTAS:$_D ALGO_SOLUTIONS:$_A MUTATIONS:$_M"
```

```bash
# Hackathon override (< 100ms) — use when GSTACK_HACKATHON=1
_B=$(git branch --show-current 2>/dev/null || echo "?")
_S=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "?")
_L=$(cat ~/.gstack/projects/$_S/learnings.jsonl 2>/dev/null | tail -3 | jq -sc '.' 2>/dev/null || echo "[]")
echo "B:$_B S:$_S L:$_L"
```

---

## Output Contract

**AI_MODE** — every skill invocation:
```json
{
  "skill": "<name>",
  "session": "<SID>",
  "mode": "AI_MODE",
  "duration_s": 0,
  "confidence": 0,
  "verdict": "pass|warn|fail|needs_input",
  "quality_score": 0.0,
  "findings": [],
  "next": [],
  "truncated": false,
  "attempt": 1
}
```

**ALGO_MODE** — every algorithm solution:
```json
{
  "mode": "ALGO_MODE",
  "session": "<SID>",
  "algorithm": "ALGO-XX",
  "algorithm_name": "",
  "complexity_time": "O(?)",
  "complexity_space": "O(?)",
  "fits_constraints": true,
  "pattern_matched": "",
  "edge_cases": [{"case": "", "result": "pass|fail"}],
  "overflow_safe": true,
  "solution": "<complete code>",
  "confidence": 0,
  "scoring": {
    "correctness": 0,
    "efficiency": 0,
    "elegance": 0,
    "edge_cases": 0,
    "total": 0
  }
}
```

Deviation from either schema → route to DEBUGGER, not to user.

---

## Quick Reference: Mode Detection

```
Has constraint like "1 ≤ N ≤ 10^6"?         → ALGO_MODE
Has "time limit" / "memory limit"?           → ALGO_MODE
Has "implement algorithm for"?               → ALGO_MODE
Has graph/tree/DP/greedy problem structure?  → ALGO_MODE
Has codebase context (file paths, diffs)?    → AI_MODE
Has "/skill-name" command?                   → AI_MODE
Ambiguous → AI_MODE (safer default)
```

---

## Quick Reference: Complexity Gate (R11)

```
N ≤ 10³   → O(N²) ok, O(N³) ok for small
N ≤ 10⁴   → O(N²) ok
N ≤ 10⁵   → O(N log N) required
N ≤ 10⁶   → O(N) or O(N log N) only
N ≤ 10⁹   → O(log N) or O(√N) only
Violation → select better algorithm, never emit suboptimal solution
```

---

## Quick Reference: Algorithm Selection

```
Shortest path (weighted)    → Dijkstra (ALGO-01)
Shortest path (unweighted)  → BFS (ALGO-02)
All-pairs shortest path     → Floyd-Warshall (ALGO-03)
Minimum spanning tree       → Kruskal + DSU (ALGO-04)
Knapsack / subset sum       → 0-1 DP (ALGO-05)
LCS                         → DP O(N×M) (ALGO-06)
LIS                         → DP + BinSearch O(N log N) (ALGO-07)
Interval scheduling         → Greedy by end time (ALGO-08)
String matching             → KMP O(N+M) (ALGO-09)
Max flow                    → Dinic's O(V²E) (ALGO-12)
Bipartite matching          → Hopcroft-Karp (ALGO-13)
Range queries               → Seg Tree (ALGO-15) or BIT (ALGO-17)
Tree path queries            → HLD (ALGO-18)
Combinatorics/modular math  → Math templates (ALGO-21)
Convex hull                 → Andrew's monotone (ALGO-23)
```
