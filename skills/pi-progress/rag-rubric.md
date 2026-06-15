# RAG Rubric & Field Map

Reference for the `/pi-progress` skill. The RAG status is proposed by reasoning over live
Jira fields against the team's definitions. The proposal is **never** auto-applied — it is
shown to the user, who confirms or overrides.

## Field map (TomTom Jira)

| Concept | Field id | Type / shape | Used for |
|---|---|---|---|
| RAG status (recorded) | `customfield_10159` | select `{value: "Green"|"Amber"|"Red", id}` | the current/previous RAG on the epic. Green = option id `14003`. |
| End date (target launch) | `customfield_10156` | date `YYYY-MM-DD` | the "End date" in the Red/Green definitions. **Primary** target date. |
| Start date | `customfield_10015` | date `YYYY-MM-DD` | pace: elapsed-time % across the Start→End window. |
| Team | `customfield_10150` | select `{value}` | scope a team's epics (matches team dashboards). |
| Quarter | `fixVersions[]` | array `{name, releaseDate}` | delivery quarter; `releaseDate` is the **fallback** End date when `customfield_10156` is empty. |
| Status | `status` | `{name, statusCategory:{name}}` | statusCategory ∈ {To Do, In Progress, Done}. |
| Flagged | `customfield_10021` | option / null | impediment flag → risk signal. |
| Blockers | `issuelinks` | array | open "is blocked by" links → risk signal. |
| Progress | child query | `parent = <EPIC-KEY>` | % done = children with `statusCategory = Done` ÷ total. |

Statuses that effectively mean delivered (treat as Done-ish for Green): `In Production`,
`In Public Preview`, `Ready for deployment`, plus any `statusCategory = Done`.

## Decision rubric

Compute these signals per epic, using **today's date**:
- `done?` — statusCategory is Done, or status is one of the delivered-ish statuses above.
- `endDate` — `customfield_10156`, else the latest `fixVersions[].releaseDate`.
- `overdue?` — `endDate` is in the past.
- `progress%` — child completion.
- `elapsed%` — fraction of the Start→End window that has passed.
- `atRisk?` — `flagged`, OR has an open blocker link, OR `progress%` lags `elapsed%` by a
  wide margin, OR `endDate` is imminent (e.g. within ~2 weeks) with material work left.

Then propose:

- **Green** — `done?`, **or** (`endDate` in the future AND not `atRisk?` AND `progress%`
  roughly on pace with `elapsed%`). On track to deliver by the target launch date.
- **Amber** — not `done?` and `atRisk?` but not yet certain to miss: behind pace, flagged,
  blocked, or End date imminent with work remaining. Problems/risks that could affect the ETA.
- **Red** — `overdue?` and not `done?`, **or** projected completion is clearly beyond
  `endDate` (e.g. very low `progress%` with little time left). The epic will miss the End date.

**Tie-breaker / nuance:** weight completion and status over the raw date. An epic whose End
date has passed but which is resolved/delivered is **Green**, not Red (e.g. SEARCHPU-35780).
Conversely, an early epic with no progress and an imminent End date trends Amber→Red even if
not yet overdue.

Output per epic: the proposed RAG **plus a one-line rationale that names the signals used**,
e.g. `Red — End date 2026-06-05 passed, status In Progress, 40% of children done`.

## Reconciliation goal

A **mismatch** is when the recorded `customfield_10159` disagrees with the data-driven
proposal. Reconciliation (guided, with the user) resolves it by agreeing the RAG **and**
correcting the underlying fields (typically `End date`) so that re-running this rubric on the
updated data yields the **same** RAG that was agreed — i.e. the epic would now **pass** the
check with no mismatch.
