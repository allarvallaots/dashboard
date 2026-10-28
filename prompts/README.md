# Scout pipeline — agent prompts (generic templates)

The dashboard (`../index.html`) is a **viewer**. It is populated by a background
multi-agent "scout" pipeline that reads your sources, writes structured results
into the artifact database, and verifies its own output. These files are generic
templates of the five agent prompts. They use placeholder project IDs
(`p1`, `p2`, `p3`), placeholder sources and keywords, and `example.com` as the
internal domain — replace those with your own.

## Topology

```
          ┌──────────────┐
  cron →  │ coordinator  │  (3×/day on weekdays)
          └──────┬───────┘
                 │ spawns
                 ▼
          ┌──────────────┐
          │    pull      │  → writes state/pull-latest + state/partner-activity
          └──────┬───────┘
                 │ then spawns (parallel)
        ┌────────┴─────────┐
        ▼                  ▼
  ┌───────────┐     ┌───────────┐
  │ synthesize│     │    dle    │  synth returns JSON; dle writes state/dle + decisions
  └─────┬─────┘     └─────┬─────┘
        └────────┬────────┘
                 ▼
        coordinator assembles state/dashboard.projects[]
                 │ fires
                 ▼
          ┌──────────────┐
          │  verifier    │  → writes state/verification
          └──────────────┘
```

## Core invariants (hold across every agent)

1. **Sole writer per collection.** Each DB collection has exactly one agent
   that writes it (see each prompt's "Access" section). No agent self-validates.
2. **Read-only on real systems.** Agents observe sources (mail, chat, calendar,
   files); they never send, reply, or modify anything outside the DB.
3. **The dashboard reads `commsMessages` + `calendarEvents` directly from
   `state/pull-latest`.** The coordinator never copies those into
   `state/dashboard` (doing so re-introduces context-compaction corruption).
4. **Carry forward, never blank.** A quiet or failed pull must leave the last
   good data in place. Durable stores (`state/partner-activity`, merged
   `recentActivity`) only advance.
5. **Real provenance.** Every message keeps a reference to the source tool call
   that produced it and a real timestamp. Null-time or un-sourced items are
   dropped, not reconstructed.

## Wiring

- Run the **coordinator** on a schedule (the reference cadence is 3×/day on
  weekdays, e.g. 09:00 / 13:00 / 16:00 in your timezone). It spawns the others.
- The **verifier** is fired by the coordinator at the end of each run (no
  schedule of its own).
- `pull`, `synthesize`, and `dle` are spawned by the coordinator (no schedules).

## Configure before use

- Project IDs and their source config live in the DB
  (`config/projects-<id>`, `config/sources-<id>`) — not in these prompts.
- Replace every `p1`/`p2`/`p3`, every placeholder keyword, and `example.com`.
- Point each source step at the connector tools available in your environment
  (an email/calendar connector, a chat connector, a files/wiki connector, and a
  local-file bridge if you use one).
