# Assignment 3 Analysis

## Algorithm Comparison

The forecast branch uses a hybrid pattern: a deterministic moving average multiplied by a seasonal index creates the baseline, and the Forecast Context Adjuster is allowed to revise that number only when contextual evidence justifies it. For SKU-007, Wool Gloves, the classical baseline should be treated cautiously. The recent sales pattern is extreme: the last three months are far above the prior nine-month average, which is exactly the kind of viral spike the simple moving average can understate if the most recent month is still accelerating. I would trust the LLM-revised forecast more for reorder sizing if it explicitly cites the spike and keeps the revision tied to recent observed demand rather than vague optimism.

For a well-behaved seasonal SKU such as SKU-001, Sunglasses, I would generally trust the classical baseline more. Its seasonal pattern is visible in the historical data and the moving-average x seasonal-index method is transparent, cheap, and reproducible. The LLM should make only a small or zero adjustment unless the input contains a real contextual signal such as a promotion, weather anomaly, or supply disruption. This contrast is the point of the hybrid design: arithmetic handles stable patterns, while the LLM handles business context and anomaly interpretation.

In the forecast run, the Forecast Context Adjuster did not revise either SKU because the `notes` field had no explicit external context such as a promotion, weather anomaly, supply disruption, or viral social signal. That is a useful result: the LLM respected the classical baseline instead of inventing a reason to change it. I would still treat SKU-007 cautiously because the inventory branch later detects its viral-spike pattern from sales history.

| SKU | Classical baseline | LLM revised forecast | Decision |
| --- | ---: | ---: | --- |
| SKU-007 | 487 units in the n8n forecast run | 487 units | Keep forecast unchanged in this branch; rely on inventory exception handling for the viral spike |
| SKU-001 | 7 units in the n8n forecast run | 7 units | Prefer classical baseline because no contextual signal was present |

## EOQ Assumption Analysis

The EOQ implementation uses Wilson's formula, `sqrt((2 * D * S) / H)`, but it does not blindly trust the formula. `detectViolations()` flags four assumption failures. `viral_spike` fires when the last-three-month average is more than 2.5x the prior-nine-month average; SKU-007 triggered this because demand is accelerating sharply and on-hand inventory is below the reorder point. In the corrected inventory run, SKU-007 had annual demand of 1,673 units, a classical EOQ of 431 units, and the LLM exception handler recommended reordering 600 units. I agree with the larger order because the viral-spike and long-lead-time flags break the stable-demand assumptions behind EOQ, so service level matters more than minimizing holding cost.

`declining` fires when the last-three-month average is less than half the first-three-month average; SKU-013 triggered this because demand is fading while inventory is far above the reorder point. In the corrected run, SKU-013 had annual demand of 948 units and the LLM recommended markdown with no reorder. That matches the business objective of reducing excess inventory rather than buying more. `low_velocity` catches annual demand below 60 units, where EOQ can produce a precise-looking but operationally weak answer. `long_lead_time` catches lead times above 28 days combined with volatile demand, because a slow replenishment cycle makes the constant-demand assumption riskier.

I intentionally did not implement perishability detection because the assignment data does not include perishable SKU fields in `current_inventory.csv`. In a real implementation, perishability would need a shelf-life field and likely a spoilage-cost model.

One interim inventory run showed correct flags but inflated annual-demand and EOQ quantities because the n8n file-read chain duplicated parsed CSV rows. The workflow was updated to deduplicate CSV rows before calculations and prompts, and the corrected rerun produced the final inventory quantities below.

| SKU | Corrected signal | Classical EOQ | LLM action | Why |
| --- | --- | ---: | --- | --- |
| SKU-007 | `viral_spike`, `long_lead_time` | 431 | Reorder 600 | Current stock is only 18 units against an 80-unit reorder point, and the stable-demand EOQ assumption is broken. |
| SKU-013 | `declining` | 397 | Markdown, reorder 0 | Current stock is 540 units against an 80-unit reorder point, so reordering would worsen overstock while demand is falling. |
| SKU-005 | `long_lead_time` | 149 | Hold, reorder later with buffer | Stock is still well above the reorder point, but the next replenishment should account for lead-time risk. |
| SKU-009 | `long_lead_time` | 160 | Reorder 210 | Lead-time demand can consume most of the cushion before replenishment arrives, so a buffer over EOQ is justified. |

