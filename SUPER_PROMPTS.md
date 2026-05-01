# Super-Prompts v4.0 — gstack Elite + ICPC

> Ultra-compressed. Schema-enforced. Mutation-ready. Zero token waste.
> Fill `{VARS}` → fire. Never stack > 3. Schema is part of prompt.
> New in v4: SP-15 (Algo Solver), SP-16 (Complexity Verifier), SP-17 (Competitive Scorer)

---

## SP-01: Universal Skill Activator
**Cost: ~90 tokens**

```
Role: gstack/{SKILL} principal engineer.
Ctx: branch={BRANCH} slug={SLUG} sid={SID}
Priors: {LEARNINGS_JSON}
Task: {TASK}
Budget: {BUDGET} tokens

Rules: cite file:line per finding | confidence 1-10 | output ONLY JSON | no prose outside JSON

{"skill":"{SKILL}","session":"{SID}","verdict":"pass|warn|fail","confidence":0,"quality_score":0.0,
"findings":[{"severity":"P0|P1|P2|P3","confidence":0,"location":"file:N","description":"","evidence":"file:N — exact line","cause_effect":"X → Y"}],
"next":[],"truncated":false}
```

---

## SP-02: Debug / Root Cause
**Cost: ~110 tokens**

```
Iron Law: no fix without root cause.

Error: {ERROR}
Stack: {STACK}
Ctx: {FILES_SNIPPET}
Priors: {LEARNINGS_JSON}

Protocol:
1. INVESTIGATE: read 3 most likely files. Quote exact lines.
2. ANALYZE: what invariant is violated?
3. HYPOTHESIZE: one root cause. Confidence N/10. Evidence.
4. FIX: minimal change. No collateral. Mental test.

Reject hypothesis if confidence < 7. Re-investigate.

{"root_cause":"","evidence":["file:N — quote"],"fix":{"file":"","change":""},"confidence":0,"test_command":"","side_effects":[]}
```

---

## SP-03: Code Review Gate
**Cost: ~95 tokens**

```
Review: {DIFF_OR_FILES}
Priors: {LEARNINGS_JSON}

Check order (stop at first P0):
1. SQL injection / parameterized queries
2. LLM trust boundary violations
3. Side effects inside conditionals
4. Unhandled errors on critical paths
5. Schema changes without migration
6. Secrets or credentials in code
7. Race conditions / missing locks

Emit findings confidence ≥ 6 only.
Format: [SEV] conf:N file:N — description | evidence: file:N — quote

Final line MUST be:
SHIP: {reason}     ← zero P0/P1
NO-SHIP: {reason}  ← any P0/P1
```

---

## SP-04: Architecture Planner (CEO)
**Cost: ~80 tokens**

```
Evaluate: {PLAN}

Answer exactly — no prose, no hedging:
SIGNAL: {1 sentence: strongest insight}
BETS:   {≤3 assumptions that must be true}
RISKS:  {≤3 killers, ranked probability×impact}
SCOPE:  lake (boilable) | ocean (unbounded) + 1 reason
VERDICT: BACK IT | SHRINK IT | KILL IT

{"signal":"","bets":[],"risks":[],"scope":"lake|ocean","scope_reason":"","verdict":"BACK IT|SHRINK IT|KILL IT"}
```

---

## SP-05: Security Audit
**Cost: ~110 tokens**

```
Audit: {CODEBASE_SUMMARY}
Components: {COMPONENTS}
Priors: {LEARNINGS_JSON}

Scan OWASP Top 10 in priority order.
Per finding: file:line | attack vector (1 line) | CVSS Low/Med/High/Crit | minimal fix | confidence N/10
STRIDE per component: [MITIGATED] or [EXPOSED]
Omit confidence < 7. Sort by CVSS descending.

[{"location":"file:N","owasp":"A0X","vector":"","cvss":"Low|Med|High|Critical","fix":"","confidence":0,
"stride":{"S":false,"T":false,"R":false,"I":false,"D":false,"E":false}}]
```

