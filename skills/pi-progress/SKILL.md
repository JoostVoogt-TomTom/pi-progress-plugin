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
- **Extra JQL** (optional) — appended verbatim for power users.
- Whether to allow Jira writes this run (**default: no** — markdown only).

The **delivery quarter** and **team** are NOT asked as free text — they are chosen from live
option lists in step 2b. Skip that picker only if the user already named them in the invocation
(e.g. `Q26.2`/`NavSDK_2026_Q2` for the quarter, `DragonFly` for the team).

### 1a. Offer one-time read-only authorization (before the first Jira call)
To avoid a permission prompt on every data fetch, offer **once** to pre-authorize this skill's
**read-only** Jira tools. Ask a single y/n: *"Approve the read-only Jira data tools for this
project so I don't have to ask each time? (Writes always stay gated.)"*

If **yes**, add these rules to `.claude/settings.local.json` under `permissions.allow` (create
the file/key if absent; the `update-config` skill can do this for you). Use the MCP tool-name
form that matches this environment's Atlassian server (here `mcp__claude_ai_Atlassian__…`):
- `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources`
- `mcp__claude_ai_Atlassian__getJiraProjectIssueTypesMetadata`
- `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields`
- `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`
- `mcp__claude_ai_Atlassian__getJiraIssue`
- `Bash(jq:*)`

<HARD-GATE>
NEVER add a Jira **write** tool (`editJiraIssue`, `addCommentToJiraIssue`,
`transitionJiraIssue`) to the allow-list. The IRON-LAW write gate (step 10) must always require
explicit per-action confirmation. This one-time authorization covers **data fetching only**.
</HARD-GATE>

