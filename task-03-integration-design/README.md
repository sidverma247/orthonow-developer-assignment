# Task 03 — Integration Design

Server-side architecture for turning a submitted lead into a HubSpot contact, a WhatsApp confirmation
(within a 2-minute SLA) and a Google Ads conversion — without duplicates or lost leads.

- **[README_Task_03_Integration_Design.md](README_Task_03_Integration_Design.md)** — the integration
  design write-up: end-to-end architecture, the dedup trap, failure/fallback and the WhatsApp SLA.

## The core decisions, at a glance

| Decision | Choice | Why |
|----------|--------|-----|
| Orchestration | Custom backend API | Needs dedup, retries, SLA, idempotency — connectors hide these. |
| No-code (Zapier/Make) | Rejected | Opaque steps, no DLQ control, per-task cost, weak monitoring. |
| HubSpot dedup | **Search by phone** before upsert | HubSpot dedupes by *email*; our audience is phone-first. |
| Shared phone, different name | Create + associate + `needs_review` | Never silently overwrite the wrong person. |
| Reliability | SQS + Dead Letter Queue + idempotency key | No lost leads, safe retries, no double sends. |
| WhatsApp | Karix via queue, monitored p95 | Decouples the 2-min SLA from the request thread. |
| Google Ads | dataLayer → GTM → GA4 → import | One deduplicated source of truth; server-side reliable. |

Export the data-flow diagram to `architecture-diagram.png` (the ASCII version is in the doc).
