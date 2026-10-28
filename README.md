# Portfolio Scout Dashboard (generic template)

A single-file, live portfolio dashboard that surfaces status, comms, partner
activity, milestones and a Gantt view across multiple projects — fed by a
background multi-agent "scout" pipeline. This repository is a **genericized
template**: all organisation-, project- and person-specific data has been
replaced with placeholders. Real data is never stored in this file — it lives
in the artifact's runtime database at run time.

## What this is / isn't

- **Is:** the rendering + interaction engine, the data-health/normalisation
  hardening, the governance/pipeline views, and a small set of placeholder
  seed data (`Project Alpha/Beta/Gamma`, `Partner A/B/C`, `Your Org`).
- **Isn't:** a standalone web app. The page is built for the **Claude artifact
  runtime** and expects three capabilities to be granted when published:
  - `db` — the shared document store (projects, comms, partner activity, etc.)
  - `mcp` — a calendar/email connector (e.g. an Outlook / Microsoft 365
    connector) for the in-page calendar + free-time tools
  - `sample` — the in-page assistant (the chat box)

  Without those, the page renders from the placeholder seeds only.

## Configure (top of the main script)

```js
const TIMEZONE      = 'Europe/Tallinn';   // IANA tz for all date/time display
const CONNECTOR_M365 = 'Microsoft 365';   // display name of your calendar/email connector
```

Replace the connector name with whatever your Outlook/Microsoft 365 (or
equivalent) connector is called in your environment.

## Data model (runtime database)

The page reads/writes these collections (all project-specific content lives
here, not in the HTML):

- `config/projects-<id>` — base project structure (milestones, deliverables, partners)
- `config/sources-<id>`  — per-project source config (keywords, channels, folders)
- `config/gantt-<id>`    — Gantt work-package structure + owners
- `config/partner-rules` — domain→partner map, internal domains, consortium lists
- `config/mute-list`     — muted senders
- `state/pull-latest`    — latest pulled comms + calendar + recent activity
- `state/dashboard`      — assembled projects[] + sync metadata
- `state/partner-activity` — durable per-partner "last seen"
- `state/acks` `state/moves` `state/rejections` `state/dismissals` — user actions
- `state/sync` `state/coord-log` `state/verification` `state/dle` — pipeline bookkeeping

Project IDs in the placeholder seed are `p1`, `p2`, `p3`. Use your own IDs and
the page will subscribe to the matching `config/*` / `state/*` documents.

## The scout pipeline

The dashboard is populated by a scheduled multi-agent pipeline (coordinator →
pull → synthesize + decision-engine → verifier). Those agent prompts are **not**
included here (they contain environment-specific source config); this repo is
the front-end + data contract only.

## Placeholder data

`projects`, `ganttData`, `calendarEvents` and `commsMessages` near the top of
the script are illustrative placeholders so the page renders before any database
is attached. Replace or empty them for your own deployment.

## License

Add your preferred license here.
