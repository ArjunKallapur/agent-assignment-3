## Grade: 99 / 100

**Assignment:** E-Commerce Supply Chain Manager (n8n)  
**Attempt:** 1 of 2  ·  **Graded:** 2026-07-18  ·  Commit `196a35e`

### Score breakdown
| Criterion | Max | Earned | Notes |
|-----------|-----|--------|-------|
| mp_1 | 8 | 8 | System prompt defines a full output schema (plan_id, reasoning, subgoals[], next_subgoal) and constrains next_subgoal to exactly forecast/inventory/supplier/logistics; the Route on Subgoal Switch routes on {{ $json.next_subgoal }} across all 4 branches with a fallback. Docked 1: full-branch end-to-end demo is only an unverifiable Loom link (analysis.md:61). Demo credit restored: student submitted a working demo video link (a required deliverable); per instructor decision on 2026-07-18, a provided demo earns full credit for this criterion. (`workflows/supply-chain-manager-starter.json (Master Planner Agent node system prompt; Route on Subgoal switch)`) |
| mp_2 | 10 | 10 | Two worked HTN-style decomposition examples embedded in the prompt, each with subgoal ordering and a dependency rationale (inventory depends on forecast; supplier gating). Reflective analysis.md present (1457 words) and discusses planner/next_subgoal handling. (`workflows/supply-chain-manager-starter.json (Master Planner Agent node, 'Worked example 1' / 'Worked example 2'); analysis.md (Reflection On Primer Questions)`) |
| df_1 | 6 | 6 | movingAverage() correctly averages the trailing 3-month window via slice(-3), divides by window.length (handles <3 months), returns 0 on empty series. (`custom-nodes/demand-forecast.js:70-75`) |
| df_2 | 8 | 8 | seasonalIndex() computes mean(units for matching month-of-year) / overall mean with proper guards (empty series, no matching rows, zero overall mean all default to 1.0). Matches the spec definition. (`custom-nodes/demand-forecast.js:95-107`) |
| df_3 | 4 | 4 | Prompt takes the numeric forecast_units in, revises only on contextual signals, bounds revisions (+/-25% weak, +/-50% strong), and emits schema-conformant JSON (sku, revised_forecast, delta_pct, reasoning, context_signals_used). (`workflows/supply-chain-manager-starter.json (Forecast Context Adjuster node system prompt)`) |
| eoq_1 | 8 | 8 | eoq() = round(sqrt(2*D*S/H)) with guards returning 0 when any of D/S/H <= 0. Correct closed form. (`custom-nodes/eoq-optimizer.js:75-81`) |
| eoq_2 | 10 | 9 | detectViolations() catches the viral-spike demand signature (last3 > 2.5x prior9) and the dying/declining signature (last3 < 0.5x first3), plus low_velocity and a volatility-aware long_lead_time check. analysis.md defends each flag. Docked 1: the code keys purely off sales trends and does not itself couple to inventory position (on_hand vs reorder_point); that below/above-reorder-point reasoning lives only in the downstream LLM handler and the write-up, not in the detector. (`custom-nodes/eoq-optimizer.js:113-148; analysis.md (EOQ Assumption Analysis)`) |
| eoq_3 | 4 | 4 | Prompt emits a per-flag exception action (recommended_action reorder/markdown/liquidate/hold, recommended_qty, reasoning) with an explicit policy branch for viral_spike, declining, low_velocity, long_lead_time, and no-flags. (`workflows/supply-chain-manager-starter.json (Inventory Exception Handler node system prompt)`) |
| sp_1 | 14 | 14 | 4-dimension weighted rubric with explicit weights (reliability 35, quality 25, lead_time 25, cost 15), roster normalization, tiering, and structured score_breakdown output. analysis.md defends the weight choices against a stated business context. (`workflows/supply-chain-manager-starter.json (Supplier Performance Monitor node system prompt); analysis.md (Supplier Rubric Defense)`) |
| lg_1 | 6 | 6 | Prompt reasons over hard constraints first (region, deadline, weight capacity, perishability) then soft preferences (cost, transit-time slack, mode risk, relationship continuity), with an explicit tie-break rule. (`workflows/supply-chain-manager-starter.json (Logistics Coordinator (LLM) node system prompt)`) |
| lg_2 | 10 | 10 | pickCheapestFeasible() filters by all 5 hard constraints (origin, destination, transit<=deadline, capacity, perishable), maps total_cost_usd = weight*cost_per_kg, returns null when infeasible, and reduces to the minimum-cost survivor. Correct greedy planner. (`custom-nodes/classical-logistics.js:84-101`) |
| lg_3 | 2 | 2 | Prompt sets use_classical_fallback: true only for purely numeric requests; the IF node routes on JSON.parse($json.content[0].text).use_classical_fallback === true to the classical fallback branch. (`workflows/supply-chain-manager-starter.json (Logistics Coordinator (LLM) node prompt; Use Classical Fallback? IF node)`) |
| fn_1 | 10 | 10 | Aggregates any branch's output into a business-readable key_findings list (branch-specific formatters for forecast/inventory/supplier/logistics), robustly unwraps Anthropic envelopes, and sets next_recommended_subgoal via a subgoal->next map. Docked 1: full credit's end-to-end multi-branch demo is only an unverifiable Loom link (analysis.md:61). Demo credit restored: student submitted a working demo video link (a required deliverable); per instructor decision on 2026-07-18, a provided demo earns full credit for this criterion. (`workflows/supply-chain-manager-starter.json (Final Output code node jsCode)`) |
| Integrity deduction | — | 0 | Provided files unmodified |
| **Total** | **100** | **99** | |

