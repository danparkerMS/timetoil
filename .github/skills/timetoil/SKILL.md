---
name: "timetoil"
description: "Reconstruct a workday for timekeeping using calendar, Teams chats, email, selected local Edge/Chrome browser profiles, and local VS Code/Git/PowerShell activity anchors, with CSV/XLS browser-history export as fallback."
---

Use this skill when the user wants to reconstruct a workday for a timesheet, time-tracking system, billing record, or personal work log.

Goal:
Produce a timekeeping-ready workday reconstruction using Microsoft 365 context, browser-history evidence from the user's selected local Microsoft Edge and/or Google Chrome profiles, and local activity anchors from VS Code/Git and PowerShell where available. Keep the result system-neutral unless the user names a specific timekeeping system or supplies its categories or format. Use a user-supplied browser-history export only as a fallback when the selected local browser history databases cannot be read or when the user explicitly provides one.

First-run preferences:
- Before collecting evidence, check memory for existing TimeToil preferences covering all three:
  1. confidence threshold to include in results and allocations; and
  2. browser/profile selection for local browsing evidence; and
  3. time increment for the daily breakdown.
- If any preference is missing, ask the user once at the start of the run and then remember the answer for future TimeToil runs.
- For confidence threshold, offer these choices:
  - High only: include only high-confidence work blocks in allocations.
  - Medium and high: include medium- and high-confidence work blocks in allocations. This is the default if the user skips or gives no preference.
  - Low, medium, and high: include all supported work blocks, while still labeling uncertainty.