---

## SP-06: QA / Dogfooding
**Cost: ~90 tokens**

```
Target: {URL}
Flows: {FLOWS}
Priors: {LEARNINGS_JSON}

Per flow: navigate → interact → assert
On failure: screenshot + console errors + repro steps

{"bugs":[{"severity":"P0|P1|P2|P3","flow":"","description":"","evidence":"","repro":""}],
"flows_passed":0,"flows_total":0,"verdict":"PASS|PARTIAL|FAIL"}
```

---

## SP-07: Performance Benchmark
**Cost: ~75 tokens**

```
Target: {TARGET}
Baseline: {BASELINE}
Runs: {N} | Threshold: +{THRESHOLD}%

Measure p50/p95/p99/error_rate. Compare vs baseline.
If regression: identify slow path (file:function).

{"p50":0,"p95":0,"p99":0,"error_rate":0,"regression":false,"delta_pct":0,
"verdict":"pass|warn|fail","bottleneck":"file:fn — description"}
```

---

## SP-08: Auto-Ship Checklist
**Cost: ~90 tokens**

```
Branch: {BRANCH} | Base: {BASE}

Run order, halt on BLOCK:
□ bun test → 0 failures
□ No P0/P1 in last /review
□ git diff {BASE}..HEAD | grep -iE 'api_key|secret|password|token' → 0
□ No destructive migrations without rollback
□ New behavior behind feature flag?
□ Post-deploy monitoring configured?

Per item: ✓ PASS | ✗ BLOCK:reason | ⚠ WARN:reason

{"checks":{"tests":"PASS|BLOCK|WARN","security":"PASS|BLOCK|WARN","migrations":"PASS|BLOCK|WARN",
"flags":"PASS|BLOCK|WARN","monitoring":"PASS|BLOCK|WARN"},"verdict":"SHIP|HOLD","blockers":[]}
```

---

## SP-09: Learning Synthesizer
**Cost: ~70 tokens**

```
Findings: {FINDINGS_JSON}
Priors:   {PRIOR_JSON}
Retries:  {RETRY_COUNT}
Outcomes: {OUTCOMES_JSON}
Mode:     {AI_MODE|ALGO_MODE}

NEW_PATTERNS:   findings not in priors → add
CONFIRMED:      findings matching priors → boost confidence
REFUTED:        priors contradicted → mark stale
CALIBRATION:    did confidence predict outcomes? → delta

{"new_learnings":[{"pattern":"","skill_or_algo":"","confidence_adjustment":0}],
"confirmed":[],"refuted":[],"calibration_delta":0.0}
```

---

## SP-10: Task Decomposer
**Cost: ~75 tokens**

```
Task: {TASK}
Time: {MINUTES} min | Stack: {TECH_STACK}
Priors: {LEARNINGS_JSON}

Map to gstack skill chain OR algo template.
Mode: AI_MODE | ALGO_MODE (per router rules in CLAUDE.md)
Mark: PARALLEL | SEQUENTIAL per skill.
Flag ONE-WAY → gate: human. Flag lam_required.
Drop lowest-priority if total > budget; note omissions.

{"mode":"AI_MODE|ALGO_MODE",
"chain":[{"skill_or_algo":"","mode":"parallel|sequential","budget_min":0,"gate":"human|auto","lam_required":false}],
"total_min":0,"critical_path":"","dropped":[]}
```

---

## SP-11: Critic Self-Evaluation
**Cost: ~60 tokens**

```
Evaluate before showing to user:
{OUTPUT}

Score 0-10:
1. Evidence: file:line citations real + specific?
2. Calibration: confidence matches evidence strength?
3. Schema: output matches contract exactly?
4. Completeness: all scope areas addressed?

Any score < 6 → DO NOT present. State failing dimension + fix required.
All ≥ 6 → present with scores as metadata.

{"scores":{"evidence":0,"calibration":0,"schema":0,"completeness":0},"passed":false,"fix_required":"","overall":0.0}
```

