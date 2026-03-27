# CEO Operations System — Deterministic System Spec V2.1

_Created: 2026-03-26 | V2: 2026-03-26 23:40 ET | V2.1: 2026-03-27 00:00 ET_
_Status: DRAFT V2.1 — incorporates Steven's second review (4 fixes)_
_Principle: Manual first → prove quality → semi-auto → full auto_

---

## System 0: Operations Heartbeat (Meta-System)

Runs after all other systems. Confirms operational integrity.

### Trigger
Daily at 9:30 PM (after Systems 1 and 5 complete their 8-9 PM window).

### Processing
```
1. Check: did each enabled system run today?
   - System 1 (Agendas): check if agenda was written to projects/ceo-ops-system/runs/agendas/YYYY-MM-DD.md
   - System 2 (Slack): check if screening was written to projects/ceo-ops-system/runs/screening/YYYY-MM-DD.md
   - System 3 (Follow-up): Monday only — check if report exists
   - System 4 (Marketing): Monday only — check if report exists
   - System 5 (Calendar): check if brief was written to projects/ceo-ops-system/runs/calendar/YYYY-MM-DD.md
2. Read SCORECARD.md → verify it was updated after each delivery
3. Check for any Steven override flags in projects/ceo-ops-system/overrides.json
4. Check shared data freshness (see Shared Data Layer below)
   - If roster >14 days stale: auto-run `scripts/refresh-team-roster.sh`, alert Steven only if new gaps found
5. WEEKLY (Fridays only): Read corrections.log. Group entries by root cause keyword.
   IF 3+ entries share the same root cause → flag: "Recurring issue: [root cause] — appeared in [N] corrections across [systems]"
   This catches cross-system patterns (e.g., "roster staleness" causing errors in both Systems 1 and 3).
6. Compile status
```

### Output
```
🔄 CEO OPS HEARTBEAT — [Date]

Systems: [N]/[N] ran
✅ Agendas: delivered (3 meetings)
✅ Calendar: delivered
⚠️ Slack Screening: skipped (Steven override — "pause until Monday")
➖ Follow-up: not scheduled today (Monday only)
➖ Marketing: not scheduled today (Monday only)

Scorecard: updated ✅
Data freshness: roster 14 days stale ⚠️ | tracker OK | calendar OK
Overrides active: Slack paused until 2026-03-31
```

Delivery: Telegram topic 1 (only if something is ⚠️ or ❌). Silent if all green — Steven doesn't need noise confirming everything worked.

**Exception:** On Mondays (all systems active), always deliver the heartbeat regardless.

---

## Shared Data Layer

These inputs are shared across multiple systems. Pulled ONCE per cycle, stored locally, consumed by all systems.

### Shared Resources

| Resource | Pulled By | When | Stored At | Consumed By | Freshness Requirement |
|----------|-----------|------|-----------|-------------|----------------------|
| Tomorrow's calendar | System 5 | 8 PM | `runs/shared/calendar-YYYY-MM-DD.json` | Systems 1, 5 | Daily (same-day pull required) |
| Team roster | `scripts/refresh-team-roster.sh` | On demand (auto-triggered by System 0 when >14 days stale) | `projects/fireflies-accountability/team-roster.json` | Systems 1, 2, 3 | <14 days. If >14 days: System 0 runs refresh script automatically, then alerts Steven only if new gaps found |
| Action tracker snapshot | System 3 | Monday 8:30 AM | `runs/shared/tracker-YYYY-MM-DD.json` | Systems 1, 3 | Weekly (Monday pull). System 1 uses latest available snapshot on non-Mondays |
| Slack messages (last 12h) | System 2 | After digest (10:15 AM, 4:45 PM) | `runs/shared/slack-YYYY-MM-DD-[am|pm].json` | Systems 1, 2 | Per-run |

### Pull Commands (exact)
```bash
# Calendar (System 5 pulls, System 1 consumes)
gog calendar list --cal steven@manhattanmhc.com --from tomorrow --to tomorrow --format json > runs/shared/calendar-YYYY-MM-DD.json

# Team roster refresh (auto-triggered by System 0 when >14 days stale)
bash scripts/refresh-team-roster.sh
# Validates structure, updates timestamp, reports gaps (missing Slack IDs, missing roles)
# Preserves all existing role/department/channel annotations

# Action tracker
gog sheets get 1aWQ9N8bJHAR9ZecxNOoEC9CqLvaIWfH5Z0M6v4Bs-TI --format json > runs/shared/tracker-YYYY-MM-DD.json
```

### Inter-System Data Flow
```
System 5 (8 PM) → pulls calendar → writes shared/calendar-YYYY-MM-DD.json
System 1 (9 PM) → reads shared/calendar-YYYY-MM-DD.json (never re-pulls)
System 0 (9:30 PM) → reads all run outputs + SCORECARD.md → validates
```

This eliminates duplicate API calls and ensures Systems 1 and 5 see identical calendar data.

---

## Context Persistence & SCORECARD Schema