- For browser/profile selection, discover available Chromium profiles before asking. Inspect profile directories under:
  - Edge: `%LOCALAPPDATA%\Microsoft\Edge\User Data\`
  - Chrome: `%LOCALAPPDATA%\Google\Chrome\User Data\`
- Candidate profile directories include `Default`, `Profile 1`, `Profile 2`, and other directories containing a `History` database. Read friendly profile names from each browser's `Local State` file when available (`profile.info_cache.<profileDir>.name`), and present choices as `Browser / Friendly Name (profileDir)`. If a friendly name is unavailable, use the profile directory name.
- Allow the user to select one or more profiles across Edge and Chrome. If discovery fails or the user skips profile selection, default to `Edge / Default` only.
- For the daily breakdown increment, offer these choices:
  - 15 minutes
  - 30 minutes
  - 1 hour
- If the user skips the increment question or gives no preference, default to 30 minutes.
- Store remembered preferences concisely, for example: `TimeToil confidence threshold: Medium and high`, `TimeToil browser profiles: Edge Default; Chrome Profile 2 (Work)`, and `TimeToil breakdown increment: 30 minutes`.
- If the user explicitly supplies a confidence level, browser/profile, or breakdown-increment override in a later run, use that override for the current run and update memory when it appears to be a durable preference.

Inputs:
- Target day. If omitted, default to yesterday based on the current date/time supplied by the host.
- Primary browser-history source: the selected local Edge and/or Chrome Chromium profile History databases.
- Local activity-anchor sources: VS Code logs/state timestamps and PowerShell PSReadLine command history where available.
- Browser-history export, usually CSV or XLS/XLSX, is optional fallback or supplemental evidence. If neither selected local browser history nor a browser-history file is available, continue with calendar/Teams/email and clearly mark browser evidence as missing.

Local Chromium browser history handling:
- Use the remembered or user-selected Edge and Chrome profiles. Do not include any profile that was not selected unless the user explicitly asks.
- Browsers may be open and locking live History databases. To read a profile, copy its `History` file to a temporary session or system temp location, then query the copied SQLite database read-only.
- Query the `urls` table for rows whose `last_visit_time` falls within the target day. Chromium `last_visit_time` is WebKit time: microseconds since 1601-01-01 UTC.
- Convert `last_visit_time` to UTC, then normalize to the user's local timezone before grouping rows into the selected breakdown increment.
- Tag each browser-history item with browser and profile, e.g. `Edge / Default`, `Chrome / Work (Profile 2)`.
- Merge browser history across all selected profiles, deduplicate repeated page loads across profiles where appropriate, and summarize by meaningful title, host, URL path, account/customer context, and activity. Prefer specific titles and hosts over raw URLs in the final Evidence cells.
- Clean up any temporary History database copies after extraction.
- If SQLite CLI tools are unavailable, use Python's built-in `sqlite3` module rather than requiring the user to install an exporter.

Local VS Code/Git and PowerShell activity anchors:
- Use local activity anchors to strengthen or bound coding-heavy work blocks, especially when browser history is sparse or the user mentions VS Code, Git, terminal, shell, repository, branch, commit, merge, rebase, or command-line work.
- Inspect VS Code logs under `%APPDATA%\Code\logs\`, especially `window*\exthost\vscode.git\Git.log`, for timestamped Git operations such as `git status`, `git fetch`, `git show`, `git blame`, `git diff`, branch/ref scans, repository opens, and file paths. These are strong evidence that VS Code was active in a specific repository or file context when they align with other signals.
- Inspect VS Code state and workspace timestamps under `%APPDATA%\Code\User\globalStorage\` and `%APPDATA%\Code\User\workspaceStorage\` to identify active VS Code windows/workspaces and last-update boundaries. Treat state-file LastWriteTime as an activity boundary, not as proof of a specific task unless corroborated by Git logs, terminal history, browser history, or M365 signals.
- Inspect PowerShell PSReadLine history at `%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` and, when relevant, shell histories such as `%USERPROFILE%\.bash_history` or `%USERPROFILE%\.zsh_history`. Use the history file LastWriteTime as a terminal activity boundary and the recent commands as task context. Do not assume every listed command ran during the target window unless the file timestamp or other evidence supports it.
- Prefer timestamped VS Code Git log entries over untimestamped shell-history lines for precise placement. Use shell-history content to explain what the user was likely doing, and use the history file LastWriteTime plus surrounding evidence to bound when it happened.
- Treat VS Code/Git and PowerShell anchors as active-work evidence when they show repository interaction, file navigation, diffs, branch operations, commits, fetches, tests, builds, or meaningful commands. Treat automatic background polling, periodic fetches, extension startup noise, and state-file writes as medium or low confidence unless corroborated.
- Summarize these anchors in Evidence cells as `**VS Code/Git:** ...` and `**Terminal:** ...`, naming repositories, branches, files, or command themes when available without overloading the table.
- For after-hours work, local VS Code/Git and terminal anchors can establish a defensible "sat down and resumed work" time when they follow a no-signal gap and align with browser searches or M365 activity.

Data sources to use:
1. Calendar events for the target day via workiq_list_events.
2. Teams chats/messages for chats active on or near the target day via workiq_list_chats and workiq_list_chat_messages.
3. Email evidence from Inbox and Sent Items for the target day via workiq_list_emails.
4. Local browser history from the selected Edge and/or Chrome profiles as described above; use an attached browser-history export only as fallback or supplemental evidence if the user explicitly provides one.
5. Local VS Code/Git logs/state timestamps and PowerShell/shell history as described above.
6. For calendar-backed Teams meetings where actual attendance or early adjournment matters, use available Teams meeting metadata, chat activity, and transcripts when accessible via workiq_list_meeting_transcripts and workiq_get_meeting_transcript. Treat transcript availability as optional; do not fail the reconstruction if it is unavailable.

Required output format:
- Use a Markdown table.
- Rows must use the selected breakdown increment and align to natural clock boundaries:
  - 15 minutes: e.g. 8:00-8:15, 8:15-8:30, 8:30-8:45.
  - 30 minutes: e.g. 8:00-8:30, 8:30-9:00, 9:00-9:30.
  - 1 hour: e.g. 8:00-9:00, 9:00-10:00, 10:00-11:00.
- Default visible range should cover the plausible workday from the first meaningful signal through the last meaningful signal, but include nearby empty blocks when useful for gaps/OOF.
- Columns: Time, Likely activity, Confidence, Evidence.
- Every row must have a Confidence value: High, Medium, Low, or None.
- In the Evidence cell, use bullets grouped by data category, for example:
  - **Meeting:** ...
  - **Browser:** `Chrome / Work`: ...
  - **VS Code/Git:** ...
  - **Terminal:** ...
  - **Teams:** ...
  - **Email:** ...
- Keep evidence concise but specific enough to justify the classification.
- The table should cover the full span from the first meaningful work signal through the last meaningful work signal, including after-hours blocks when supported by active evidence, and explicitly marking longer gaps with no work activity signals.
- State the active confidence threshold, browser profiles, and breakdown increment used before the table.

Confidence handling:
- Assign confidence per breakdown row based on the strength and agreement of evidence.
- High confidence usually requires direct active-work evidence such as browser activity in work tools/documents, sent Teams messages, sent email, or confirmed meeting attendance signals aligned to the activity.
- High confidence for a meeting ending early requires strong actual-duration evidence, such as a transcript or recording ending early, explicit meeting chat indicating adjournment, or a clear shift from active meeting participation to unrelated active work immediately after the inferred end.
- Medium confidence usually has a plausible calendar or passive-context anchor plus some supporting evidence, but not enough direct active-work evidence.
- Low confidence is ambiguous, weak, passive, or inferred activity.
- None means no work activity signals.
- Apply the user's confidence threshold to the Suggested allocation table. Do not allocate blocks below the selected threshold to work categories; instead include them as `Below selected confidence threshold` or `No work activity signals` so the user can exclude or review them.
- Still show lower-confidence rows in the daily reconstruction when they fall inside the reconstructed span, but label them clearly.

Analysis guidance:
- Use calendar meetings as the strongest scheduling anchor, but do not assume tentative/free meetings were attended unless browser/Teams evidence supports it.
- Distinguish scheduled meeting duration from inferred actual meeting duration. If a calendar block is scheduled for longer than the evidence supports, split the scheduled block at the selected breakdown boundary where meeting evidence appears to stop. Label the remaining time according to subsequent active evidence, or as ambiguous/no activity if there are no signals.
- Infer early adjournment only when supported by evidence such as transcript end time, explicit Teams chat, meeting attendance metadata, or unrelated browser/email/Teams activity beginning before the scheduled end. If only the calendar event exists, keep the full scheduled meeting as a calendar anchor and mark the confidence appropriately.
- Use browser history to identify active workstreams, accounts, tools, documents, and customer/opportunity context.
- Use local VS Code/Git and PowerShell anchors to identify repository, file, branch, command, and coding-task context.
- Use Teams messages to identify active participation or meeting-topic context; distinguish active messages from passive meeting chat if possible.
- Use email subjects/senders only as supporting signals unless the user asks to inspect full email bodies.
- Mark personal, OOF, admin, below-threshold, or ambiguous blocks plainly rather than forcing them into work categories.
- Infer likely projects, clients, workstreams, or activity types from the available evidence. Prefer the user's supplied timekeeping categories when available; otherwise use concise, neutral labels such as project work, client work, meetings, administration, time entry, training, or support. Keep confidence labels on all rows.

Timezone and activity-signal handling:
- Always normalize all timestamps to the user's local timezone before placing evidence in the table. Treat ISO timestamps ending in `Z` as UTC and convert them explicitly; do not display or reason from raw UTC times as if they were local.
- Use browser history, timestamped VS Code/Git activity, terminal activity boundaries, sent messages, sent email, and active meeting participation as stronger evidence of active work than passive signals.
- Treat Teams thread messages from other people, calendar RSVP artifacts, meeting acceptances, and automated/system events as weak/passive evidence unless they align with browser history, user-authored messages, or other active-work signals.
- Include after-hours work inline in the daily reconstruction table when there are actual work activity signals. Do not move after-hours work into a separate summary section unless the user asks.
- If there is a gap longer than one selected breakdown increment with no work activity signals, include the gap in the table and mark it plainly, e.g. "No work activity signals during this time."
- Do not infer work from a passive timestamp alone, especially outside normal working hours. Mark it as low confidence or no activity unless supported by active evidence.

Summary allocation:
- Always include a "Suggested allocation" table after the daily reconstruction table unless the user explicitly asks not to.
- The allocation table should group time by the most relevant project, client, workstream, or activity, using the selected breakdown increment: 0.25h for 15 minutes, 0.5h for 30 minutes, or 1h for 1 hour.
- Apply the selected confidence threshold to allocations. For example, if the threshold is `Medium and high`, exclude Low-confidence work blocks from work allocations and list them separately as below-threshold review time.
- If the user supplies timekeeping-system categories, project codes, billing codes, or required fields, map allocations to them without inventing unsupported codes or values.
- Include non-work/personal/no-signal time as a separate row when it appears in the reconstructed span, so the user can exclude it from time entry.
- Keep allocations traceable to the daily reconstruction table; do not allocate time to workstreams that are supported only by passive timestamps below the user's threshold.
- Use ranges, such as 2.5-3.0h, when the evidence is mixed or confidence is lower.

Privacy/safety:
- Treat all M365 and browser-history data as private. Do not send, share, or create outbound messages from this data without explicit confirmation.
- Do not write private details to files unless the user explicitly asks for a file output.