---

## SP-12: Failure Recovery
**Cost: ~65 tokens**

```
Skill/Algo: {NAME}. Error: {ERROR}. Attempt: {N}/3.
Mode: {AI_MODE|ALGO_MODE}

Diagnose:
1. Tool error → fix environment first
2. Schema error → strict_schema + schema template
3. Timeout → split smaller unit
4. Context error → reload preamble + re-inject state
5. LAM error → check registry + approval
6. Algo WA → trace failing case + fix specific line
7. Algo TLE → escalate to next complexity tier
8. Algo overflow → add ll cast + intermediate mod

Recovery: {ONE SPECIFIC FIX}
Delta from last attempt: {WHAT CHANGED}

If attempt ≥ 3: ESCALATE with evidence. Never retry.

{"diagnosis":"tool|schema|timeout|context|lam|algo_wa|algo_tle|algo_overflow","recovery_action":"",
"modified_prompt_delta":"","escalate":false,"evidence":[]}
```

---

## SP-13: Prompt Mutation Generator
**Cost: ~80 tokens**

```
Skill {SKILL} degraded over {N} sessions.
Mean quality: {MEAN} (target ≥ 0.75)
Failures: {FAILURE_MODES}
Current prompt: {PROMPT}
Retry patterns: {PATTERNS}

Generate ONE mutation hypothesis:
1. Weakest structural element in current prompt
2. ONE targeted change (not rewrite)
3. Which failure mode it addresses
4. Success: quality_delta ≥ +0.05 over 3 sessions
5. Revert: quality_delta < 0 over 3 sessions

{"hypothesis":"","weak_element":"","change_type":"add_constraint|add_example|reorder|compress|expand_schema",
"diff":"","targets_failure":"","success_condition":"quality_delta >= 0.05","revert_condition":"quality_delta < 0"}
```

---

## SP-14: LAM Plan Generator
**Cost: ~85 tokens**

```
Skill: {SKILL} | Goal: {GOAL}
Files: {RELEVANT_FILES}
Registry: {REGISTRY} | Session approved: {SESSION_APPROVED}

Minimum-action LAM plan:
- Prefer reversible over irreversible
- Every action: expected_output = success condition
- on_failure='halt' for blockers | 'skip' for optional
- Max 10 actions before validation checkpoint
- Production deploys → gate: human always

{"goal":"","actions":[{"id":"a1","type":"","tool":"","params":{},"reversible":true,
"expected_output":"","on_failure":"halt|skip|retry","risk":"none|low|high|critical"}],
"requires_approval":false,"estimated_side_effects":[]}
```

---

## SP-15: Algorithm Solver — NEW
**Cost: ~100 tokens**

```
Problem:
{PROBLEM_STATEMENT}

Constraints: N={N}, Q={Q}, time={TIME_LIMIT_S}s, memory={MEM_MB}MB
Priors (similar solved): {ALGO_SOLUTIONS_JSON}

Protocol (MUST follow order):
1. COMPLEXITY REQUIRED: O(?) needed → compute from N + time limit
2. PATTERN: match to ALGO_CORE registry. State: pattern → algo_id
3. COMPLEXITY CHECK: does algo fit? If not → select next algorithm
4. INSTANTIATE: fill template with problem variables
5. EDGE CASES (R12): trace 5 cases mentally. State each pass/fail.
6. OVERFLOW: is ll needed? Is % MOD needed at every step?

Confidence MUST be ≥ 9 to emit solution. If < 9, re-examine.

{"algorithm":"ALGO-XX","algorithm_name":"","complexity_time":"O(?)","complexity_space":"O(?)",
"fits_constraints":true,"pattern_matched":"","n_constraint":0,
"edge_cases":[{"case":"","result":"pass|fail"}],"overflow_safe":true,
"solution":"<complete code>","confidence":0,
"scoring":{"correctness":0,"efficiency":0,"elegance":0,"edge_cases":0,"total":0}}
```

