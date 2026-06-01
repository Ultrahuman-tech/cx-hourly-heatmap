# CX Hourly Heatmap

Single-file, dependency-free dashboard for CX agent hourly productivity.

- Heatmap: agents × hours (ordered 7am→7am shift-day), colour-graded, count on each tile.
- Per-agent shift total + hourly totals + summary cards.
- Click a tile → replies/escalated/closed breakdown for that agent×hour.
- Duty (Email/Deferred) comes from the **console queue** field in the data, not a roster.
- Import CSV/JSON (drag-drop), export CSV/JSON.

## Use
Open `index.html` (or the GitHub Pages URL) and drag in a CSV/JSON. Ships with synthetic sample data.

## Data formats
- **Hourly long CSV:** `Agent, Duty, Status, Hour, Replies, Escalated, Closed` (or a `Count` column)
- **Hourly wide CSV:** `Agent, Duty, 0, 1, 2, … 23`
- **Canonical JSON:** `{ "date","metric","fields", "agents":[{"name","duty","status","hours":{"0":n},"breakdown":{"0":{"replies":n,...}}}] }`
