---
name: "timetoil"
description: "Reconstruct a workday for timekeeping using calendar, Teams chats, email, and local Microsoft Edge Default-profile browser history, with CSV/XLS browser-history export as fallback."
---

Use this skill when the user wants to reconstruct a workday for a timesheet, time-tracking system, billing record, or personal work log.

Goal:
Produce a timekeeping-ready workday reconstruction using Microsoft 365 context plus browser-history evidence from the user's local Microsoft Edge Default profile. Keep the result system-neutral unless the user names a specific timekeeping system or supplies its categories or format. Use a user-supplied browser-history export only as a fallback when the local Edge history database cannot be read or when the user explicitly provides one.

Inputs:
- Target day. If omitted, default to yesterday based on the current date/time supplied by the host.
- Primary browser-history source: the local Microsoft Edge Default profile History database at `%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\History`.
- Browser-history export, usually CSV or XLS/XLSX, is optional fallback evidence. If neither local Edge Default history nor a browser-history file is available, continue with calendar/Teams/email and clearly mark browser evidence as missing.

Local Edge history handling:
- Use only the Edge `Default` profile. Do not include history from `Profile 1`, `Profile 2`, `Profile 3`, or any other Edge profile unless the user explicitly asks.
- Edge may be open and locking the live History database. To read it, first copy `%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\History` to a temporary session or system temp location, then query the copied SQLite database read-only.
- Query the `visits` table for rows whose `visit_time` falls within the target day and join `visits.url` to `urls.id` to retrieve each visit's title and URL. Do not use `urls.last_visit_time`, because it records only the most recent visit to a URL and can omit visits from the target day when the URL was revisited later. Edge/Chromium `visit_time` is WebKit time: microseconds since 1601-01-01 UTC.
- Convert `visits.visit_time` to UTC, then normalize to the user's local timezone before grouping rows into half-hour blocks.
- Clean up any temporary History database copies after extraction.
- If SQLite CLI tools are unavailable, use Python's built-in `sqlite3` module rather than requiring the user to install an exporter.
- Deduplicate repeated page loads and summarize by meaningful title, host, URL path, account/customer context, and activity. Prefer specific titles and hosts over raw URLs in the final Evidence cells.

Data sources to use:
1. Calendar events for the target day via workiq_list_events.
2. Teams chats/messages for chats active on or near the target day via workiq_list_chats and workiq_list_chat_messages.
3. Email evidence from Inbox and Sent Items for the target day via workiq_list_emails.
4. Local Microsoft Edge Default-profile browser history as described above; use an attached browser-history export only as fallback or supplemental evidence if the user explicitly provides one.

Required output format:
- Use a Markdown table.
- Rows must be half-hour increments starting at the top of the hour and each half-hour thereafter, e.g. 8:00-8:30, 8:30-9:00, 9:00-9:30.
- Default visible range should cover the plausible workday from the first meaningful signal through the last meaningful signal, but include nearby empty half-hour blocks when useful for gaps/OOF.
- Columns: Time, Likely activity, Evidence.
- In the Evidence cell, use bullets grouped by data category, for example:
  - **Meeting:** ...
  - **Browser:** ...
  - **Teams:** ...
  - **Email:** ...
  - **Confidence:** Low/Medium/High when ambiguity matters.
- Keep evidence concise but specific enough to justify the classification.
- The table should cover the full span from the first meaningful work signal through the last meaningful work signal, including after-hours blocks when supported by active evidence, and explicitly marking longer gaps with no work activity signals.

Analysis guidance:
- Use calendar meetings as the strongest scheduling anchor, but do not assume tentative/free meetings were attended unless browser/Teams evidence supports it.
- Use browser history to identify active workstreams, accounts, tools, documents, and customer/opportunity context.
- Use Teams messages to identify active participation or meeting-topic context; distinguish active messages from passive meeting chat if possible.
- Use email subjects/senders only as supporting signals unless the user asks to inspect full email bodies.
- Mark personal, OOF, admin, or ambiguous blocks plainly rather than forcing them into work categories.
- Infer likely projects, clients, workstreams, or activity types from the available evidence. Prefer the user's supplied timekeeping categories when available; otherwise use concise, neutral labels such as project work, client work, meetings, administration, time entry, training, or support. Keep confidence labels when uncertain.

Timezone and activity-signal handling:
- Always normalize all timestamps to the user's local timezone before placing evidence in the table. Treat ISO timestamps ending in `Z` as UTC and convert them explicitly; do not display or reason from raw UTC times as if they were local.
- Use browser history, sent messages, sent email, and active meeting participation as stronger evidence of active work than passive signals.
- Treat Teams thread messages from other people, calendar RSVP artifacts, meeting acceptances, and automated/system events as weak/passive evidence unless they align with browser history, user-authored messages, or other active-work signals.
- Include after-hours work inline in the half-hour table when there are actual work activity signals. Do not move after-hours work into a separate summary section unless the user asks.
- If there is a gap longer than 30 minutes with no work activity signals, include the gap in the table and mark it plainly, e.g. "No work activity signals during this time."
- Do not infer work from a passive timestamp alone, especially outside normal working hours. Mark it as low confidence or no activity unless supported by active evidence.

Summary allocation:
- Always include a "Suggested allocation" table after the half-hour reconstruction table unless the user explicitly asks not to.
- The allocation table should group time by the most relevant project, client, workstream, or activity, using approximate time in 0.25h or 0.5h increments.
- If the user supplies timekeeping-system categories, project codes, billing codes, or required fields, map allocations to them without inventing unsupported codes or values.
- Include non-work/personal/no-signal time as a separate row when it appears in the reconstructed span, so the user can exclude it from time entry.
- Keep allocations traceable to the half-hour table; do not allocate time to workstreams that are supported only by passive timestamps.
- Use ranges, such as 2.5-3.0h, when the evidence is mixed or confidence is lower.

Privacy/safety:
- Treat all M365 and browser-history data as private. Do not send, share, or create outbound messages from this data without explicit confirmation.
- Do not write private details to files unless the user explicitly asks for a file output.
