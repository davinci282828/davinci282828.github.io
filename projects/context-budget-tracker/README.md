# Agent Context Budget Tracker

Single-page dashboard for monitoring AI session context usage against a 1,000,000-token window. It tracks cumulative input/output tokens, utilization %, context zone (SAFE/WARNING/CRITICAL), and model-based cost estimates.

## Files

- `index.html` — self-contained app (inline CSS/JS) using Chart.js via CDN
- `README.md` — usage and data format reference

## Usage

1. Open `index.html` in a browser.
2. Choose **Session JSON** or **Manual totals** as the token source.
3. For JSON mode:
   - Paste output from `openclaw sessions --active 60 --json`.
   - Click **Parse JSON** (or wait for auto-parse while typing/pasting).
4. For manual mode:
   - Enter cumulative **input tokens** and **output tokens** directly.
5. Select a pricing model from the dropdown.
6. Review totals, percentage used, zone badge, cost estimate, and the accumulation chart.

## What the dashboard shows

- Input token total (cumulative)
- Output token total (cumulative)
- Total tokens = input + output
- Percentage of context limit used (default limit: 1,000,000)
- Remaining tokens
- Running cost estimate from selected model rates
- Zone indicator:
  - **SAFE** (green): `< 60%`
  - **WARNING** (yellow): `60% to 80%`
  - **CRITICAL** (red): `> 80%` (suggests restarting session soon)
- Line chart:
  - x-axis: message count
  - y-axis: cumulative tokens

## JSON format specification

The parser is tolerant and accepts common shapes. Best-supported formats include:

### 1) Session object with messages array

```json
{
  "messages": [
    {"input_tokens": 1200, "output_tokens": 300},
    {"input_tokens": 900, "output_tokens": 450}
  ]
}
```

### 2) Arrays/objects using usage fields

```json
{
  "events": [
    {"usage": {"input_tokens": 1000, "output_tokens": 250}},
    {"usage": {"input_tokens": 1100, "output_tokens": 260}}
  ]
}
```

### 3) Totals-only fallback (single-point chart)

```json
{
  "total_input_tokens": 450000,
  "total_output_tokens": 100000
}
```

## Token field aliases recognized

Input-like fields:
- `input_tokens`, `prompt_tokens`, `tokens_in`, `usage.input_tokens`, `usage.prompt_tokens`, `usage.input`

Output-like fields:
- `output_tokens`, `completion_tokens`, `tokens_out`, `usage.output_tokens`, `usage.completion_tokens`, `usage.output`

Total fallback fields:
- `total_tokens`, `usage.total_tokens`

If both input/output are missing but `total_tokens` is present per message, total is counted as input for aggregation.

## Model pricing table (per 1M tokens)

| Model | Input / 1M | Output / 1M |
|---|---:|---:|
| Claude Opus 4 | $15.00 | $75.00 |
| Claude Sonnet 4.5 | $3.00 | $15.00 |
| Kimi K2.5 | $0.30 | $1.20 |
| GPT-4o | $2.50 | $10.00 |
| Llama 3.3 70B | $0.35 | $0.40 |

Cost formula:

`cost = (input_tokens / 1,000,000 * input_rate) + (output_tokens / 1,000,000 * output_rate)`

## Validation examples

- Manual `450,000 input + 100,000 output` -> `550,000 total`, `55%` of 1M limit.
- A JSON session with 50 messages will render 50 line-chart points (one cumulative point per message).