### SCORECARD.md Format
```markdown
# CEO Ops — Trust Scorecard
# Written by: Da Vinci (after every delivery)
# Read by: Da Vinci (before every delivery, to check current phase + streak)

| system | phase | streak | last_error_date | last_error_desc | last_delivery | gate_target | gate_met |
|--------|-------|--------|-----------------|-----------------|---------------|-------------|----------|
| agendas | 1 | 3 | none | none | 2026-03-29 | 15 | no |
| calendar | 1 | 3 | none | none | 2026-03-29 | 20 | no |
| slack | 1 | 4 | none | none | 2026-03-29 | 20 | no |
| followup | 1 | 1 | none | none | 2026-03-31 | 8 | no |
| marketing | 0 | 0 | none | none | none | 4 | no |
```

### Read/Write Protocol
- **After every delivery:** Da Vinci reads SCORECARD.md → increments streak (or resets to 0 on error) → writes updated SCORECARD.md → logs update in daily log
- **Before every delivery:** Da Vinci reads SCORECARD.md → checks current phase → runs appropriate phase logic
- **On Steven 👍:** Increment streak. Source: Steven reacts with 👍 emoji to the delivery message in Telegram. Da Vinci monitors reactions on delivery messages only (not other messages in the thread).
- **On Steven 👎:** Reset streak to 0. Log the error in corrections log (see below). Da Vinci reads the 👎 and asks Steven: "What was wrong?" in thread reply.
- **On no reaction within 24h:** Count as neutral — do not increment or reset. Log: "No rating received."

### Feedback Capture (exact mechanism)
Steven reacts to delivery messages with:
- 👍 = good, streak increments
- 👎 = error, streak resets, corrections log entry required
- No reaction = neutral, no streak change

Da Vinci checks for reactions on previous delivery messages at the start of each run. Uses Telegram message ID stored in `runs/[system]/YYYY-MM-DD.md` metadata header.

**Ambiguity resolution:** Only reactions on messages sent by Da Vinci that match the system's output format header (📋 for agendas, 📅 for calendar, 🔔 for Slack, 📊 for follow-up, 📈 for marketing) count. Reactions on other messages are ignored.

---

## Corrections Log

Location: `projects/ceo-ops-system/corrections.log`

### Format
```
[DATE] | [SYSTEM] | [WHAT WENT WRONG] | [CORRECT OUTPUT] | [RULE CHANGE]
2026-03-28 | agendas | Listed cancelled meeting as active | Should have excluded declined/cancelled | Add filter: exclude events with status=cancelled or responseStatus=declined
```

### Protocol
- Every 👎 reaction → corrections log entry required within same session
- Da Vinci asks Steven: "What was wrong?" — logs the answer
- Da Vinci proposes a rule change to prevent recurrence
- Rule change applied to processing logic before next run
- If same error type occurs twice → escalate: flag to Steven that the fix didn't work

---

## Steven Override Mechanism

Steven can pause, skip, or modify any system without breaking streak counts.

### Override Commands (in Telegram)
Steven says any of these and Da Vinci acts:
- "Skip agendas tonight" → System 1 skipped, streak preserved, logged in overrides
- "Pause Slack screening until Monday" → System 2 paused with end date
- "No calendar brief this week" → System 5 paused for 7 days
- "Stop [system]" → System frozen until Steven says "resume [system]"

### Implementation
Overrides stored in: `projects/ceo-ops-system/overrides.json`
```json
{
  "overrides": [
    {
      "system": "slack",
      "action": "pause",
      "until": "2026-03-31",
      "reason": "Steven: 'pause until Monday'",
      "created": "2026-03-28T15:00:00"
    }
  ]
}
```

Every system checks `overrides.json` before running. If paused → skip silently. System 0 reports active overrides in heartbeat.

Overrides do NOT reset streaks. Overrides DO get logged in SCORECARD.md notes.

---

## Rollback Protocol

### Demotion Rules
- **Auto-demotion:** If a Phase 2 system produces 3 errors in 10 consecutive deliveries → auto-demote to Phase 1. Da Vinci alerts Steven: "[System] demoted to Phase 1 — 3 errors in last 10 runs."
- **Manual demotion:** Steven says "demote [system]" → immediate Phase 1, streak reset.
- **Post-demotion:** System must re-earn its full phase gate from scratch. No partial credit.
- **Phase 3 → Phase 2 demotion:** Same rules. Phase 3 re-approval requires fresh explicit Steven approval.

---

## System 1: Meeting Agendas

### Current State
Pre-brief cron (`71cface9`) fires 30 min before meetings. Drops context paragraph to Telegram. No structure, no action items, no attendee history.

### Target State
Structured agenda for all of tomorrow's meetings, delivered as a nightly batch.

### Trigger
- **Phase 1 (manual):** Da Vinci runs nightly at 9 PM for next day's meetings (batch delivery)
- **Phase 2 (semi-auto):** Cron at 9 PM, auto-delivers batch to Telegram
- **Phase 3 (auto):** Same nightly batch PLUS per-meeting real-time updates 45 min before each meeting if anything changed since the 9 PM batch (new Slack threads, calendar changes). Existing pre-brief cron (`71cface9`) remains for day-of context — agendas are the strategic prep, pre-briefs are the tactical reminder.

