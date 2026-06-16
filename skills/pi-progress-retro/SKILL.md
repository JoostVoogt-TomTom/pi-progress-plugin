---
name: pi-progress-retro
description: "Use at the end of a /pi-progress run to let the tool learn from itself — analyzes where the agreed RAG overrode the proposed RAG (rubric calibration), plus session friction and report-quality gaps, and applies confirmed edits to rag-rubric.md / SKILL.md / report-template.md. One retro per report, saved alongside it."
---

# PI Progress — Self-Retrospective

The feedback loop that keeps `/pi-progress` sharp. After a progress report is written, this
skill looks back at the run and turns three kinds of signal into concrete edits to the
plugin's own instruction files:

1. **RAG override calibration (the prime signal)** — every epic where the **agreed RAG
   differed from the RAG the rubric proposed** is direct evidence `rag-rubric.md` is
   miscalibrated. This is unique to this tool: the user's override *is* the ground truth.
2. **Session friction** — corrections, re-asks, confusion → fixes to `pi-progress/SKILL.md`.
3. **Report quality** — gaps or awkwardness in the generated markdown → fixes to
   `report-template.md`.

It writes a retro markdown file next to the report, then — gated by an explicit per-change
y/n — applies the confirmed edits.

**Announce at start:** "I'm using the pi-progress-retro skill to learn from this run."

<HARD-GATE>
- **No invented findings.** Every finding MUST quote or reference a concrete moment from THIS
  session — a RAG override, a correction, a re-ask, a redo, or a report gap the user pointed
  at. If you cannot cite the moment, the finding does not go in. A smooth run with one finding
  (or zero) is a valid retro; do not pad.
- **Plugin instruction files only.** This skill edits exactly three files:
  `skills/pi-progress/rag-rubric.md`, `skills/pi-progress/SKILL.md`, and
  `skills/pi-progress/report-template.md`. It MUST NOT call any Jira tool — the `/pi-progress`
  IRON-LAW (read-only against Jira) still holds in full. It MUST NOT edit the generated report
  or the drafted Jira changes; those are the record of what happened, not retro inputs to
  rewrite.
- **Per-change confirmation.** Each file edit requires an explicit **y/n** from the user
  before it is applied. Findings the user does not confirm are recorded as `applied: no`.
</HARD-GATE>

## Prerequisites

- A `/pi-progress` run happened in THIS session and wrote a report under `docs/pi-progress/`.
- You can see the run in the conversation (the proposed RAG table from step 6 and the agreed
  RAGs from reconciliation) — the override signal comes from there.

If no `/pi-progress` run is present in this session, stop and tell the user:
*"The retrospective needs a /pi-progress run in this session to learn from. Run /pi-progress
first, then come back."*

## Process

Create a task for each step and complete them in order.

### 1. Locate the run
Find the report just written: `docs/pi-progress/<YYYY-MM-DD>-<team-or-project>-<quarter>.md`
(the path `/pi-progress` reported in its Done message; if several exist, use the one from this
session). Derive the retro path by inserting `-retro` before `.md`, e.g.
`docs/pi-progress/2026-06-15-dragonfly-Q26.2-retro.md`.

### 2. Check for an existing retro
If the `-retro.md` file already exists, read it and ask the user:
*"A retrospective for this report already exists. Overwrite with a fresh pass that merges the
previous findings? (yes/no)"* — If no → STOP. If yes → you MUST carry every previous finding
forward into the new file (union of old + new); do not silently drop findings you disagree
with — add a new note explaining the disagreement instead.

### 3. Harvest calibration signals (the prime bucket)
From the session and the report, list **every epic where the agreed RAG ≠ the proposed RAG** —
i.e. the user overrode what the rubric in `rag-rubric.md` computed. For each override capture:
- epic key,
- **proposed** RAG (what the rubric said) and **agreed** RAG (what the user decided),
- the **signals the rubric used** in its rationale (status, End date vs today, % done,
  flagged/blocked, pace),
- the **reason the user gave** for overriding.

