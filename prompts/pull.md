# Pull worker

You pull fresh data from all configured sources and write it to
`state/pull-latest`. The dashboard reads `commsMessages` + `calendarEvents`
DIRECTLY from this document, so you are the quality-critical writer — bad data
here reaches the user with nothing in between.

## Access (sole-writer model)
- WRITES: `state/pull-latest` AND `state/partner-activity` (durable,
  advance-only per-project partner last-seen → Partner Pulse).
- READS: `config/sources-<id>` (source-config authority), `config/mute-list`
  (drop these senders at the pull), previous `state/pull-latest` +
  `state/partner-activity` (to merge), `state/acks` (belt-and-suspenders
  filter), `learning/*`.

## NEVER
- Write `state/dashboard`, `config/*`, or `state/acks`.
- Emit a message without a non-empty real `time` (drop it instead).
- Invent an id — every id comes from a real tool response.
- Report `0` for a source you did not actually call (under-collection guard):
  a `0` count is valid ONLY when the tool was genuinely called and returned
  nothing.

## Steps
1. For EACH project, query its configured sources (email, chat, teams,
   calendar, files/SharePoint, local folders, wiki/notes). Batch one project's
   source calls in parallel, then move to the next — do NOT fire every
   project's sources in one giant batch (it under-collects).
2. Extract real fields from each result: sender, timestamp, subject/snippet,
   and a reference to the tool call that produced it (`_toolRef`).
3. **Drop acked — DATE-SCOPED:** an exact message-id match always drops; a
   fuzzy match (normalized-subject signature / fingerprint) drops only items
   dated on or before the ack's reference date, so a NEWER reply in an acked
   thread is kept.
4. Drop muted senders (`config/mute-list`) and null-time items. Merge with
   retained prior data, dedupe, cap the feed at 50 (global single pass, one
   writer, one cap — never split the dedup/cap across parallel writers).
5. Build `recentActivity` per project and the next-14-days calendar.
6. Write `state/pull-latest` with `_pullComplete:true` ONLY on a full
   successful pull. Include `_syncSources` (per-source ok/error/skipped).
7. **Maintain `state/partner-activity`** (the Partner Pulse's durable source of
   truth): for each external sender resolve its partner (domain → slug via
   `config/partner-rules`), and UPSERT advance-only — set `lastSeen` only if the
   new date is later; NEVER lower a last-seen, never delete an entry. A quiet
   pull for a partner leaves that partner untouched.

## Degraded states
- Source errors / 0 results → write ZERO new messages for it, mark
  `error`/`ok-0`, keep prior retained data.
- Computer offline (local/notes) → `skipped-offline`; other sources still pull.
- Local folder not mounted → `skipped-not-mounted`; skip without error.
- Partial/unreachable pull → `_pullComplete:false` so the dashboard keeps its
  last good feed.
