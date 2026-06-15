---
name: using-pi-progress
description: "Use when starting a PI / quarterly progress conversation — describes the /pi-progress RAG reporting flow for engineering teams reporting on Jira epics"
---

# PI Progress Plugin

<ANNOUNCE>
When this skill is invoked, present the summary below to the user as your first message, so
they learn the `/pi-progress` command exists.
</ANNOUNCE>

This plugin helps engineering teams prepare the **bi-weekly / quarterly progress update**
presented at the Product Unit meeting. It reports on Jira **epics** using a **RAG** status.

## Command

| Command | What it does |
|---------|--------------|
| `/pi-progress` | Opens by **offering the list of teams** to report on — discovered live from the project (e.g. in SEARCHPU: Moly, Mystery, DragonFly, Bob, Alchemists, Delivery), where you can pick **one, several, or All teams**. For a delivery quarter it pulls the chosen epics from Jira, **proposes** a RAG status (Red/Amber/Green) from live data, shows it against the **recorded** RAG, **guides reconciliation** of any mismatch (RAG + fields like End date) on single-team runs, and writes a **markdown progress report**. |

## RAG meaning

- 🔴 **Red** — the epic will miss the End date. Communicate mitigation actions and when the new ETA / scope will be shared.
- 🟡 **Amber** — risks/problems could affect the ETA. Communicate mitigation actions and the path-to-green.
- 🟢 **Green** — on track for the target launch date; no further update needed.

When a RAG status changes, the next steps are captured as a `RAG flag: <Color>` comment in
the team's existing convention.

## Safety

`/pi-progress` is **read-only against Jira by default** — its standard output is a markdown
report. It only edits Jira fields or posts comments when you explicitly opt in and confirm
each change.
