# PI Progress Plugin

A Claude Code plugin that helps engineering teams prepare the **bi-weekly / quarterly
progress update** presented at the Product Unit meeting. It reports on Jira **epics** using a
**RAG** (Red / Amber / Green) status, proposing that status from live Jira data and producing
a ready-to-present **markdown report**.

## What it does

`/pi-progress`:

0. **Opens with a quarter picker and a team picker** — both discovered from the project's
   field metadata (the `Teams` field's allowed values and the project's versions), in one fast
   call rather than scanning tickets. You can pick **one team**, **several teams**, or **All
   teams** (every epic, grouped by team). On first run it also offers a one-time read-only
   authorization so data fetches don't prompt each time.
1. Fetches the chosen team's epics from Jira for the delivery quarter (via the Atlassian MCP server).
2. **Proposes** a RAG status for each epic from real data — status, **End date**, child-issue
   progress, flags and blockers — with a one-line rationale.
3. Shows the proposal against the **recorded** RAG (`RAG` field on the epic) and flags any
   **mismatch**.
4. **Takes you by the hand** to reconcile each mismatch: what the RAG should become, why, and
   which Jira fields (e.g. **End date**) to update so the epic is self-consistent and would
   *pass* the RAG check on a re-run.
5. Writes a **markdown report** to `docs/pi-progress/<date>-<team-or-project>-<quarter>.md`. It
   **leads with Highlights** (the transitions into Amber/Red and new risks to discuss in the PI
   meeting), then a RAG summary, a **Risk register** listing every Amber/Red epic with its
   reason (signals + blockers) and follow-up action, per-epic detail, and the agreed next steps
   + drafted `RAG flag: <Color>` comment per change. **All-teams runs** add a **per-team
   freshness line** ("reviewed N of M epics this cycle") so you can see which teams refreshed
   their tickets.
6. **Optionally runs a retrospective** (`/pi-progress-retro`) so the tool learns from itself —
   feeding this run's **RAG overrides** (where you disagreed with the proposed status),
   friction, and report-quality gaps back into the rubric / skill / report template as
   **confirmed edits** (each gated by a y/n).

## RAG definitions

- 🔴 **Red** — the epic will miss the End date. Communicate mitigation actions and **when** the new ETA / scope will be shared.
- 🟡 **Amber** — risks/problems could affect the ETA. Communicate mitigation actions and the **path-to-green**.
- 🟢 **Green** — on track for the target launch date; no further update needed.

When a RAG status changes, next steps are captured as a comment whose first line is
`RAG flag: <Color>` — matching the team's existing Jira convention.

## Safety

**Read-only against Jira by default.** The standard output is the markdown report; the plugin
only edits Jira fields or posts comments when you explicitly opt in and confirm each change.
This is deliberate for the testing phase.

## Requirements

- The **Atlassian (Jira) MCP server** connected in Claude Code. Read path:
  `getAccessibleAtlassianResources`, `getJiraProjectIssueTypesMetadata` +
  `getJiraIssueTypeMetaWithFields` (team/quarter discovery from field metadata),
  `searchJiraIssuesUsingJql`, `getJiraIssue`. Opt-in write path only:
  `editJiraIssue`, `addCommentToJiraIssue`.
- On first run the plugin offers a **one-time read-only authorization** — it can add the
  read tools above (plus `Bash(jq:*)`) to `.claude/settings.local.json` so data fetches don't
  prompt each time. Write tools are never pre-authorized; they stay gated per the IRON-LAW.
- `jq` available for parsing large Jira responses (they auto-save to a file when they exceed
  the inline limit).

## Install

From GitHub (recommended for teammates):

```
/plugin marketplace add JoostVoogt-TomTom/pi-progress-plugin
/plugin install pi-progress@pi-progress
```

Or from a local checkout (for development):

```
/plugin marketplace add /path/to/pi-progress-plugin
/plugin install pi-progress@pi-progress
```

Then start a session and run `/pi-progress`.

## Contributing

Issues and pull requests welcome — clone the repo, make changes, and point a
local marketplace at your checkout to test (`/plugin marketplace add <path>` then
`/reload-plugins`). The plugin is plain markdown + shell, no build step.

## Jira field map

The RAG logic is grounded in the TomTom Jira schema (see
`skills/pi-progress/rag-rubric.md`):

| Concept | Field |
|---|---|
| RAG status | `customfield_10159` (Green/Amber/Red select) |
| End date (target launch) | `customfield_10156` (fallback: fixVersion release date) |
| Start date | `customfield_10015` |
| Team | `customfield_10150` |
| Quarter | `fixVersions[].name` (e.g. `Q26.2`) |
| Progress | child issues (`parent = <EPIC-KEY>`), % with `statusCategory = Done` |

## Layout

```
.claude-plugin/      plugin.json, marketplace.json
skills/
  pi-progress/       SKILL.md (workflow) + rag-rubric.md + report-template.md
  pi-progress-retro/ SKILL.md (self-retrospective: learns from a run, edits the files above)
  using-pi-progress/ SKILL.md (overview, auto-announced on session start)
hooks/               SessionStart announce (hooks.json + run-hook.cmd + session-start)
```
