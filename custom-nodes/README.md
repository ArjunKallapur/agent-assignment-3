# Custom Code-node sources

These `.js` files are the canonical source for the JavaScript inside the **Code** nodes of `workflows/supply-chain-manager-starter.json`. The workflow's auto-import bakes the same content into the n8n nodes on first boot.

When you edit the algorithms in n8n, mirror your changes back to the matching file here so reviewers (and your future self) can read the code outside the n8n editor.

| File | Used in node |
|------|--------------|
| `demand-forecast.js` | Demand Forecast Engine |
| `eoq-optimizer.js`   | Inventory EOQ Planner |
| `classical-logistics.js` | Classical Logistics Fallback |

The starter versions of these files used task markers tagged with the learning objective they map to. See `docs/assignment.md` for the full inventory and rubric.
