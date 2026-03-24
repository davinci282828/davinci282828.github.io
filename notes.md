# Skill Utilization Heatmap — Build Notes
Build #27 — March 25, 2026

## Candidates Considered
1. **Mission Control Mobile Companion** — Duplicate effort. MC already has mobile client, no requested improvements.
2. **Slack Digest Quality Analyzer** — Too meta. Digest quality is Steven's call, no operational pain.
3. **Skill Utilization Heatmap** — SELECTED. 72 skills installed, many dormant. Operational intelligence gap.

## Selection Reasoning
- Reality signal: 0 (no competition)
- Non-duplicate: static audit exists (March 13) but no dynamic tracker
- Operational value: helps Da Vinci identify underutilized skills and ROI opportunities
- Data sources available: session logs, cron configs, skill directories

## Build Plan
1. Parse installed skills from `~/.openclaw/skills/` and workspace skills
2. Scan session logs for skill invocations (last 30 days)
3. Calculate dormancy (days since last use)
4. Visualize: heatmap grid, color-coded by usage frequency
5. ROI estimate: time saved vs install/maintenance cost

## Acceptance Criteria
- Loads skill list dynamically from directory scan
- Parses session logs for invocation timestamps
- Displays heatmap with color-coded cells (green=active, yellow=stale, red=dormant)
- Shows days since last use for each skill
- Works standalone (single HTML file, GitHub Pages compatible)