### Inputs
1. `runs/shared/calendar-YYYY-MM-DD.json` (pulled by System 5 at 8 PM — never re-pull)
2. Fireflies extractions (existing pipeline — uses Google Drive, NOT Fireflies API):
   ```bash
   # Daily extractions auto-land here via cron cc7c7c02:
   ls projects/fireflies-accountability/extractions/extraction_YYYY-MM-DD_*.json
   # Each file contains: meeting title, date, attendees, action items with assignees
   # Search by attendee name:
   python3 -c "
   import json, glob
   files = sorted(glob.glob('projects/fireflies-accountability/extractions/extraction_*.json'))[-30:]
   for f in files:
       data = json.load(open(f))
       for meeting in data.get('meetings', []):
           if '<attendee_name>' in str(meeting.get('attendees', [])):
               print(json.dumps(meeting, indent=2))
   "
   ```
3. Action items from Fireflies (same extraction files — no separate API call needed):
   ```bash
   # Action items are pre-extracted in each extraction file, field: action_items[]
   # Each item has: text, assignee, priority, section
   ```
4. Action tracker: `runs/shared/tracker-YYYY-MM-DD.json` (latest available snapshot)
5. Team roster: `projects/fireflies-accountability/team-roster.json`
6. Slack search per attendee (requires Slack ID from roster):
   ```bash
   # Via openclaw message tool:
   message(action=read, channel=slack, query="from:<user_id>", limit=20, after=<7_days_ago_ISO>)
   ```

### Processing (deterministic steps)
```
1. Read shared calendar JSON
2. Filter: exclude events where status=cancelled or Steven's responseStatus=declined
3. Count remaining meetings. If 0 → deliver "No meetings tomorrow" → done.
4. FOR each meeting:
   a. Extract: title, start_time, end_time, attendees (email→name via roster), location/link
   b. IF attendees is empty or Steven is the only attendee:
      → Solo event. Output abbreviated block:
        "📋 [Title] — [Time] | Solo block — no attendees, no prep needed"
      → Skip steps c-d, continue to next meeting
   c. FOR each attendee (excluding Steven):
      i.   Roster lookup → role, department, Slack ID. If not found: role="unknown (not in roster)"
      ii.  Fireflies search by name → get last 3 meeting IDs with this person
      iii. FOR each meeting ID: pull summary → extract action items
      iv.  Cross-reference action items with tracker → filter to status ≠ "complete"
      v.   Slack search by user ID → last 7 days → extract threads with questions or @steven mentions
   c. Classify meeting type:
      - Attendees count = 2 (Steven + 1) → "1:1"
      - Attendees in same department → "team"
      - Any attendee not in roster → "external"
      - Default → "group"
   d. Build agenda sections:
      - CONTEXT: synthesize from most recent Fireflies summary with this attendee group
      - DECISIONS: items from Slack threads with questions + overdue action items needing escalation
      - OPEN ITEMS: per attendee, sorted by age descending, with exact dates
      - PREP: any Google Doc/Sheet links found in calendar event description or recent Slack threads
5. Run quality gate
6. Write to runs/agendas/YYYY-MM-DD.md (with Telegram message ID header placeholder)
7. Deliver to Telegram
8. Record message ID in run file header
9. Update SCORECARD.md
```

### Output Format
```
📋 TOMORROW'S AGENDAS — [Day, Date]
[N] meetings

---
📋 [Meeting Title] — [Time]
👥 [Name (Role), Name (Role)]
📍 [Location/Link]

CONTEXT
[2-3 sentences: what this meeting is about, what happened last time]

DECISIONS NEEDED
• [Decision — who needs it, why now]
(or: "No pending decisions identified")

OPEN ITEMS
[Name]:
  • [Item] — assigned [YYYY-MM-DD], [N] days ago
[Name]:
  • [Item] — assigned [YYYY-MM-DD], [N] days ago
(or: "[Name]: no open items")

PREP
• [Doc/link]
(or: "No prep materials identified")

---
[Next meeting...]
```

### Quality Gate
ALL must pass:
- [ ] Meeting count matches filtered calendar (log: "Calendar: N events, N after filtering cancelled/declined")
- [ ] Every attendee has a role (or explicitly "unknown — not in roster")
- [ ] Every action item has a date in YYYY-MM-DD format (not "recently")
- [ ] Decisions section populated OR explicitly "none identified" (never blank)
- [ ] All Fireflies lookups returned data OR explicit "no meeting history found for [name]"
- [ ] No attendee silently skipped (all listed even if no data found)

Fail → do not deliver. Log failure to `runs/agendas/YYYY-MM-DD-FAILED.md` with reason. Alert Steven: "Agenda failed: [reason]."

