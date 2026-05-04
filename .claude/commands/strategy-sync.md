---
description: Sync Strategy.md todos to Google Calendar — sweep all Plan events (past + future), move strikethroughs to Completed, schedule remaining todos around existing events
---

You are executing the Strategy.md Sync workflow. Read `docs/Strategy.md` and follow its "Todo Management" instructions exactly. Use these defaults — do NOT ask clarifying questions, except for the three retrospective questions in step 3 below:

**Defaults**
- Account: `personal` (Google Calendar MCP, gerald.tcd@gmail.com)
- Calendar: `primary`
- Timezone: `Asia/Hong_Kong`
- Working window: 10:30–24:00 HKT
- Event prefix: `Plan: ` (e.g., `Plan: Study Fa`)
- Past `Plan` sweep lookback: 1 day
- Future `Plan` sweep lookahead: 30 days (so re-running the sync is idempotent)
- Ignore the "My Daily Routine" section in Strategy.md

**Steps (execute sequentially, in parallel where independent)**

1. **Pull latest from origin.** Run `git pull --ff-only` from the repo root before doing anything else. If the pull fails (non-fast-forward, conflicts, dirty working tree blocking it), stop and report to the user — do not attempt to merge, rebase, or stash. The user resolves and re-runs.

2. **Read Strategy.md** at `docs/Strategy.md`.

3. **Move strikethroughs to the Completed table.** Find any todo lines with `~~strikethrough~~` markdown in the "Todo Management" list. For each strikethrough item, in file order:
   - **Ask the user these three retrospective questions** (batch them in a single message, one item at a time so answers don't get crossed):
     1. Tell me what was actually achieved
     2. Tell me what you learned
     3. How long did it really take?
   - Wait for the user's answers before moving on to the next strikethrough item.
   - Append the item as a new row to the single `Completed:` markdown table with columns: `Todo | OKR.KR | What was Achieved | What I Learned | Estimate (min) | Actual (min)`. Pull the estimate from the `~ N ~` portion of the original todo line; pull the actual from question 3 (convert to minutes).
   - Remove the line from the active todo list.
   - If there are no strikethroughs, skip and note "no completed items to move".
   - Use the Edit tool to update Strategy.md (one Edit per row append is fine, or rewrite the whole table block in one Edit).

4. **Sweep all `Plan` events** (parallel with step 5):
   - Call `mcp__google-calendar__get-current-time` (account: personal, tz: Asia/Hong_Kong) to anchor "now".
   - Call `mcp__google-calendar__search-events` with query `Plan`, timeMin = (now - 1 day), timeMax = (now + 30 days).
   - For each result whose `summary` starts with `Plan` (case-insensitive — match `Plan:` and `plan:`): call `mcp__google-calendar__delete-event` with `sendUpdates: "none"`. Sweep BOTH past AND future events so re-running the sync is idempotent and never produces duplicates.
   - Report which were deleted (or "none").

5. **List today + upcoming events** (parallel with step 4) — **MANDATORY every run, no exceptions**:
   - Call `mcp__google-calendar__list-events` for primary, timeMin = today 00:00 HKT, timeMax = today + 7 days, to identify existing fixed events (non-`Plan` events) to schedule around.
   - **You MUST call `list-events` fresh on every invocation, even if you already called it earlier in the same session.** Never reuse a previous list-events result. New fixed events may have been added between runs, and stale data will cause `Plan:` events to overlap real commitments. Re-fetching is cheap; reusing is not.
   - Treat the recurring "Csl Meeting" as a 30-minute slot at its start time only — its data has a multi-year end which is a known anomaly; ignore the long end.

6. **Build the schedule.** Parse the active todo list. Each line is `N. <todo> ~ <minutes> ~ <OKR.KR>` with optional sub-bullets. Schedule todos sequentially in file order:
   - Start at `max(now, today 10:30 HKT)` rounded up to the next 30-min boundary.
   - Only place todos within 10:30–24:00 HKT.
   - Skip over existing fixed (non-`Plan`) events — never overlap.
   - **Split todos that don't fit a single contiguous slot.** When the next available gap (between fixed events, the working-window edges, or end of day) is smaller than the todo's remaining minutes, place a chunk that fills the gap and carry the leftover minutes into the next available slot. Continue chunking across slots and days until the full duration is scheduled.
     - **Minimum chunk size: 30 minutes.** If an available gap is shorter than 30 min, skip it entirely (don't create a sliver).
     - **Track chunks per todo.** Number each chunk in file order: `(1/N)`, `(2/N)`, …, where `N` is the total number of chunks created for that todo. If a todo fits in one slot, no chunk suffix is added.
   - Use as many days as needed.

7. **Create events** with `mcp__google-calendar__create-event` (one call per event — this server has no bulk variant):
   - `summary`: `Plan: <todo text>` — for split todos, append ` (i/N)` (e.g., `Plan: Set up wiki (2/3)`).
   - `description`: `<OKR.KR>` on the first line; if there are sub-bullets in the source, append them as `- <bullet>` lines.
   - `start` / `end`: ISO 8601 local time, no offset.
   - `colorId`: `"2"` (Sage) — all `Plan:` events created by this sync must be sage-coloured.
   - account: `personal`, calendarId: `primary`, timeZone: `Asia/Hong_Kong`.

8. **Report** a final markdown table: Time | Event | OKR. Include both pre-existing fixed events and newly created `Plan:` events for the day(s) scheduled.

9. **Commit and push Strategy.md changes.** If `git status --porcelain docs/Strategy.md` shows the file was modified by step 3:
   - `git add docs/Strategy.md`
   - `git commit -m "Sync strategy: move completed todos"` (use a more specific message if step 3 moved a single named item)
   - `git push`
   - If push fails (rejected, network), report the error and stop — do not force-push or rewrite history. The user resolves manually.
   - If Strategy.md was not modified (no strikethroughs this run), skip — nothing to commit.

**Failure modes to handle silently**
- MCP auth expired → tell the user to run `/mcp` to reconnect, then stop.
- No active todos → report "no todos to schedule" and skip steps 6–7.
- No strikethroughs and no `Plan` events to sweep → just proceed; this is normal.
