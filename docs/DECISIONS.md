# Architecture Decision Records

Format: each decision gets a number, date, status, context, decision and consequences.
Status is one of: Proposed, Accepted, Superseded by ADR-N.

---

## ADR-001: TypeScript full-stack with Next.js PWA
- **Date:** 2026-10-19 · **Status:** Proposed
- **Context:** Solo developer. Needs a doctor dashboard (mobile-first) and later a patient portal.
- **Decision:** One TypeScript codebase. Next.js App Router, delivered as an installable PWA.
- **Consequences:** No app-store release cycle. Shared types across UI, API and workflows. Patient push notifications are limited until a native app exists, so SMS covers that.

## ADR-002: PostgreSQL (Supabase) as source of truth
- **Date:** 2026-10-19 · **Status:** Proposed
- **Context:** Scheduling needs strong consistency (no double-booking, atomic slot claims) and realtime UI updates.
- **Decision:** Postgres via Supabase. Use a `tstzrange` exclusion constraint to prevent overlaps, and Supabase Realtime for live UI.
- **Consequences:** Some vendor coupling. Mitigated because it is still standard Postgres and can be moved.

## ADR-003: Durable workflow engine (Inngest) for autonomy
- **Date:** 2026-10-19 · **Status:** Proposed
- **Context:** Autonomous behaviour means long-running, time-based logic ("wait 15 min for reply, else…") that must survive deploys and crashes.
- **Decision:** Inngest functions triggered by domain events.
- **Alternatives:** Temporal (too heavy), Graphile Worker or pg-boss (possible later self-hosted migration).

## ADR-004: SMS behind a provider adapter; link-based responses
- **Date:** 2026-10-19 · **Status:** Proposed
- **Context:** SMS pricing and delivery vary a lot by country. Two-way SMS is unreliable in many regions.
- **Decision:** `SmsProvider` interface with a local aggregator as primary. Patient responses come through signed links, not SMS replies.
- **Consequences:** Patients need a data connection to respond. That is acceptable, and SMS replies can be added later.

## ADR-005: "Suggest" mode before "auto" mode
- **Date:** 2026-10-19 · **Status:** Proposed
- **Context:** Trust. Wrong autonomous messages to patients are costly.
- **Decision:** Every autonomous policy has a mode: `off`, `suggest` (the doctor approves) or `auto`.

## ADR-006: Bangladesh as launch market
- **Date:** 2026-10-19 · **Status:** Accepted
- **Context:** The first user operates in Bangladesh, at one location.
- **Decision:** Timezone `Asia/Dhaka`. `+880` phone numbers. Bangla and English templates. A local SMS gateway as primary provider. Host in the Singapore region.
- **Consequences:** Sender-ID registration lead time. Bangla SMS costs more parts per message. SMS text must make sense without a link, for feature-phone users.

## ADR-007: Role-based access (doctor, assistant)
- **Date:** 2026-10-19 · **Status:** Accepted
- **Context:** The doctor's assistant needs access to run day-to-day logistics.
- **Decision:** A `staff` table with roles and per-permission overrides. The doctor controls settings and staff. The assistant manages schedule and patients and can send signals on the doctor's behalf. Every action records its actor.
- **Consequences:** Authorization checks (Postgres row-level security) are needed from Phase 1.

## ADR-008: Doctor-configurable procedures with protocol snapshots
- **Date:** 2026-10-19 · **Status:** Accepted
- **Context:** Procedure types and follow-up schedules must not be fixed in code. The doctor edits them.
- **Decision:** Procedures and protocols are data, editable in-app, archived instead of deleted. Each surgery booking stores a frozen JSON snapshot of its protocol.
- **Consequences:** Edits never silently reshuffle existing patients' follow-ups. Re-planning an existing patient is an explicit action.
