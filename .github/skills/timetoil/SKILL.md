---
name: "timetoil"
description: "Reconstruct a workday for timekeeping using calendar, Teams chats, email, and selected local Edge/Chrome browser profiles, with CSV/XLS browser-history export as fallback."
---

Use this skill when the user wants to reconstruct a workday for a timesheet, time-tracking system, billing record, or personal work log.

Goal:
Produce a timekeeping-ready workday reconstruction using Microsoft 365 context plus browser-history evidence from the user's selected local Microsoft Edge and/or Google Chrome profiles. Keep the result system-neutral unless the user names a specific timekeeping system or supplies its categories or format. Use a user-supplied browser-history export only as a fallback when the selected local browser history databases cannot be read or when the user explicitly provides one.

First-run preferences:
- Before collecting evidence, check memory for existing TimeToil preferences covering both:
  1. confidence threshold to include in results and allocations; and
  2. browser/profile selection for local browsing evidence.
- If either preference is missing, ask the user once at the start of the run and then remember the answer for future TimeToil runs.
- Only ask for these durable preferences in this first-run preferences prompt. Do not include target dates in the same prompt.
- The first-run preferences prompt must not block the user from accepting defaults. If the UI allows blank submissions, a blank submission means "use defaults." If the UI disables Submit for blank text, provide an explicit `Use defaults` / `Skip` choice so the user can continue without typing a value.
- For confidence threshold, offer these choices:
  - High only: include only high-confidence work blocks in allocations.
  - Medium and high: include medium- and high-confidence work blocks in allocations. This is the default if the user skips or gives no preference.
  - Low, medium, and high: include all supported work blocks, while still labeling uncertainty.