Each override is a candidate `rag-rubric.md` change: a threshold to adjust (e.g. the "~2 weeks
imminent" window), wording to clarify, a delivered-ish status to add to the Green list, or a
tie-breaker to reweight. **Distinguish a systematic miscalibration from a one-off:** if the
override reason is specific to that single epic ("this one is special because…"), it is a
`hint`, not a rubric change — only generalizable overrides edit the rubric.

### 4. Scan session friction
Walk the conversation for friction moments and quote each one:
- **Corrections** — "no", "stop", "that's wrong".
- **Re-asks** — the user had to restate something they already said.
- **Redos** — the report or a draft was rewritten to undo something.
- **Confusion** — the user asked what you were doing, or got output they didn't expect.
- **Missing context** — the user pointed you at a convention/field/doc you should have known.

A plain "no" given as a genuine answer to a question is not friction — only flag "no" when it
pushes back on something the skill did or proposed.

### 5. Scan report quality
Note any place the user reacted to the **generated markdown** itself — a missing column, a
section that was always empty, an ordering that buried the important rows, a field they had to
add by hand. These are candidate `report-template.md` changes.

### 6. Categorize each finding and propose the concrete fix
Put every finding in exactly **one** bucket and write the exact change — file path + a
diff-style before→after. No hand-waving like "improve the rubric".

| Bucket | Lands in | When |
|---|---|---|
| `rubric` | `skills/pi-progress/rag-rubric.md` | a RAG override that generalizes — recalibrate a signal, threshold, status list, or tie-breaker |
| `skill` | `skills/pi-progress/SKILL.md` | the workflow told you the wrong thing or omitted a step → session friction |
| `template` | `skills/pi-progress/report-template.md` | the report shape produced friction → a section/column/ordering change |
| `hint` | (not a file) | needs human judgement or was a one-off; presented to the user, never auto-applied |

### 7. Self-review (subagent)
Dispatch one review subagent with the draft findings and this skill as reference. It checks:
1. every finding quotes a concrete moment from this session;
2. every `rubric`/`skill`/`template` fix names a concrete file + a minimal diff (not a rewrite);
3. every `rubric` finding traces to an actual override (proposed ≠ agreed), not a hunch;
4. one-off overrides are filed as `hint`, not `rubric`.
Fix and re-dispatch until it returns `APPROVED` (cap at 3 iterations; if still not approved,
record `Self-review: DISPUTED` in the retro header and proceed).

### 8. Write the retro file
Write the `-retro.md` file next to the report using the structure below.

### 9. Auto-apply gate
Present the proposed edits as a concrete list — for each: `file → before → after`. Ask the
user **y/n per change**. Apply only the confirmed ones via `Edit` to the three plugin files.
After each, update the matching finding in the retro file to `applied: yes` (or `no`). Never
touch Jira, the report, or any other file.

### 10. Done
Report: the retro file path, the count of findings per bucket, the number of RAG overrides
analyzed, and exactly which plugin files were edited (with a one-line summary each).

## Retro file structure

```markdown
# PI Progress Retrospective — {team-or-project} — {quarter}

**Date:** {YYYY-MM-DD}
**Scope:** {project} · {quarter} · {team (if any)}
**Source report:** {path to the <date>-<team>-<quarter>.md report}
**Self-review:** {APPROVED | DISPUTED}

---

## Calibration overrides
_Epics where the agreed RAG differed from the rubric's proposal — the primary learning signal._

| Epic | Proposed | Agreed | Rubric signals used | Override reason | Proposed rubric change | Applied? |
|------|----------|--------|---------------------|-----------------|------------------------|----------|
| {KEY} | {🟢/🟡/🔴} | {🟢/🟡/🔴} | {status, End date, %…} | {why the user overrode} | {rag-rubric.md: before → after, or "hint (one-off)"} | {yes/no} |

_(empty if every agreed RAG matched the proposal — note that explicitly.)_

---

## Findings

### `rubric` — RAG calibration (rag-rubric.md)
- **Moment:** {quoted override / correction}
- **Change:** `skills/pi-progress/rag-rubric.md` — {before → after}
- **applied:** {yes/no}

### `skill` — workflow instructions (pi-progress/SKILL.md)
- **Moment:** {quoted friction}
- **Change:** `skills/pi-progress/SKILL.md` — {before → after}
- **applied:** {yes/no}

### `template` — report shape (report-template.md)
- **Moment:** {quoted report-quality reaction}
- **Change:** `skills/pi-progress/report-template.md` — {before → after}
- **applied:** {yes/no}

### `hint` — for next time (not auto-applied)
- {collaborative "next time, try X because Y" — including one-off overrides}

_(omit any bucket with no findings.)_
```

## Anti-rationalization

| Excuse | Reality |
|--------|---------|
| "The run went fine, nothing to retro" | If the user overrode a proposed RAG or corrected you, that's a finding. If truly nothing happened, a zero-finding retro is fine — don't invent. |
| "This override means the rubric is wrong" | Only if it generalizes. A one-off reason specific to one epic is a `hint`, not a `rag-rubric.md` edit. |
| "Let me rewrite the rubric to be safe" | Propose the minimal diff that fixes the cited override. Big rewrites belong in their own session. |
| "I'll just fix the report too while I'm here" | No. The report is the record of the run. Retro edits the instruction files only. |
| "I'll apply all the edits, they're obviously right" | No. Every edit needs an explicit y/n. The human is the authority — same as RAG reconciliation. |

## Red flags — STOP

- Writing a finding without quoting the moment.
- Proposing a vague fix ("clarify the rubric") with no before→after diff.
- Editing the report, the Jira drafts, or any file outside the three instruction files.
- Calling any Jira tool.
- Applying an edit the user did not confirm y/n.
- Overwriting an existing retro without merging its findings forward.
