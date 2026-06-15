---
name: pi-progress
description: "Use when preparing a bi-weekly / quarterly engineering progress update — pulls a team's epics from Jira for a delivery quarter, proposes a RAG status (Red/Amber/Green) from live Jira data, guides the user through reconciling any mismatch (RAG + fields like End date), and writes a markdown progress report. Never writes to Jira without explicit confirmation."
---

# PI Progress — RAG Reporting

Generates the RAG (Red / Amber / Green) progress report engineering teams present at the
bi-weekly quarterly-progress meeting. For a chosen **team / project + delivery quarter**, it
reads the relevant **epics** from Jira, **proposes** a RAG status from real Jira data, shows
it against the **currently recorded** RAG, and when they disagree it **takes the user by the
hand** to reconcile the epic so its data and RAG line up. Output is a **markdown report**.

<IRON-LAW>
This skill is READ-ONLY against Jira by default. It MUST NOT call any Jira write tool
(`editJiraIssue`, `addCommentToJiraIssue`, `transitionJiraIssue`, …) unless the user has
explicitly opted into Jira writes AND confirmed that specific action. The default,
always-safe output is the markdown report. This matters especially during the testing phase.
</IRON-LAW>

## RAG definitions (authoritative)

- **Red** — the epic will **miss the End date**. Next steps MUST state the mitigation
  actions and **when** the new ETA / scope will be communicated.
- **Amber** — problems encountered or risks identified that could affect the final ETA. Next
  steps MUST state the mitigation actions and the **path-to-green**.
- **Green** — on track to deliver by the target launch date. No further update needed.

When a RAG status **changes**, the next steps are recorded as a comment in the team's
existing convention: a comment whose first line is `RAG flag: <Color>` followed by the
next-steps text. Reproduce this format exactly.

## Field map (TomTom Jira — cloudId resolved at runtime)

See `rag-rubric.md` (next to this file) for the full map and the decision rubric. Key fields:

| Concept | Field | Notes |
|---|---|---|
| RAG status (recorded) | `customfield_10159` | Select — `{value: "Green"|"Amber"|"Red"}`. Green option id `14003`. |
| End date (target launch) | `customfield_10156` | Per-epic date. Primary date for Red/Green. Fallback: fixVersion `releaseDate`. |
| Start date | `customfield_10015` | For pace / elapsed calculation. |
| Team | `customfield_10150` | Select — e.g. `{value: "DragonFly"}`. How team dashboards scope epics. |
| Quarter | `fixVersions[].name` | e.g. `Q26.2` = 2026 Q2 (SEARCHPU), `NavSDK_2026_Q2` (GOSDK). |
| Status | `status` / `statusCategory` | Custom names; standard categories To Do / In Progress / Done. |
| Flagged | `customfield_10021` | Risk signal. |
| Blockers | `issuelinks` | "is blocked by" open links = risk signal. |

## Process

### 1. Collect inputs
Establish, in order (accept any provided in the invocation; otherwise ask):
- **Project key** — default `SEARCHPU` (also valid: `GOSDK`, …).
- **Delivery quarter / fixVersion** — e.g. `Q26.2` (SEARCHPU) or `NavSDK_2026_Q2` (GOSDK).
- **Team** — chosen from the project's actual teams (see step 1a). This is the value of the
  `Teams` field (`customfield_10150`); team dashboards scope by it.
- **Extra JQL** (optional) — appended verbatim for power users.
- Whether to allow Jira writes this run (**default: no** — markdown only).

### 1a. Offer the team list and let the user choose (the opening interaction)
This is the **first thing** the skill does after resolving `cloudId` (step 2): it **offers the
list of teams** and asks which to generate the overview for. Discover the teams live rather
than hard-coding them — query the distinct `Teams` values:
```
project = <P> AND issuetype = Epic AND "Teams" is not EMPTY ORDER BY updated DESC
```
requesting only `fields: ["customfield_10150"]`, `maxResults: 100`. The response will
auto-save to a file — extract the distinct, ordered team list with `jq`:
```
jq -r '.issues.nodes[].fields.customfield_10150.value // empty' <file> | sort | uniq -c | sort -rn
```
Then **present the discovered teams as a selectable list** (each with its epic count) using a
selection prompt (`AskUserQuestion`, multi-select), always including an **"All teams"** option.
The user may pick:
- **one team** → single-team run with interactive reconciliation (steps 6–7);
- **several teams** → run each selected team and produce a combined overview grouped by team;
- **All teams** → every epic in the project + quarter, grouped by team (read-only roll-up).

Notes:
- As of this writing, SEARCHPU teams seen are: **Moly, Mystery, DragonFly, Bob, Alchemists,
  Delivery** — a fallback hint only; the live discovery query is the source of truth.
- If the user already named a team (or teams) in the invocation, skip the prompt and use it,
  validating each appears in the discovered list (warn and re-offer the list if not).
- Multi-team and All-teams runs follow the read-only roll-up behavior in step 7.

### 2. Resolve cloudId
Call `getAccessibleAtlassianResources`; use the `tomtom` site's `id` as `cloudId` for all
subsequent calls. (Do this before step 1a, since team discovery needs it.)