If the user **declines**, proceed normally (you'll be prompted per call).

### 2. Resolve cloudId
Call `getAccessibleAtlassianResources`; use the `tomtom` site's `id` as `cloudId` for all
subsequent calls. (This is the first Jira call — it runs after the step 1a authorization offer.)

### 2b. Discover teams + quarters and let the user choose (the opening interaction)
Discover both lists from **field metadata** — one fast, complete call — rather than scanning
tickets:

1. Resolve the **Epic** issue-type id for the project via `getJiraProjectIssueTypesMetadata`
   (find the type named `Epic`; commonly `10000` in SEARCHPU — verify, don't hard-code).
2. Call `getJiraIssueTypeMetaWithFields(projectIdOrKey=<P>, issueTypeId=<Epic id>)`. The
   response is large and auto-saves to a file — extract with `jq` (do NOT read the whole file):
   - **Teams** — allowed values of the `Teams` field:
     ```
     jq -r '.. | objects | select(.fieldId? == "customfield_10150" or .key? == "customfield_10150") | .allowedValues[]?.value' <file> | sort
     ```
   - **Quarters** — the project's fixVersions, filtered to the `Q<YY>.<N>` quarter pattern:
     ```
     jq -r '.. | objects | select(.fieldId? == "fixVersions") | .allowedValues[]?.name' <file> | grep -E '^Q[0-9]{2}\.[0-9]$' | sort -t. -k1.2n -k2n
     ```

Then present **two selection lists**:

- **Quarter** — an `AskUserQuestion` option list. Compute the current quarter from **today's
  date** (e.g. 2026-06-16 → `Q26.2`) and offer it (labelled *Recommended*) plus its nearest
  neighbours (previous + next quarter), max 4 options. The auto **"Other"** lets the user type
  any version verbatim (e.g. a `GOSDK` version like `NavSDK_2026_Q2`).
- **Team** — always including an **"All teams"** option:
  - **≤4 teams** → an `AskUserQuestion` chip list (the teams + "All teams").
  - **>4 teams** (e.g. SEARCHPU's 11) → present a clean **numbered list** and let the user
    reply with the number(s) or name(s). `AskUserQuestion` caps at 4 options, so the numbered
    list is the fallback whenever the project has more teams than fit.

The user may pick:
- **one team** → single-team run with interactive reconciliation (steps 6 and 8);
- **several teams** → run each selected team and produce a combined overview grouped by team;
- **All teams** → every epic in the project + quarter, grouped by team (read-only roll-up).

Notes:
- The `Teams` field allowed-values are the source of truth — they include teams that may have
  **no** epics in the chosen quarter (that's fine; those simply yield an empty run).
- If the user already named a team/quarter in the invocation, skip the relevant picker, but
  still validate the team appears in the discovered allowed values (warn + show the list if not).
- Multi-team and All-teams runs follow the read-only roll-up behavior in step 8.

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

### 7. Gather risk-register & freshness inputs
These power the **Highlights**, **Risk register**, and **per-team freshness** sections of the
report. All reads — they never write to Jira.

1. **Latest `RAG flag:` comment (Amber/Red epics only).** For every epic whose agreed RAG is
   **Amber or Red**, fetch its comments read-only via `getJiraIssue` requesting the `comment`
   field (if the response auto-saves to a file, extract with `jq`). Find the **latest** comment
   whose first line is `RAG flag: <Color>`; capture its body (the **follow-up action**) and its
   **created date**. Only the at-risk subset is fetched, to keep this cheap.
2. **Cycle window.** `(previous-report-date, today]`, where previous-report-date is the date of
   the most recent prior report for the same scope+quarter in `docs/pi-progress/`. If no prior
   report exists, default the window to **the last 14 days** (bi-weekly cadence).
3. **Reviewed this cycle?** — true when the epic has a `RAG flag:` comment whose created date
   falls in the cycle window. (For Green epics with no comment fetched, treat as not-reviewed
   only if you have a date to judge by; otherwise mark `n/a`.)
4. **Transitioned into risk?** — true when the epic's previous Agreed/Recorded RAG was Green
   (or absent) and its current Agreed RAG is **Amber or Red**. The previous RAG comes from the
   prior report you already read for "Changes since last report".
5. **Reason (per Amber/Red epic)** — the one-line rubric rationale from step 5 **plus** any open
   "is blocked by" links from `issuelinks`.

### 8. Guided reconciliation (take the user by the hand)
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

### 9. Write the markdown report (always)
Create `docs/pi-progress/<YYYY-MM-DD>-<team-or-project>-<quarter>.md` from
`report-template.md` (use the team name in the filename for a single-team run, or `all-teams`
for an all-teams run). The report leads with the meeting-ready material:
- **Highlights — for discussion** (up front): the things to raise in the PI meeting — epics
  that **transitioned into Amber/Red** this cycle, newly Red, large progress swings, newly
  blocked. Use the empty-state line if nothing material changed.
- **RAG summary** table.
- **Risk register (Amber & Red)**: one row per **every** Amber/Red epic (not just reconciled
  ones) — owner, RAG, the **reason** (signals + blockers from step 7.5), the **follow-up
  action** (this cycle's drafted next steps if reconciled, else the latest `RAG flag:` comment
  body from step 7.1), **reviewed this cycle?**, and **transitioned?**.
- **Per-epic detail** (status, End date, % progress, recorded → agreed RAG, rationale), and for
  each reconciled epic the agreed next steps and the drafted `RAG flag: …` comment + proposed
  field edits.
- **Changes since last report** at the bottom (the detailed diff; Highlights is its curated
  front-page subset).

For an **All teams** run, group the summary table and detail sections under a `## <Team>`
heading per team, and lead with an overall R/A/G roll-up plus a per-team R/A/G breakdown. Add a
**per-team freshness line** — "reviewed N of M epics this cycle" — and call out teams with any
unreviewed epics, so the reader can see which teams updated their tickets. This file also serves
as a local history of past reports — read the previous one in the folder if present (it is also
the source for the cycle window and transitions in step 7).

### 10. Optional Jira writes (gated — OFF by default)
<HARD-GATE>
Only if the user opted into Jira writes in step 1, offer to apply the drafted changes from
step 8. Present them as a concrete list per epic (field → new value, comment text) and require
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
epics were reconciled). Also surface the at-risk picture: how many epics are Amber/Red, how many
**transitioned into risk** this cycle, and how many epics/teams were **not reviewed this cycle**.
If Jira writes were applied, list exactly what was changed.

Then **offer the retrospective**: *"Want me to run `/pi-progress-retro` so the tool can learn
from this run?"* It turns this session's RAG overrides, friction, and report-quality gaps into
confirmed edits to the rubric / skill / template. It is especially worth running when the user
**overrode one or more proposed RAGs** this cycle (each override is a signal the rubric may be
miscalibrated). Offer it — do not auto-invoke.
