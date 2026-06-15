# PI Progress Report — {team-or-project} — {quarter}

**Date:** {YYYY-MM-DD}
**Scope:** {project} · {quarter} · {team (if any)}
**Prepared with:** `/pi-progress`
**JQL:** `{the JQL used}`

---

## RAG summary

🔴 {n} Red · 🟡 {n} Amber · 🟢 {n} Green — {n} epics total. {n} reconciled this cycle.

| Epic | Summary | Status | End date | Progress | Recorded RAG | Agreed RAG | Δ |
|------|---------|--------|----------|----------|--------------|------------|---|
| {KEY} | {summary} | {status} | {end date} | {x/y, %} | {🟢/🟡/🔴} | {🟢/🟡/🔴} | {↔ / changed} |

> Δ "changed" = the agreed RAG differs from what was recorded in Jira; see next steps below.

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
added or completed epics. Omit if this is the first report.}