### 3. Fetch epics
`searchJiraIssuesUsingJql` with:
```
project = <P> AND issuetype = Epic AND fixVersion = "<Q>" [AND "Teams" = <Team>] [AND <extra>] ORDER BY status
```
Request `fields`: `summary, status, customfield_10159, customfield_10156, customfield_10015,
customfield_10150, fixVersions, assignee, customfield_10021, labels, issuelinks`.
For a single-team run, include `AND "Teams" = <Team>`. For an **All teams** run, omit that
clause and use `customfield_10150` (already requested) to group the output by team.

> Jira responses routinely exceed the inline token cap and auto-save to a file. When that
> happens, do NOT try to read the whole file — extract what you need with `jq` (or hand the
> file to a subagent with the exact schema). Pull per epic: key, summary, status.name,
> status.statusCategory.name, RAG (`customfield_10159.value`), End date (`customfield_10156`),
> Start date (`customfield_10015`), fixVersions[].name, assignee.displayName, Flagged,
> issuelinks.

### 4. Per-epic progress
For each epic, `searchJiraIssuesUsingJql` with `parent = <EPIC-KEY>` (fields: `status`).
Compute **% done** = children with `statusCategory = "Done"` ÷ total children. (Again, use
`jq` on the saved file if large; just count by `statusCategory`.)

### 5. Propose RAG
Apply the rubric in `rag-rubric.md` using **today's date**, weighting **completion / status
over the raw date** (an epic past its End date but effectively delivered is Green, not Red).
For each epic produce: **proposed RAG + a one-line rationale naming the signals** (status,
End date vs today, % done, flagged/blocked). Compare to the recorded `customfield_10159`.

### 6. Present the table
Show one row per epic: `key | summary | recorded RAG | proposed RAG | match? | rationale`.
A **mismatch** is where the recorded RAG is inconsistent with what the data implies. State
clearly which epics match (no action) and which need reconciliation.

### 7. Guided reconciliation (take the user by the hand)
> **All-teams and multi-team runs are a read-only roll-up.** With potentially dozens of
> epics, do not run the interactive per-epic flow below; instead list the mismatches in a
> *Recommended reconciliations* table and suggest a focused single-team run to resolve them.
> The step-by-step reconciliation below applies to **single-team** runs.

For **each mismatched epic, one at a time**, walk the user through resolving it. Ask, in
order, and wait for the answer before continuing:

1. **What should the RAG status become?** — Red / Amber / Green. Offer the proposal as the
   default and let them override. The human is the authority.
2. **Why?** — capture the reason / next steps, shaped by the chosen RAG:
   - Red → mitigation actions **and when** the new ETA / scope will be communicated.
   - Amber → mitigation actions **and** the path-to-green.
   - Green → brief confirmation it's on track.
3. **Which Jira fields need updating so the ticket is consistent with that RAG?** — most
   importantly **End date** (`customfield_10156`), and where relevant status,
   fixVersion / quarter, or Flagged. Be concrete and explain the link, e.g. *"the recorded
   End date 2026-06-05 has passed, so the data reads Red; if the real target is now Q3, set
   End date to 2026-09-30 and the epic reads Green again."* The objective is that **after the
   proposed updates, a re-run of the RAG check would PASS** (proposed == recorded, no
   mismatch).

Then **re-evaluate** the proposed RAG against the proposed field values and confirm the
mismatch is resolved. Draft, but do not yet apply:
- the new RAG value (`customfield_10159`),
- any field edits (e.g. End date),
- the `RAG flag: <Color>` + next-steps comment.

Repeat for the next mismatched epic.

### 8. Write the markdown report (always)
Create `docs/pi-progress/<YYYY-MM-DD>-<team-or-project>-<quarter>.md` from
`report-template.md` (use the team name in the filename for a single-team run, or `all-teams`
for an all-teams run). Include the summary RAG table, per-epic detail (status, End date,
% progress, recorded → agreed RAG, rationale), and for each reconciled epic the agreed next
steps and the drafted `RAG flag: …` comment + proposed field edits. For an **All teams** run,
group the summary table and detail sections under a `## <Team>` heading per team, and lead
with an overall R/A/G roll-up plus a per-team R/A/G breakdown. This file also serves as
a local history of past reports — read the previous one in the folder if present to note what
changed since last cycle.

### 9. Optional Jira writes (gated — OFF by default)
<HARD-GATE>
Only if the user opted into Jira writes in step 1, offer to apply the drafted changes from
step 7. Present them as a concrete list per epic (field → new value, comment text) and require
an explicit per-epic (or per-action) **y/n** confirmation. Apply only what is confirmed:
- field edits via `editJiraIssue` (`customfield_10159`, `customfield_10156`, …),
- the comment via `addCommentToJiraIssue`.
For the RAG select, fetch the Amber/Red option IDs at write time via
`getJiraIssueTypeMetaWithFields` (do not hard-code beyond Green=`14003`). If the user did not
opt in, STOP after the markdown report and tell them the drafts are in the report, ready to
apply on a future run.
</HARD-GATE>

## Done
Report the path to the markdown file and a one-line RAG summary (counts of R/A/G and which
epics were reconciled). If Jira writes were applied, list exactly what was changed.
