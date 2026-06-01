# Console integration — how this merges into cx-ai-ops

This dashboard is built to drop into the **cx-ai-ops admin console** with no reshaping.
The data path is identical to the console's own, so "pipe in live data" = point it at the
existing endpoint. There are two integration tiers — start with Tier 1, graduate to Tier 2.

## The data contract (the seam)

The dashboard renders from one canonical model:

```ts
type HeatmapData = {
  date: string;
  metric?: string;                 // default "tickets"
  fields?: string[];               // breakdown keys, default ["replies","escalated","closed"]
  agents: {
    name: string;
    email?: string;
    duty: "Email" | "Deferred" | "Both" | "Unknown";   // from console queue, NOT roster
    status?: string;               // online/away/busy
    hours: Record<number, number>;        // hour(0-23) -> count
    breakdown?: Record<number, { replies?: number; escalated?: number; closed?: number }>;
  }[];
};
```

It **also** ingests the console's existing wire shape verbatim — see `fromConsoleEmailLog()`
in `index.html`. The console's `GET /api/v1/admin/email/drafts/log?date=YYYY-MM-DD` (hourly mode)
already returns:

```jsonc
{ "date":"2026-06-01",
  "hourly_buckets":[ { "hour":0, "agents":[
      { "agent_id":"a@x", "display_name":"…", "replies":4, "escalated":0, "closed":1, "total":5 } ] } ],
  "daily_total":{…}, "agent_breakdown":[…] }
```

(see `cx-ai-ops/admin-api/api/admin.py` → `get_email_log`). The adapter maps
`hourly_buckets[].agents[]` → `agents[].hours{hour}` + `breakdown`. **No backend change needed
for the counts.**

### The one backend addition: duty

Duty must come from the **console queue**, not the roster. `cxops_agent_presence` already has
`active_queue` / `assignment_tier`. Add that field to each agent object in the email-log
response (and/or the leaderboard response):

```python
# admin.py get_email_log _agent_list(): include the agent's active_queue
"duty": presence_active_queue.get(agent_id)   # "email" | "deferred" → mapped client-side
```

The dashboard reads `a.duty || a.active_queue || a.queue` and normalises it. Until that field
ships, duty shows "Unknown" (never guessed from a roster).

## Tier 1 — embed now (zero backend work)

Host `index.html` (already on GitHub Pages) and drop it into a console tab as an island that
renders an iframe pointed at the same origin:

```html
<!-- in the console shell, a new tab container -->
<div id="content-heatmap" class="hidden">
  <iframe id="heatmap-frame" style="width:100%;height:80vh;border:0"></iframe>
</div>
<script>
  // when the tab is shown:
  document.getElementById('heatmap-frame').src =
    '/static/heatmap/index.html?source=console&date=' + currentDate;  // same origin → auth cookie travels
</script>
```

`?source=console&date=…` makes the page auto-call `loadFromConsole()` on open, which does
`fetch(origin + '/api/v1/admin/email/drafts/log?date=…', {credentials:'include'})` — the same
base + cookie as `frontend/src/lib/api.ts`. Live data, no rebuild. Override the base if needed
with `window.HEATMAP_API_BASE`.

## Tier 2 — native React island (matches the stack)

Promote it to a first-class mount like `leaderboard`:

1. `frontend/src/features/heatmap/HeatmapTab.tsx` — port the render (the color/grid logic is
   ~120 lines of pure functions; lift them as-is). Fetch via the existing typed client +
   `@tanstack/react-query` (`staleTime: 30_000`, `credentials:'include'` is already the default).
2. `frontend/src/mounts/heatmap.tsx` — copy `mounts/leaderboard.tsx`: expose
   `window.mountHeatmapIsland(el)` / `unmountHeatmapIsland`, auto-mount when `#content-heatmap`
   is visible into `#heatmap-react-root`.
3. `frontend/vite.config.heatmap.ts` — copy `vite.config.leaderboard.ts` (new entry).
4. Add the nav tab + container to the console shell.

Data: reuse `/email-drafts/log` (hourly) → the same `fromConsoleEmailLog` mapping, now in TS.
Add a typed `EmailLogHourly` to `frontend/src/lib/types.ts`.

## Auth & CORS

- Live fetch uses `credentials:'include'` → relies on the admin cookie, so it only works
  **same-origin** (embedded in the console) or with explicit CORS `allow-credentials`.
- The standalone GitHub Pages copy **cannot** reach the console (cross-origin, by design) —
  there it's import-a-file only. That's the safe separation.
