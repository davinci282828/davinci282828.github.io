# TASK: Skill Utilization Heatmap

## Goal
Build a single-file HTML dashboard that visualizes OpenClaw AgentSkill usage patterns: which skills get invoked, which are dormant 30+ days, usage frequency heatmap, and basic ROI indicators.

## Requirements
1. **Data ingestion** (client-side only, no backend):
   - Accept drag-and-drop of session log JSONL files (from `~/.openclaw/sessions/`)
   - Parse installed skill list from a separate JSON manifest (structure: `{skills: [{name, category, install_date}]}`)
   - Parse session logs for skill invocations (search for patterns like `read.*SKILL.md` or skill name mentions in tool calls)
   
2. **Heatmap visualization**:
   - Grid layout with one cell per skill
   - Color-coded by usage frequency: green (used <7 days ago), yellow (7-30 days), red (30+ days), gray (never used)
   - Display skill name, category, and days since last use in each cell
   - Hoverable tooltip with: total invocations, last used date, install date
   
3. **Summary metrics** (top of page):
   - Total skills installed
   - Active skills (used <7 days)
   - Stale skills (7-30 days)
   - Dormant skills (30+ days)
   - Never used skills
   
4. **ROI indicators**:
   - For each skill: estimated time saved (placeholder formula: invocations × 15 min)
   - Total time saved across all skills
   - Top 5 most valuable skills by time saved
   
5. **Technical constraints**:
   - Single HTML file with inline CSS/JS
   - No external dependencies except Chart.js CDN (optional, prefer vanilla CSS grid)
   - Responsive design (mobile-friendly)
   - Dark theme
   - Works on file:// protocol (drag-and-drop, no fetch required)

## File Output
- `projects/yolo-builds/2026-03-25-skill-heatmap/index.html`
- `projects/yolo-builds/2026-03-25-skill-heatmap/README.md` (usage instructions)
- `projects/yolo-builds/2026-03-25-skill-heatmap/sample-manifest.json` (example skill manifest format)

## Acceptance Criteria (BDD)
- **Given** a user drags a session log JSONL file onto the dashboard
  **When** the file is parsed
  **Then** skill invocations are extracted and displayed in the heatmap with correct timestamps
  
- **Given** multiple skills with varying usage patterns
  **When** the heatmap is rendered
  **Then** cells are color-coded correctly (green <7d, yellow 7-30d, red 30+d, gray never)
  
- **Given** a user hovers over a skill cell
  **When** the tooltip appears
  **Then** it shows: total invocations, last used date, install date, estimated time saved
  
- **Given** skills with zero invocations
  **When** the dashboard loads
  **Then** they appear in gray with "Never used" status
  
- **Given** the dashboard loads on mobile
  **When** viewed on a narrow viewport
  **Then** the heatmap grid collapses to 2-3 columns and remains readable

## Anti-Compaction
If context compaction occurs, re-read this task file before continuing.