---

## SP-16: Complexity Verifier — NEW
**Cost: ~55 tokens**

```
Verify solution complexity:
{SOLUTION_CODE}

Problem N: {N} | Time limit: {TIME_S}s

1. Identify all loops. State: for each loop → O(?)
2. Identify all recursive calls. State: recurrence → O(?)
3. Identify all data structure operations. State each → O(?)
4. Compose total: worst-case time O(?) + space O(?)
5. Compare against required: O(?) needed for N={N}
6. Overflow scan: any int that could exceed 2^31? → flag

PASS if fits constraints. FAIL with specific fix if not.

{"time_complexity":"O(?)","space_complexity":"O(?)","fits":true,"overflow_risk":false,
"loops":[{"location":"line:N","complexity":"O(?)"}],"verdict":"PASS|FAIL","fix":""}
```

---

## SP-17: Competitive Scorer — NEW
**Cost: ~50 tokens**

```
Score this competitive solution:
{SOLUTION_CODE}
Problem: {PROBLEM_SUMMARY}
N: {N} | Time: {TIME_S}s | Memory: {MEM_MB}MB

Score on 4 dimensions (0-10 each):
1. Correctness: does it produce correct output for all cases?
2. Efficiency: how much headroom vs constraint? (beats by 10× = 10)
3. Elegance: minimal code, clear naming, no redundant variables
4. Edge coverage: how many edge cases explicitly handled?

Deductions:
  -3 per missing edge case (from R12 checklist)
  -2 per unnecessary variable or loop
  -5 if overflow risk present
  -5 if wrong complexity tier

{"correctness":0,"efficiency":0,"elegance":0,"edge_coverage":0,
"deductions":[],"total":0,"verdict":"SUBMIT|REVIEW|REWRITE","notes":""}
```

---

## SP Efficiency Index v4

| Prompt | Tokens | Signal | Primary Use |
|---|---|---|---|
| SP-11 | ~60 | HIGH | always (critic gate) |
| SP-09 | ~70 | HIGH | always (learning) |
| SP-12 | ~65 | HIGH | on failure |
| SP-17 | ~50 | HIGH | competitive scoring |
| SP-16 | ~55 | HIGH | complexity verification |
| SP-04 | ~80 | HIGH | planning |
| SP-07 | ~75 | HIGH | benchmarks |
| SP-10 | ~75 | HIGH | decomposition |
| SP-13 | ~80 | HIGH | mutation |
| SP-14 | ~85 | HIGH | LAM |
| SP-15 | ~100 | HIGH | algo solve (most valuable) |
| SP-03 | ~95 | MED | code review |
| SP-08 | ~90 | MED | pre-ship |
| SP-06 | ~90 | MED | QA |
| SP-01 | ~90 | MED | universal |
| SP-02 | ~110 | MED | debug |
| SP-05 | ~110 | MED | security |

---

## Composition Rules

1. **Never stack > 3 super-prompts** in one message
2. **Fill all {VARS} before firing** — unfilled vars waste tokens
3. **Schema is always part of the prompt**, not separate instructions
4. **Terse evidence** — `file:N — exact quote` > paragraph
5. **Gate statements** — every prompt touching state ends with binary verdict
6. **Parallel where possible** — SP-04 + SP-05 + SP-03 fire simultaneously on independent scope
7. **Mutations** — if prompt fails 3+ sessions, invoke SP-13. Never manually edit.
8. **Algo mode** — SP-15 always precedes SP-16. SP-17 is final scoring only.
9. **Competitive** — SP-15 → SP-16 → SP-17 is the canonical competitive pipeline.
10. **Confidence gate** — SP-15 requires confidence ≥ 9 before emitting solution.
