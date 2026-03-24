# OpenClaw Skill Utilization Heatmap

Single-file, client-side dashboard for visualizing AgentSkill usage from OpenClaw session logs.

## Files
- `index.html` — Dashboard UI (inline CSS/JS, no backend)
- `sample-manifest.json` — Example skill manifest format

## How to use
1. Open `index.html` in a browser (double-click or open via `file://`).
2. In **Skill Manifest**, drop/select a JSON file with this shape:
   - `{ "skills": [{ "name", "category", "install_date" }] }`
3. In **Session Logs**, drop/select one or more JSONL session files.
4. Review:
   - Summary metrics (total, active, stale, dormant, never used)
   - Heatmap cells by recency
   - Tooltip details on hover (invocations, last used, install date, estimated time saved)
   - ROI section with total time saved and top 5 skills

## Color legend
- **Green**: used in the last 7 days
- **Yellow**: used 7–30 days ago
- **Red**: used 30+ days ago
- **Gray**: never used

## Invocation matching
The parser extracts skill usage by scanning JSONL lines for:
- skill directory patterns ending in `SKILL.md` (e.g., `.../<skill-name>/SKILL.md`)
- direct skill-name mentions in tool call content

## ROI formula
- Per-skill estimated time saved: `invocations × 15 minutes`
- Total saved time: sum across all skills

## Notes
- Entirely client-side; no network or backend required.
- Works from `file://`.
- Mobile-friendly responsive layout with compact heatmap columns on narrow screens.