## Supplier Rubric Defense

The supplier rubric assumes Coastal Goods is a mid-sized direct-to-consumer e-commerce company where customer experience and working capital both matter, but missed deliveries and defects are more damaging than slightly shorter payment terms. I weighted reliability at 35 points because late supplier deliveries directly create stockouts and missed customer promises. Quality receives 25 points because defects create returns, support costs, and customer trust damage. Lead time also receives 25 points because shorter replenishment cycles reduce safety stock needs and make the planner more responsive. Cost receives 15 points, using payment terms as the proxy, because better cash timing matters but should not outweigh operational reliability.

In the supplier branch run, the LLM normalized each KPI across the six-supplier roster and applied the rubric. The ranking matched my intuition: strong reliability, low defects, and short lead times beat favorable payment terms alone. The final workflow output preserved the supplier result as `unparsed_text` because the LLM included explanatory prose before its JSON block, but the Supplier Performance Monitor node itself contained a complete scored ranking.

| Rank | Supplier | Score | Why it makes sense |
| --- | --- | ---: | --- |
| Top | SUP-001 GreenLeaf Goods | 78.09 | Best lead time in the roster, strong on-time rate, and low defect rate. Its only major weakness is Net-30 payment terms. |
| Near top | SUP-006 Bavaria Paper Mills | 75.18 | Best reliability and quality scores, but slower lead time than SUP-001 and weak Net-30 payment terms kept it just below the top score. |
| Bottom | SUP-002 Pacific Rim Trading | 21.25 | Net-60 terms were attractive, but worst-in-roster on-time rate and defect rate made it the clear deprioritize candidate. |

## Run Metrics
Costs are calculated using Anthropic's pricing for the Sonnet 4.6 Model of $3 / MTok for input tokens and $15 / MTok for output tokens

| Run | Goal / branch | Tokens | Approx cost | Latency | Notes |
| --- | --- | ---: | ---: | ---: | --- |
| 1 | forecast | ~2k input, ~1k output | $0.02 | 49s | N/A |
| 2 | inventory | ~2k input, ~1k output | $0.02 | 45s | Corrected run used deduped rows; final inventory quantities are reported in the EOQ section. |
| 3 | supplier | ~2k In, ~2.5k out | $0.04 | 1m12s | N/A |

Callout: during debugging, the CSV-read portion of the workflow was changed from parallel fan-out into a serial file-read chain so `Build Context Summary` would not execute before every parsed dataset was available. That solved the missing-upstream-node error, but it also created duplicated n8n items and temporarily inflated prompts and EOQ totals. The final workflow compensates by deduplicating canonical CSV rows inside the code nodes and compacting the planner context before any LLM call. A production version should replace that workaround with explicit merge/data-loader nodes so the data dependency is clear without item multiplication.

## Reflection On Primer Questions

A classical algorithm can produce a wrong answer in the inventory branch when EOQ recommends a tidy reorder quantity for a SKU whose demand is exploding or collapsing; the assumption flags route those cases to the LLM exception handler. The LLM can produce a wrong answer in logistics if it overthinks a purely numeric carrier choice; the `use_classical_fallback` flag lets the deterministic greedy planner handle those cases. If the Master Planner emits an unknown `next_subgoal`, the Switch fallback output would not reach a useful branch, so a production version should validate the vocabulary and fail loudly or route to a repair node. EOQ is wrong for SKU-013 because declining demand and excess stock violate constant-demand and no-stockout assumptions; the `declining` flag prevents a wasteful reorder. Supplier scoring is an LLM job because it requires weighted judgment over multiple KPIs and recommendations, while EOQ is a classical job because it is a closed-form formula with explicit assumptions.

## Demo video
https://www.loom.com/share/ef07f01f5a4b4c5a9d115dee214c992a