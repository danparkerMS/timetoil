# TimeToil

This repository stores two variants of TimeToil:

| Variant | Skill | Purpose |
| --- | --- | --- |
| Scout reference | [`timetoil`](.github/skills/timetoil/SKILL.md) | Original Scout workflow, preserved unchanged as a reference. |
| GitHub Copilot | [`timetoil-copilot`](.github/skills/timetoil-copilot/SKILL.md) | Daily reconstruction adapted for the GitHub Copilot app's available tools, permissions, and session context. |

TimeToil reconstructs a workday for timesheets, billing records, and personal work logs. It correlates evidence from:

- Microsoft 365 calendar events
- Microsoft Teams chats and messages
- Inbox and sent email
- Local Microsoft Edge and other browser history, providing control for which profiles are included
- CSV or Excel browser-history exports when local history is unavailable

Both variants normalize timestamps to the user's timezone and produce a Markdown timeline with concise supporting evidence. The breakdown is configurable (15 minutes, 30 minutes, or 1 hour; default 30 minutes), as is the confidence threshold used for suggested allocations. Allocations cover projects, clients, workstreams, meetings, administration, and explicit gaps/uncertainty without double-counting elapsed time.

TimeToil distinguishes active work signals from passive activity, reports uncertainty instead of inventing details, and treats Microsoft 365 and browser-history data as private.

## Using the Copilot variant

To include Microsoft 365 calendar, email, and Teams evidence, **enable Microsoft Work IQ in the app's Customize area before running the skill**:

1. Open app **Settings** and find the **Customize** section, then select **Plugins**. In versions with a separate **Customize** page, open that page instead.
2. Find and enable **Work IQ**. If it is not listed, add the official [`microsoft/work-iq`](https://github.com/microsoft/work-iq) plugin marketplace and install the `workiq` plugin.
3. Complete any sign-in or consent prompts using your Microsoft work account. Authentication may be requested on the first Microsoft 365 read rather than when the plugin is enabled. Appropriate licensing, tenant policy, and administrator consent may be required.
4. Start a new chat session in this repository so the enabled tools load, then invoke the skill. Enabling the plugin alone does not confirm that all sources are accessible; the skill checks available capabilities and reports missing access.

Open this repository in the GitHub Copilot app with repository skills enabled/available, then explicitly name the variant to avoid ambiguity with the Scout skill:

> Use the timetoil-copilot skill to reconstruct yesterday's workday for my time log.

You can supply a date, timezone, confidence threshold, increment, and timekeeping categories in the same request. The skill asks for missing preferences and permission before optional local evidence collection. Browser profiles are selected explicitly; no profile is read by default. Preferences are reused within the conversation, not assumed to persist across sessions. Durable storage requires an available mechanism and an explicit user request.

The skill follows [Copilot's repository Agent Skills conventions](https://docs.github.com/en/copilot/concepts/customizations/agent-skills): a separate `.github/skills/timetoil-copilot/SKILL.md` with `name` and `description` frontmatter. Skill availability depends on the Copilot surface/version and repository customization settings; adding this file does not configure integrations or grant access.

## Prerequisites and limitations

- **Microsoft 365:** Calendar, Teams, email, and optional transcripts require an already configured integration exposing the relevant read capabilities. Scout's named WorkIQ tools are not assumed to exist in Copilot; no M365 integration is bundled here. The Copilot skill discovers actual tools and reports missing, authentication-blocked, or partial sources. Use the app's integration sign-in/settings when authentication is required, or provide scoped excerpts/exports.
- **Local evidence:** Selected Edge/Chrome history and approved VS Code/Git/PowerShell sources require filesystem access on the machine containing them. Windows paths are documented in the skill; remote/cloud sessions cannot assume desktop access. Local sources are optional, and declining them is supported.
- **History extraction:** A consistent SQLite snapshot and a SQLite reader (for example Python's built-in `sqlite3`) are needed for local Chromium history. The Copilot variant queries individual visits, not just each page's latest visit, and does not treat an inconsistent copy of an open database as complete. User-provided CSV/XLS/XLSX exports are fallback/supplemental sources; spreadsheet reading requires an available reader.
- **Evidence limits:** Invitations do not prove attendance, visits do not prove duration, and untimestamped shell commands cannot establish when work occurred. Missing sources and ambiguous blocks remain visible; reduced-source reconstructions are possible but not complete activity records.
- **Privacy and scope:** No real activity is collected by installing the skill. Runs remain read-only and chat-first, with no automatic submission, scheduling, connector installation, outbound sharing, or private-data files unless explicitly requested. Necessary temporary browser snapshots are scoped and cleaned up.

The Scout reference retains its original integration and memory assumptions. The Copilot variant has been checked structurally, but live M365/local-evidence behavior requires the relevant integrations, permissions, and user-approved sources; adding instructions alone is not an end-to-end integration test.

## Keeping the variants aligned

Review changes to shared behavior against both skills: daily timeline shape, confidence rules, timezone normalization, allocation traceability, gaps, and privacy. Note intentional platform differences in the change description. Keep adapters distinct: Scout tool names/memory versus Copilot capability discovery, session preferences, permission scope, and evidence-reader safeguards.

The Scout reference is unchanged by this adaptation. Future shared-behavior changes should be considered for both variants through explicit review, not copied blindly over either adapter. There is no automatic synchronization infrastructure.
