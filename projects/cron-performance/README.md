# Cron Performance Analyzer

Dashboard for visualizing OpenClaw cron job performance — runtime trends, failure patterns, schedule density, and model usage.

## Usage

1. Export cron data:
   ```bash
   openclaw cron list --json > crons.json
   ```

2. Open `index.html` in a browser

3. Drag and drop `crons.json` onto the dashboard

## Features

- **Schedule Density Heatmap**: 24-hour × 7-day grid showing when jobs are clustered
- **Job Status Overview**: Sortable grid with status indicators (green/red/yellow dots)
- **Failure Pattern Timeline**: 7-day chart showing error frequency
- **Model Usage Breakdown**: Pie chart of which models run which jobs
- **Cross-visualization highlighting**: Click a job to highlight it across all charts
- **Filters**: View all jobs, errors only, or filter by model

## Sample Data

Dashboard loads with sample data for demonstration. Real data replaces it when you drag-drop a JSON file.

## Export Format

The dashboard expects JSON output from `openclaw cron list --json` with this structure:

```json
{
  "jobs": [
    {
      "id": "...",
      "name": "Job Name",
      "schedule": {
        "kind": "cron",
        "expr": "0 5 * * *",
        "tz": "America/New_York"
      },
      "state": {
        "lastRunStatus": "ok",
        "lastRunAtMs": 1774429200017,
        "nextRunAtMs": 1774515600000
      },
      "payload": {
        "model": "venice/claude-sonnet-45"
      }
    }
  ]
}
```

## Browser Compatibility

- Modern browsers (Chrome, Firefox, Safari, Edge)
- Chart.js loaded from CDN (requires internet connection)
- Works on file:// protocol with sample data
- Drag-drop file loading requires local http server or direct file access

## Notes

- Schedule parsing supports standard cron expressions (5-7 fields)
- "Every" schedules (e.g., every 30 minutes) are converted to equivalent cron expressions for display
- Failure timeline shows historical data if available (currently limited to last run status)