### What went well
- All five classical algorithms are correct and defensively coded: movingAverage/seasonalIndex, eoq (with zero-guards), pickCheapestFeasible (full 5-constraint filter + min-by-cost), and a 4-heuristic detectViolations that catches both the viral-spike and declining demand signatures.
- LLM system prompts are unusually complete and well-specified: the Master Planner ships a strict routable schema plus two worked HTN examples, and the Supplier Monitor encodes an explicit 4-dimension weighted rubric (35/25/25/15) with roster normalization and tiering.
- The reflection (analysis.md, 1457 words) is substantive and honest: it defends the supplier weights against a stated business context, walks the SKU-007/SKU-013 boundary cases, and candidly documents a CSV row-duplication bug and its dedup fix.
- Strong engineering hygiene throughout the workflow: CSV-row dedup via Map, a Parse Plan node with markdown-fence stripping and goal-inference fallback, and a Final Output node that gracefully handles unparsed LLM prose.

### What to improve (actionable)
- detectViolations() decides purely from sales trends; fold the inventory position (on_hand vs reorder_point) into the flag logic so 'viral spike below reorder point' and 'dying SKU far above reorder point' are caught in code rather than only in the downstream LLM handler and write-up.
- The end-to-end demo is a single Loom link; an in-repo artifact (recording, or committed run outputs for >=3 branches) would let the multi-branch requirement for MP-1 and FN-1 be verified directly.
- The forecast run left both sampled SKUs unrevised because context signals only arrive via a free-text notes field; feeding the detected assumption flags (or explicit context signals) into the Forecast Context Adjuster would exercise the LLM-revision path the design intends.
- As noted in analysis.md, the serial file-read chain that caused item multiplication should be replaced with explicit merge/data-loader nodes so the dedup workaround is no longer needed.

### Automated checks
- ✅ All required files implemented
- ✅ Provided files unmodified
- ✅ 0/0 output artifacts committed
- ✅ Reflection 1457 words

### Resubmission
You may resubmit **once**. Push fixes to this repo, then notify the instructor; we'll re-grade as **Attempt 2 (final)**. This is attempt 1 of 2.

---
*Graded automatically with Claude Code against the course rubric. Questions → contact the instructor.*


---
<sub>🔎 **Autograder record** — attempt 1 of 2 · graded at commit `196a35e` · delivered 2026-07-18T20:42:36Z. Commits pushed to `main` after this timestamp are treated as a resubmission.</sub>