### Failure Modes
| Failure | Response |
|---------|----------|
| Calendar data missing (System 5 didn't run) | Pull calendar directly as fallback. Log: "System 5 missed — pulling calendar independently" |
| Fireflies extraction files missing/empty (cron cc7c7c02 didn't run) | Deliver WITHOUT action items per attendee. Flag: "⚠️ Fireflies extractions missing — action items unavailable" |
| Attendee not in roster | Include with "Role: unknown (not in roster)" — never skip |
| No meetings tomorrow | Deliver: "No meetings tomorrow ✅" (confirmation, not silence) |
| Slack search fails | Deliver without Slack context. Flag: "⚠️ Slack search failed — recent threads missing" |
| Action tracker stale (>7 days) | Use stale data. Flag: "⚠️ Action tracker last refreshed [date]" |

### Measurement
- **Accuracy:** Steven reacts 👍/👎 per SCORECARD feedback protocol. Target: 90%+ 👍 over rolling 10.
- **Coverage:** % of meetings that got an agenda. Target: 100%. Measured: meetings in output ÷ meetings in calendar.
- **Action item freshness:** Weekly spot-check: pick 3 random items from last agenda, verify they're actually still open. Target: >90% accurate.
- **Phase gate:** 15 consecutive 👍 with 0 factual errors → Da Vinci proposes Phase 2, Steven approves.

### Cost Budget
Estimated per run (8-meeting day, 3 attendees avg):
- Calendar pull: shared (free via System 5)
- Fireflies: 0 cost (reads local extraction JSON files, no API calls)
- Slack: ~24 message reads ≈ 0 cost (API)
- Action tracker: shared (free via System 3 pull)
- LLM (synthesis/formatting): ~4K tokens in, ~2K out per meeting × 8 = ~32K in, ~16K out
- **Estimated cost per run: ~$0.15 (Venice Sonnet) or $0 if done in main Opus session**

---

## System 2: Slack Thread Screening

### Current State
2x daily Slack digest (cron `015ff99e`, 10 AM + 4:30 PM). Summarizes everything. Steven has to scan the whole digest to find what needs him.

### Target State
Separate signal from noise. Steven gets ONLY threads requiring his decision, with the decision framed clearly.

### Trigger
- **Phase 1 (manual):** Da Vinci runs after each existing digest (~10:15 AM, ~4:45 PM)
- **Phase 2 (semi-auto):** Replaces digest cron with filtered version
- **Phase 3 (auto):** Same, earned after 20 consecutive screenings with 0 missed decisions

### Inputs
1. Slack digest output (existing cron captures all channels)
2. Direct Slack API query for @steven mentions:
   ```bash
   message(action=read, channel=slack, query="<@U07BMJJ3CBS>", limit=50, after=<12h_ago_ISO>)
   ```
3. Slack API query for approval/decision keywords in channels Steven is in:
   ```bash
   message(action=read, channel=slack, query="approve OR sign off OR your call OR waiting on you OR need you to", limit=30, after=<12h_ago_ISO>)
   ```
4. Thread context for flagged messages:
   ```bash
   message(action=read, channel=slack, channelId=<channel_id>, threadId=<thread_ts>, limit=20)
   ```

### Processing
```
1. Collect all messages from digest window (inputs 1-3, deduplicated by message ID)
2. FOR each message/thread:
   a. Apply DECISION classifier:

      DECISION (any ONE = flag):
        - Contains <@U07BMJJ3CBS> AND contains "?" (question to Steven)
        - Contains: "approve", "sign off", "your call", "need you to", "waiting on you", "can you", "should we"
        - Contains deadline keyword ("by EOD", "by Friday", "deadline", "due", "urgent", "ASAP") within 48h
        - Thread has 3+ participants disagreeing (detect: "but", "however", "I think", "disagree", "alternatively" from 2+ users)
        - Contains: "budget", "spend", "hire", "fire", "terminate", "contract"
        - Contains: "patient complaint", "escalat", "incident"

      FYI (ALL of these = skip):
        - No question mark AND no approval keywords
        - Status update patterns: "update:", "FYI:", "done:", "completed", "shipped"
        - Single-person thread (monologue update)
        - Bot messages (automated notifications)

      AMBIGUOUS: flag as DECISION (false positive > false negative)

   b. If DECISION:
      i.   Pull full thread context (up to 20 messages)
      ii.  Frame the decision as a question: "[Person] needs you to decide: [what?]"
      iii. Extract deadline if present
      iv.  Suggest options if the thread contains alternatives

3. Group DECISION items by urgency: deadline within 24h first, then 48h, then no deadline
4. Count FYI items (for summary line, not delivered individually)
5. Run quality gate
6. Write to runs/screening/YYYY-MM-DD-[am|pm].md
7. Deliver
8. Update SCORECARD.md
```

### Output Format
```
🔔 SLACK: [N] items need your decision

1. #[channel] — [Person]: [Decision framed as question]
   Context: [1-2 sentences]
   ⏰ Deadline: [date/time] (or omit if none)

2. #[channel] — [Person]: [Decision framed as question]
   Context: [1-2 sentences]

---
Filtered: [N] FYI threads (no action needed)
[1-line notable mention if anything interesting but not decision-requiring]
```

### Quality Gate
- [ ] Every DECISION item has the decision framed as a question (not a summary)
- [ ] No message containing `<@U07BMJJ3CBS>` + `?` classified as FYI
- [ ] Channel names are human-readable (not IDs)
- [ ] Person names are human-readable (resolved from Slack IDs via roster)
- [ ] Deadline times are in ET (Steven's timezone)

### Failure Modes
| Failure | Response |
|---------|----------|
| Slack API down | Alert: "Slack screening failed — check digest manually" |
| No decisions found | Deliver: "✅ No decisions needed — [N] FYI threads filtered" |
| Misclassification caught by Steven | Log to corrections.log. If 3+ in 10 screenings → auto-pause, retune rules, alert Steven |

### Recall Audit (anti-circular measurement)
Every Friday at 8:30 PM ET (after System 5's weekly overview at 8 PM, before System 1's agenda run at 9 PM), Da Vinci generates a **recall audit**: pull 5 random FYI-classified threads from the week and present them to Steven as a batch:
```
🔍 WEEKLY RECALL CHECK — was anything miscategorized?
1. #channel — [summary of FYI-classified thread]
2. #channel — [summary]
3. #channel — [summary]
4. #channel — [summary]
5. #channel — [summary]
Should any of these have been flagged as decisions? Reply with numbers or "all good."
```
Any miss found in audit = streak reset + corrections log entry.

### Measurement
- **Precision:** % of DECISION items Steven acted on. Target: 70%+.
- **Recall:** Missed decisions. Target: 0. Measured via: (a) Steven self-reports, (b) Friday recall audit.
- **Phase gate:** 20 consecutive screenings with 0 missed decisions (including audit-caught misses) → propose Phase 2.

### Cost Budget
- Slack API calls: ~80-100 per run ≈ $0 (API)
- LLM (classification + framing): ~8K tokens in, ~2K out per run
- **Estimated cost per run: ~$0.05 (Venice Sonnet) or $0 in main session**

---

## System 3: Fireflies Action Item Follow-up

### Current State
Pipeline V2 exists. 1100+ items in tracker. 32+ meetings processed. No automated staleness detection.

### Target State
Weekly staleness report to Steven. No automated DMs. Steven decides what to act on.

### Trigger
- **Phase 1 (manual):** Da Vinci generates report every Monday at 9 AM
- **Phase 2 (semi-auto):** Cron generates and delivers to Telegram Monday 9 AM
- **Phase 3 (semi-autonomous):** Same report, PLUS Da Vinci drafts suggested follow-up messages per CRITICAL item. Steven approves or rejects each draft before any message is sent. This is NOT fully autonomous — it's a human-in-the-loop draft/approve workflow. Fully autonomous DMs remain GATED behind separate trust threshold (see Trust Accounting).

### Inputs
1. Action tracker snapshot:
   ```bash
   gog sheets get 1aWQ9N8bJHAR9ZecxNOoEC9CqLvaIWfH5Z0M6v4Bs-TI --format json > runs/shared/tracker-YYYY-MM-DD.json
   ```
2. Fireflies extractions (existing pipeline via Google Drive, NOT API):
   ```bash
   # Search last 30 days of extraction files for assignee name:
   python3 -c "
   import json, glob
   files = sorted(glob.glob('projects/fireflies-accountability/extractions/extraction_*.json'))[-30:]
   for f in files:
       data = json.load(open(f))
       for meeting in data.get('meetings', []):
           if '<assignee_name>' in str(meeting):
               print(f'{meeting.get(\"title\")} ({meeting.get(\"date\")}): discussed')
   "
   ```
3. Slack activity per assignee:
   ```bash
   message(action=read, channel=slack, query="from:<user_slack_id>", limit=5, after=<7_days_ago_ISO>)
   ```
4. Calendar (last 14 days, shared pull or direct):
   ```bash
   gog calendar list --cal steven@manhattanmhc.com --from -14d --to today --format json
   ```

### Processing
```
1. Read tracker JSON. Filter: status ≠ "complete" AND status ≠ "cancelled"
2. FOR each open item:
   a. age_days = (today - assigned_date).days
   b. Search Fireflies for assignee in last 30 days → was this item's topic discussed?
      discussed_since = true if any meeting summary mentions item keywords
   c. Check Slack for assignee activity in last 7 days:
      active_in_slack = true if any messages found
   d. Check calendar for Steven meeting with assignee in last 14 days:
      met_recently = true if any calendar event includes assignee

   e. CLASSIFY:
      - FRESH: age_days < 7 OR discussed_since = true
      - AGING: 7 ≤ age_days < 14 AND NOT discussed_since
      - STALE: 14 ≤ age_days < 30 AND NOT discussed_since
      - CRITICAL: age_days ≥ 30 AND NOT discussed_since AND active_in_slack = true
      - DORMANT: age_days ≥ 30 AND NOT discussed_since AND active_in_slack = false

3. Group by assignee. Sort: CRITICAL first, then STALE, then AGING.
4. FOR each person with STALE+ items:
   a. total_open = count of their open items
   b. Count per tier
   c. last_steven_meeting = most recent calendar event with them
   d. last_slack_msg = date of most recent Slack message
5. Calculate:
   - total_open_all = sum of all open items
   - completion_rate_30d = items marked complete in last 30 days ÷ (items marked complete + items still open that were assigned >30 days ago)
6. Quality gate
7. Write to runs/followup/YYYY-MM-DD.md
8. Deliver
9. Update SCORECARD.md
```

### Output Format
```
📊 ACCOUNTABILITY REPORT — Week of [date]

CRITICAL (>30 days, person active, item ignored):
• [Person] ([Role]) — [N] items
  - "[Item]" — assigned [YYYY-MM-DD] ([N] days)
  - "[Item]" — assigned [YYYY-MM-DD] ([N] days)
  Last meeting with Steven: [YYYY-MM-DD]. Last Slack activity: [YYYY-MM-DD].

STALE (14-30 days, not discussed):
• [Person] — [N] items
  - "[Item]" — [N] days

AGING (7-14 days):
• [Person]: [N] items (count only)

SUMMARY
Total open: [N] | Fresh: [N] | Aging: [N] | Stale: [N] | Critical: [N] | Dormant: [N]
30-day completion rate: [N]%
Trend: [↑/↓/→] vs last week ([N] stale last week → [N] this week)
```

### Quality Gate
- [ ] Every item has assigned_date in YYYY-MM-DD format (items without dates: flag "date unknown" but include)
- [ ] Assignee names match roster. Items with unknown assignees: include as "Unassigned: [N] items"
- [ ] Spot-check 3 random items: verify staleness tier is correct by manually checking Fireflies/Slack
- [ ] No duplicate items (deduplicate by item title + assignee)
- [ ] Completion rate denominator > 0 (avoid division by zero)

### Trust Accounting Exception
System 3 deals with 1100+ items where some classification noise is expected (stale tracker data, ambiguous assignees). The master rule "1 error = streak reset" applies to FACTUAL errors (wrong dates, wrong names, wrong math). Classification disagreements (Steven thinks an item is STALE, system classified it as AGING) count as corrections log entries but do NOT reset the streak unless the root cause is a bug in the processing logic rather than a judgment call on item status.

Steven retains override: "That's a factual error, not a judgment call" → streak resets.

### Failure Modes
| Failure | Response |
|---------|----------|
| Tracker sheet API down | Alert: "Can't pull action items — report delayed to Tuesday" |
| Extraction files missing or cron cc7c7c02 didn't run | Generate WITHOUT "discussed recently" data. All items get conservative classification (assume not discussed). Flag: "⚠️ Fireflies extractions missing for [dates]" |
| Assignee not in roster | Include with name from tracker, mark "not in roster — Slack activity unknown" |
| 0 open items | Deliver: "🎉 Zero open items. Either the team is crushing it or the tracker needs updating." |

### Measurement
- **Action rate:** % of CRITICAL items Steven acts on within 7 days of receiving report. Target: 50%+.
- **Staleness trend:** Week-over-week total stale+critical items. Should decrease over time.
- **Accuracy:** Steven flags miscategorized items. Target: <5% per report (see Trust Accounting Exception above).
- **Phase gate:** 8 consecutive weekly reports with <5% factual error rate → propose Phase 2.

### Cost Budget
- Sheets API: 1 call ≈ $0
- Fireflies: ~20 search calls (for unique assignees) ≈ $0
- Slack: ~20 activity checks ≈ $0
- LLM (synthesis): ~10K tokens in, ~3K out
- **Estimated cost per run: ~$0.06 or $0 in main session**

---

## System 4: Marketing Intelligence Closed Loop

### Current State
Content factory produces batches. Competitive intel runs weekly. Google Ads skills built, unused. GEO/AEO baseline measured. No feedback loop.

### Status: DESIGN PHASE
This system is not at the same specificity as Systems 1-3. The exact tool chains depend on:
1. **GA API verification** — OAuth exists but hasn't been tested end-to-end
2. **Aaron's ad performance data** — not currently shared
3. **Content-to-URL mapping** — which posts map to which GA pages

I'm flagging this honestly: Systems 1-3 are build-ready. System 4 needs a discovery sprint before it can be specced at the same level.

### Discovery Sprint (Week of April 7)
```
Day 1: Verify GA API end-to-end
  - Run: gog analytics report --property <property_id> --dimensions pagePath --metrics sessions,engagementRate --dateRange last7days
  - If fails: diagnose OAuth, property access, API enablement
  - If succeeds: document exact commands, available metrics, data format

Day 2: Map content to GA
  - Pull list of URLs from GA (top pages)
  - Cross-reference with content factory batch log (projects/mmhc-instagram-factory/batches/)
  - Determine: can we track individual post performance? Or only page-level?

Day 3: Aaron data gap assessment
  - Steven to ask Aaron: "Can you share a weekly Google Ads performance export to our shared Drive?"
  - If yes: define format, folder, cadence
  - If no: design System 4 without ad data (competitive intel + GA only)

Day 4-5: Write deterministic spec at Systems 1-3 level
```

**Action required from Steven:** Ask Aaron to share weekly Google Ads export by April 7. Specific ask: "Aaron, can you drop a weekly Google Ads performance CSV in our shared Drive folder? Metrics: campaign, ad group, clicks, impressions, conversions, spend, by week."

### Interim Deliverable (starts immediately)
Until the discovery sprint, System 4 runs in COMPETITIVE INTEL ONLY mode:
- Weekly: pull latest Scrapling briefing from `projects/scrapling-intelligence/briefings/`
- Summarize: what competitors changed, new themes, volume shifts
- Deliver to Telegram Monday morning

This is not the closed loop — it's the competitive input half running on its own until we can wire in the performance feedback half.

### Phase Gate
Discovery sprint completion + Steven approval of full spec → System 4 enters Phase 1.

---

## System 5: Proactive Calendar Management

### Current State
Can read calendar. Pre-briefs fire before meetings. No calendar health analysis.

### Target State
Daily calendar health report. Flag conflicts, back-to-backs, missing focus time. READ-ONLY in Phases 1-2. Phase 3 write access requires separate explicit approval.

### Trigger
- **Phase 1 (manual):** Da Vinci runs at 8 PM for next day + weekly summary Friday 8 PM
- **Phase 2 (semi-auto):** Cron at 8 PM daily, auto-delivers
- **Phase 3 (read-write):** Same + creates "Focus Time" blocks. **Requires separate explicit Steven approval — not auto-earned from streak. Steven must say "you can write to my calendar."**

### Inputs
1. Tomorrow's calendar (this system pulls it for shared use):
   ```bash
   gog calendar list --cal steven@manhattanmhc.com --from tomorrow --to tomorrow --format json
   ```
   Write to: `runs/shared/calendar-YYYY-MM-DD.json`

2. Next 7 days (for weekly):
   ```bash
   gog calendar list --cal steven@manhattanmhc.com --from tomorrow --to +7days --format json
   ```

3. Previous week's calendar brief (for trend comparison):
   Read: `runs/calendar/YYYY-MM-DD.md` from last week

### Processing
```
DAILY (8 PM):
1. Pull tomorrow's calendar → save to shared location
2. Parse events: extract start_time, end_time, title, attendee_count
3. Filter: exclude status=cancelled, responseStatus=declined, all-day events (for time calculation)
4. Sort by start_time

5. CONFLICT CHECK:
   FOR i, j in all event pairs:
     IF event_i.end > event_j.start AND event_i.start < event_j.end:
       flag CONFLICT(event_i, event_j)

6. BACK-TO-BACK CHECK:
   FOR consecutive events (sorted by start_time):
     gap = next.start - current.end
     IF gap < 15 minutes AND gap ≥ 0:
       flag BACK_TO_BACK(current, next)

7. MEETING DENSITY:
   total_meetings = count of events
   total_hours = sum of (end - start) for all events
   IF total_meetings > 5: flag HEAVY_DAY

8. FOCUS TIME:
   working_hours = 9:00 AM to 6:00 PM ET
   occupied_blocks = sorted list of (start, end) for all events within working hours
   free_blocks = gaps between occupied_blocks within working hours
   focus_blocks = [block for block in free_blocks if block.duration ≥ 90 min]
   IF len(focus_blocks) == 0: flag NO_FOCUS_TIME

9. EARLY/LATE CHECK:
   FOR each event:
     IF start_time < 9:00 AM: flag EARLY
     IF end_time > 6:00 PM: flag LATE

10. Generate brief
11. Quality gate
12. Write to runs/calendar/YYYY-MM-DD.md
13. Deliver
14. Update SCORECARD.md

WEEKLY (Friday 8 PM, appended to daily):
1. Pull next 7 days
2. Run steps 5-9 for each day
3. Aggregate:
   total_meeting_hours_next_week = sum
   total_focus_hours_next_week = sum of focus blocks
   busiest_day = day with most meetings
   lightest_day = day with fewest
4. Compare to this week (read previous Friday's weekly from runs/calendar/)
5. Generate weekly addendum
```

### Output Format (Daily)
```
📅 TOMORROW: [Day, Date]
[N] meetings | [N]h booked | [N]h focus time

⚠️ CONFLICTS:
• [Time]: "[Meeting A]" overlaps "[Meeting B]"
(or: "None")

⚠️ BACK-TO-BACKS:
• [Time]-[Time]: "[Meeting A]" → "[Meeting B]" (0 min break)
(or: "None")

FOCUS BLOCKS:
• [Start]-[End] ([N] min)
• [Start]-[End] ([N] min)
(or: "⚠️ No focus blocks ≥90 min")

🌅 Early: "[Meeting]" at [time]
🌙 Late: "[Meeting]" until [time]
(omit section if none)

SUGGESTION:
• [Specific suggestion if any flags, e.g., "Consider moving [meeting] to create a 2h focus block in the morning"]
(or: "Calendar looks clean ✅")
```

### Output Format (Weekly, Friday addition)
```
📅 NEXT WEEK OVERVIEW
[N] meetings | [N]h booked | [N]h focus time
Busiest: [Day] ([N] meetings) | Lightest: [Day] ([N] meetings)
vs. this week: [↑N/↓N/→] meetings, [↑N/↓N/→]h focus time
```

### Quality Gate
- [ ] Meeting count matches calendar after filtering (log raw vs filtered count)
- [ ] Conflict detection verified: for each flagged conflict, confirm times actually overlap (start1 < end2 AND start2 < end1)
- [ ] Focus time excludes all-day events and tentative events from blocking
- [ ] Declined meetings excluded (not phantom meetings)
- [ ] All times in ET

### Failure Modes
| Failure | Response |
|---------|----------|
| Calendar API down | Alert: "Can't pull calendar — no brief tonight." System 1 also degraded (flag in System 0) |
| No meetings tomorrow | Deliver: "Clear day tomorrow — full focus time available ✅" |
| Calendar API returns partial data | Compare event count with known typical range. If suspiciously low, alert: "Calendar returned only [N] events — possible API issue" |

### Measurement
- **Accuracy:** Target 100%. Calendar data is deterministic — errors are bugs, not judgment calls. Every flag must be factually correct.
- **Usefulness:** Weekly ask: "Did you act on any calendar suggestion this week?" Track ratio.
- **Conflict catch rate:** Any conflict Steven found himself = critical failure. Target: 0 misses.
- **Phase gate:** 20 consecutive daily briefs with 0 factual errors → propose Phase 2. Phase 3 requires separate approval.

### Cost Budget
- Calendar API: 1-2 calls ≈ $0
- LLM (formatting): ~2K tokens in, ~1K out
- **Estimated cost per run: ~$0.02 or $0 in main session**

---

## Weekend & Holiday Behavior

| System | Weekends | Holidays |
|--------|----------|----------|
| Agendas (1) | Run if meetings exist. Skip with "No meetings" if calendar empty. | Same as weekends |
| Slack Screening (2) | Skip Saturday. Run Sunday evening (catch-up for Monday). | Skip, run evening before return |
| Follow-up (3) | Monday only — no change | If Monday holiday, deliver Tuesday |
| Marketing (4) | Monday only — no change | If Monday holiday, deliver Tuesday |
| Calendar (5) | Run if next day has meetings. Skip with "clear day" if empty. | Same as weekends |
| Heartbeat (0) | Run daily — reduced output expected | Run daily |

Steven OOO Detection:
OOO is detected by ANY of these signals (case-insensitive, substring match):
- Calendar event title contains: "ooo", "out of office", "vacation", "pto", "personal day", "offsite", "time off", "day off", "leave"
- Calendar event spans full day (all-day event) AND Steven is the only attendee
- Calendar event has "outOfOffice" event type (Google Calendar native OOO)
- Steven's Slack status contains "ooo", "vacation", "off", or 🏖️/✈️/🌴 emoji

If OOO detected for multiple days → pause Systems 1-2 for that period. System 5 still runs (Steven may want to see what's piling up when he returns). System 3 still runs Monday (accountability doesn't pause). System 0 reports: "Steven OOO detected [dates] — Systems 1-2 paused."

False positive handling: If Steven sends a Telegram message during detected OOO → cancel the OOO override, resume all systems, log: "OOO cancelled — Steven active."

---

## Post-Phase 3: Evolution Path

Phase 3 is not the end state. Each system can evolve:

| System | Phase 3 State | Phase 4 (future, requires new approval) |
|--------|---------------|----------------------------------------|
| Agendas | Auto-deliver, real-time pre-meeting updates | Prep doc generation (auto-create Google Doc agenda per meeting, share with attendees) |
| Slack Screening | Auto-filter, recall audits | Smart routing: auto-respond to routine asks on Steven's behalf ("Steven will review this by EOD") |
| Follow-up | Draft follow-ups for Steven approval | Autonomous follow-up DMs after proven track record (the current accountability gate) |
| Marketing | Auto-recommendations feeding batch params | Autonomous A/B test proposals, budget reallocation suggestions |
| Calendar | Auto-create focus blocks | Meeting optimization: suggest rescheduling based on energy patterns, batch similar meetings |

Phase 4 is documented for vision, not commitment. Each requires fresh Steven approval and a new spec.

---

## Rollout Schedule

### Phase 1: Manual Proof (starting March 27)

| System | First Delivery | Frequency | Phase Gate |
|--------|---------------|-----------|------------|
| Calendar (5) | March 27, 8 PM | Daily | 20 × 0 errors |
| Agendas (1) | March 27, 9 PM | Daily | 15 × 👍 |
| Slack Screening (2) | March 28, 10:15 AM | 2x daily (weekdays) | 20 × 0 missed decisions |
| Follow-up (3) | March 31, 9 AM | Weekly Monday | 8 × <5% factual error |
| Marketing (4) | April 7 (pending discovery sprint) | Weekly Monday | 4 × useful |
| Heartbeat (0) | March 27, 9:30 PM | Daily | Always on |

### Estimated Total Cost (Phase 1, all systems)
- Daily (weekdays): ~$0.22/day (Systems 0, 1, 2×2, 5)
- Weekly (Monday): +$0.06 (System 3)
- **Monthly estimate: ~$5.50** (assuming 22 weekdays)
- $0 if run in main Opus session instead of sub-agents

---

## Dependencies & Blockers

| Dependency | Status | Blocks | Remediation |
|------------|--------|--------|-------------|
| Calendar API (gog) | ✅ Working | — | — |
| Fireflies (via Drive extractions) | ✅ Working (daily cron cc7c7c02, last run 2026-03-26) | Agendas, Follow-up | — |
| Slack API | ✅ Working | Screening | — |
| Action Tracker Sheet | ✅ Working | Follow-up, Agendas | — |
| Google Analytics API | ⚠️ Needs verification | Marketing Loop | Discovery sprint Day 1 (April 7) |
| Aaron's ad data | ❌ Not available | Marketing Loop (full) | **Steven to ask Aaron by April 7** |
| Team roster freshness | ✅ Refreshed 2026-03-26 | All (minor) | `scripts/refresh-team-roster.sh` BUILT + RAN. 29 members, 10 have Slack IDs, 19 missing. Auto-runs when System 0 detects >14 days stale. |
| Fireflies data source | ✅ Uses Google Drive pipeline (not API) | — | Existing cron `cc7c7c02` extracts daily. No API key needed. |