- For browser/profile selection, discover available Chromium profiles before asking. Inspect profile directories under:
  - Edge: `%LOCALAPPDATA%\Microsoft\Edge\User Data\`
  - Chrome: `%LOCALAPPDATA%\Google\Chrome\User Data\`
- Candidate profile directories include `Default`, `Profile 1`, `Profile 2`, and other directories containing a `History` database. Read friendly profile names from each browser's `Local State` file when available (`profile.info_cache.<profileDir>.name`), and present choices as `Browser / Friendly Name (profileDir)`. If a friendly name is unavailable, use the profile directory name.
- Allow the user to select one or more profiles across Edge and Chrome. If discovery fails or the user skips profile selection, default to `Edge / Default` only.
- Store remembered preferences concisely, for example: `TimeToil confidence threshold: Medium and high` and `TimeToil browser profiles: Edge Default; Chrome Profile 2 (Work)`.
- If the user explicitly supplies a confidence level or browser/profile override in a later run, use that override for the current run and update memory when it appears to be a durable preference.

Prompt sequence:
1. Resolve first-run preferences before collecting evidence. If confidence threshold or browser/profile selection is missing from memory, ask for those preference values only.
2. After first-run preferences have been resolved or confirmed from memory, ask for the target date or date range in a separate prompt unless the user already supplied it in the current request.
3. Do not combine target dates with confidence threshold or browser/profile selection. Target dates are per-run inputs; confidence threshold and browser/profile selection are durable preferences.
4. Every prompt in this sequence must provide a no-typing path to accept defaults. Prefer accepting an empty submission when the UI supports it; otherwise include an explicit default/skip option that enables Submit.

Inputs:
- Target dates:
	- Present the user with a separate input field with tool tips example for accepted values (today, this work week, September 30th, etc.).
	- Do not rely on a blank input field if the UI keeps Submit disabled while the field is empty. In that case, include an explicit `Use yesterday` / `Default` option or tell the user they may type `default`.
	- If omitted, submitted blank, or answered with `default`, default to yesterday based on the current date/time supplied by the host.
- Primary browser-history source: the selected local Edge and/or Chrome Chromium profile History databases.
- Browser-history export, usually CSV or XLS/XLSX, is optional fallback or supplemental evidence. If neither selected local browser history nor a browser-history file is available, continue with calendar/Teams/email and clearly mark browser evidence as missing.

Local Chromium browser history handling:
- Use the remembered or user-selected Edge and Chrome profiles. Do not include any profile that was not selected unless the user explicitly asks.
- Browsers may be open and locking live History databases. To read a profile, copy its `History` file to a temporary session or system temp location, then query the copied SQLite database read-only.
- Query the `urls` table for rows whose `last_visit_time` falls within the target day. Chromium `last_visit_time` is WebKit time: microseconds since 1601-01-01 UTC.
- Convert `last_visit_time` to UTC, then normalize to the user's local timezone before grouping rows into half-hour blocks.
- Tag each browser-history item with browser and profile, e.g. `Edge / Default`, `Chrome / Work (Profile 2)`.
- Merge browser history across all selected profiles, deduplicate repeated page loads across profiles where appropriate, and summarize by meaningful title, host, URL path, account/customer context, and activity. Prefer specific titles and hosts over raw URLs in the final Evidence cells.
- Clean up any temporary History database copies after extraction.
- If SQLite CLI tools are unavailable, use Python's built-in `sqlite3` module rather than requiring the user to install an exporter.

Data sources to use:
1. Calendar events for the target day via workiq_list_events.
2. Teams chats/messages for chats active on or near the target day via workiq_list_chats and workiq_list_chat_messages.
3. Email evidence from Inbox and Sent Items for the target day via workiq_list_emails.
4. Local browser history from the selected Edge and/or Chrome profiles as described above; use an attached browser-history export only as fallback or supplemental evidence if the user explicitly provides one.

Required output format:
- Use a Markdown table.
- Rows must be half-hour increments starting at the top of the hour and each half-hour thereafter, e.g. 8:00-8:30, 8:30-9:00, 9:00-9:30.
- Default visible range should cover the plausible workday from the first meaningful signal through the last meaningful signal, but include nearby empty half-hour blocks when useful for gaps/OOF.
- Columns: Time, Likely activity, Confidence, Evidence.
- Every row must have a Confidence value: High, Medium, Low, or None.
- In the Evidence cell, use bullets grouped by data category, for example:
  - **Meeting:** ...
  - **Browser:** `Chrome / Work`: ...
  - **Teams:** ...
  - **Email:** ...
- Keep evidence concise but specific enough to justify the classification.
- The table should cover the full span from the first meaningful work signal through the last meaningful work signal, including after-hours blocks when supported by active evidence, and explicitly marking longer gaps with no work activity signals.
- State the active confidence threshold and browser profiles used before the table.

Confidence handling:
- Assign confidence per half-hour row based on the strength and agreement of evidence.
- High confidence usually requires direct active-work evidence such as browser activity in work tools/documents, sent Teams messages, sent email, or confirmed meeting attendance signals aligned to the activity.
- Medium confidence usually has a plausible calendar or passive-context anchor plus some supporting evidence, but not enough direct active-work evidence.
- Low confidence is ambiguous, weak, passive, or inferred activity.
- None means no work activity signals.
- Apply the user's confidence threshold to the Suggested allocation table. Do not allocate blocks below the selected threshold to work categories; instead include them as `Below selected confidence threshold` or `No work activity signals` so the user can exclude or review them.
- Still show lower-confidence rows in the half-hour reconstruction when they fall inside the reconstructed span, but label them clearly.

Analysis guidance:
- Use calendar meetings as the strongest scheduling anchor, but do not assume tentative/free meetings were attended unless browser/Teams evidence supports it.
- Use browser history to identify active workstreams, accounts, tools, documents, and customer/opportunity context.
- Use Teams messages to identify active participation or meeting-topic context; distinguish active messages from passive meeting chat if possible.
- Use email subjects/senders only as supporting signals unless the user asks to inspect full email bodies.
- Mark personal, OOF, admin, below-threshold, or ambiguous blocks plainly rather than forcing them into work categories.
- Infer likely projects, clients, workstreams, or activity types from the available evidence. Prefer the user's supplied timekeeping categories when available; otherwise use concise, neutral labels such as project work, client work, meetings, administration, time entry, training, or support. Keep confidence labels on all rows.

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
- Apply the selected confidence threshold to allocations. For example, if the threshold is `Medium and high`, exclude Low-confidence work blocks from work allocations and list them separately as below-threshold review time.
- If the user supplies timekeeping-system categories, project codes, billing codes, or required fields, map allocations to them without inventing unsupported codes or values.
- Include non-work/personal/no-signal time as a separate row when it appears in the reconstructed span, so the user can exclude it from time entry.
- Keep allocations traceable to the half-hour table; do not allocate time to workstreams that are supported only by passive timestamps below the user's threshold.
- Use ranges, such as 2.5-3.0h, when the evidence is mixed or confidence is lower.

Privacy/safety:
- Treat all M365 and browser-history data as private. Do not send, share, or create outbound messages from this data without explicit confirmation.
- Do not write private details to files unless the user explicitly asks for a file output.
- Redact any PII or sensitive information from the output unless the user explicitly asks to include it.
