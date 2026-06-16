# PI Progress Report — {team-or-project} — {quarter}

**Date:** {YYYY-MM-DD}
**Scope:** {project} · {quarter} · {team (if any)}
**Prepared with:** `/pi-progress`
**JQL:** `{the JQL used}`
**Cycle window:** {previous-report-date → today, or "last 14 days (no prior report)"}

---

## Highlights — for discussion

_The important changes to raise in the PI progress meeting — read this first._

- 🔴→ {KEY} {summary} — **newly Red / transitioned into risk**: {one line — what changed & why}
- 🟡→ {KEY} {summary} — **transitioned to Amber**: {one line}
- ⛔ {KEY} — **newly blocked**: {blocker}
- 📉 {KEY} — **progress slipped**: {from → to}

_{If nothing material changed this cycle, state: "No RAG transitions or new risks since the last
report — all at-risk items carried over (see Risk register)."}_

---

## RAG summary

🔴 {n} Red · 🟡 {n} Amber · 🟢 {n} Green — {n} epics total. {n} reconciled this cycle.
{n} transitioned into risk · {n} not reviewed this cycle.

| Epic | Summary | Status | End date | Progress | Recorded RAG | Agreed RAG | Δ |
|------|---------|--------|----------|----------|--------------|------------|---|
| {KEY} | {summary} | {status} | {end date} | {x/y, %} | {🟢/🟡/🔴} | {🟢/🟡/🔴} | {↔ / changed} |

> Δ "changed" = the agreed RAG differs from what was recorded in Jira; see next steps below.

---

## Risk register (Amber & Red)

_Every at-risk epic, its reason, and the follow-up action — for stakeholders tracking what's at
risk and who is unblocking it. Covers all Amber/Red epics, not only those changed this cycle._

| Epic | Owner | RAG | Reason (signals + blockers) | Follow-up action | Reviewed this cycle? | Transitioned? |
|------|-------|-----|-----------------------------|------------------|----------------------|---------------|
| {KEY} | {assignee} | {🟡/🔴} | {rationale + open "is blocked by" links} | {latest `RAG flag:` next steps — from Jira or this cycle's draft} | {✅ yes / ⚠️ no} | {yes → from 🟢 / no} |

_{If no Amber/Red epics: "No epics at risk this cycle. 🟢"}_

---

## Per-epic detail

### {KEY} — {summary}
- **Agreed RAG:** {🟢/🟡/🔴} (was {recorded})
- **Status:** {status} · **End date:** {end date} · **Progress:** {x of y children done, %}
- **Rationale:** {one line naming the signals — status, End date vs today, %, flagged/blocked}
- **Next steps:** {for Red — mitigation + when new ETA/scope is communicated; for Amber —
  mitigation + path-to-green; for Green — on-track confirmation}
- **Proposed Jira edits:** {e.g. End date → 2026-09-30; RAG → Green} *(applied: yes/no)*
- **Drafted comment** (team `RAG flag:` convention):
  ```
  RAG flag: {Color}
  {next-steps text}
  ```

_(repeat per epic; epics that match and need no change can be listed compactly under the
summary table without a detail block)_

---

## Changes since last report
{Diff vs the previous report in docs/pi-progress/, if one exists: RAG transitions, newly
added or completed epics. Omit if this is the first report. The Highlights section above is
the curated, meeting-facing subset of this diff.}

---

<!--
ALL-TEAMS VARIANT — when the run covers every team (read-only roll-up):
- Lead with an overall 🔴/🟡/🟢 roll-up and a per-team R/A/G breakdown.
- Add a per-team FRESHNESS line so the reader sees which teams updated their tickets:
    ### {Team} — 🔴 {n} · 🟡 {n} · 🟢 {n} — reviewed {N} of {M} epics this cycle
    {if N < M, list the unreviewed epic keys: "Not reviewed: SEARCHPU-xxxx, SEARCHPU-yyyy"}
- Group the RAG summary table, Risk register, and per-epic detail under a `## {Team}` heading
  per team. The Highlights section stays a single combined list across all teams.
- Replace single-team interactive reconciliation with a "Recommended reconciliations" table.
-->

