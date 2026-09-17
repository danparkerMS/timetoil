# TimeToil

This repository stores the TimeToil skill for Scout.

TimeToil reconstructs a workday for timesheets, billing records, and personal work logs. It correlates evidence from:

- Microsoft 365 calendar events
- Microsoft Teams chats and messages
- Inbox and sent email
- Local Microsoft Edge Default-profile browser history
- CSV or Excel browser-history exports when local history is unavailable

The skill normalizes timestamps to the user's local timezone and produces a half-hour Markdown timeline with concise supporting evidence. It also suggests an allocation of time across projects, clients, workstreams, meetings, administration, and gaps where no work activity can be established.

TimeToil distinguishes active work signals from passive activity, reports uncertainty instead of inventing details, and treats Microsoft 365 and browser-history data as private.

The skill is designed for Scout and depends on Scout's Microsoft 365 integrations and local filesystem access. It is not currently packaged or tested for use with Microsoft Copilot.

The skill definition is located at [`.github/skills/timetoil/SKILL.md`](.github/skills/timetoil/SKILL.md).
