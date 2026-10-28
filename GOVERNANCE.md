# Governance — Portfolio Monitoring System

This system is governed with an **Orchestration Design Canvas** (ODC, after
Bornet et al.): a structured statement of what the automation is for, where the
human stays in control, and how it fails safe. The dashboard's "Governance" tab
renders a version of this; the generic canvas is below. Replace the placeholders
(`Project Alpha/Beta/…`, "the owner", source names) with your own.

## S1 — Source (why it exists)

**Intent.** Reduce the cognitive overhead of tracking several concurrent
programmes across multiple data sources (email, chat, group chat, shared files,
local files/notes, calendar) by automating status observation, comms triage and
cross-project surfacing into one live dashboard. **It observes and surfaces; it
does not decide, prioritise, or message anyone.**

**Core value that stays human.** Priority judgment (which risk matters when
several compete), relationship reading (when to push vs. wait), cross-domain
pattern recognition, escalation calibration (noise vs. real crisis), and final
decision authority. The dashboard surfaces; the owner acks / moves / sets status.

**Skills to protect.** Keep genuinely reviewing agent output, interrogate sources
by hand periodically, and read the decision/weekly logs yourself — so the human
layer does not atrophy.

## S2 — Success (what good looks like)

- A live dashboard that always reflects the current state of each project: what
  changed, what milestone is approaching, what needs attention; a comms feed of
  only un-actioned items; partner pulse and calendar current.
- Scannable in under two minutes on phone or desktop.
- **Quality bar.** Good: *"Alpha design review tomorrow; deliverable D1.2 at
  risk; 3 un-acked emails; 6 docs changed today."* Bad: *"Alpha looks on track;
  some files may have changed."* No null-time messages, no resurrected
  acked/rejected items, no stale calendar.
- **Target.** Every milestone entering its warning window is visible (zero
  missed warnings); meeting-ready in minutes, not half an hour of collation.

## S3 — Safety (the guardrails)

**Named owner.** One human is accountable for every surfaced finding and every
automated write.

**Agents NEVER:** send messages to partners/external parties; modify `config/*`;
write `state/acks`; write `commsMessages`/`calendarEvents`/`partnerActivity`
into `state/dashboard`; delete any file or DB document; make prioritisation
decisions; downgrade a status a human set.

**Gates** (T = architecture-enforced, H = person-checked):

| Gate | Type | Enforcement |
|---|---|---|
| Read-only on source systems | T | Agents have no send/write capability on sources |
| Sole-writer DB authority | T | Each collection has exactly one writer agent (see below) |
| Provenance + real `time` on every message | T | Pull self-check + verifier cross-check; null-time dropped |
| `_pullComplete` before the dashboard trusts a feed | T | An incomplete pull is ignored; last good feed kept |
| Ack / move / reject | H | Done by the human in the dashboard UI |
| Status → blocked, priority, partner comms | H | Agent flags; the human decides |

**Sole-writer DB authority.**

| Collection | Writer |
|---|---|
| `config/*` (incl. `config/mute-list`, `config/gantt-*`) | Human |
| `state/pull-latest` (comms + calendar + recentActivity) | Pull worker |
| `state/partner-activity` (durable, advance-only) | Pull worker |
| `state/dashboard` (projects[] + sync meta only) | Coordinator |
| `state/dle` + `decisions/*` | Decision engine |
| `state/verification` | Verifier |
| `state/acks` / `state/rejections` / `state/moves` / `state/dismissals` | Dashboard UI |

No agent self-validates — the verifier is a separate writer.

**Input contract.** A missing/degraded pull sets `_pullComplete:false` and the
dashboard holds its last good feed. Any field with no new data is carried
forward, never overwritten with null.

**Retirement criterion.** If data corruption recurs after config/prompt fixes
(a drifting design), **rebuild the pipeline rather than patch again.**

## S4 — Steering (decision rights)

- **Autonomous:** pull sources, dedup/merge comms, compute days-to-milestone,
  synthesise summaries, assemble `projects[]`, write the dashboard, process
  already-logged decisions.
- **Recommend (human/DLE accepts):** baseline-change proposals, status
  suggestions, attention ranking.
- **Escalate (human decides in the dashboard):** any status → blocked, priority
  reordering, partner/customer comms, stalled-project disposition, cross-project
  resource conflicts.

## S5 — Switch (exception → safe default)

| Trigger | Default state |
|---|---|
| Coordinator run crashes / stalls mid-pipeline | Breadcrumb `_currentStep` shows where; previous state retained |
| Degraded / partial pull | `_pullComplete:false` → keep last good feed; null-time dropped |
| A source connector unavailable | Run the rest; mark it `error`/`skipped`; keep its prior data |
| Computer offline at run time (local/notes) | `skipped-offline`; cloud sources still pulled |
| Decision-engine batch write fails | Fall back to per-decision writes; never leave decisions unmarked |
| Acked/rejected item reappears | Filtered client-side (exact id, or date-scoped fuzzy match — a newer reply in an acked thread is never suppressed) |

## S6 — Sharpen (keeping it honest)

- **Validation.** The verifier samples each run's output, checks provenance +
  structure, and writes `state/verification` (shown as badges). The coordinator
  self-audits against learned correction patterns.
- **Post-incident learning.** A `learning` collection holds correction patterns
  applied as constraints on later runs.
- **Drift signals.** Null-time count (→0), acked-leak count (→0), verifier pass
  rate, `_pullComplete` rate, runs stalling early. The **Data-health strip** at
  the top of the dashboard is the at-a-glance surface: it flags skipped/unmounted
  sources, stale durable stores and malformed rows — quiet when clean.
- **Human-layer check.** Periodically ask: is the human doing *more* judgment and
  *less* manual collation than 90 days ago? If not, the automation is not
  earning its keep.

---
*Framework: Orchestration Design Canvas (Bornet et al.). Child agent prompts:
`coordinator.md`, `pull.md`, `synthesize.md`, `dle.md`, `verifier.md`.*
