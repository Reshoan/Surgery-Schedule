# Surgery Schedule: Project Plan

> Working name. Status: **Planning** (2026-10-19)

## 1. Vision

An appointment system that **runs the logistics by itself**. The doctor makes clinical decisions and taps a button now and then. The system handles delays, rescheduling, follow-up planning and patient communication.

**First user:** a surgeon who operates on Fridays and Saturdays, outside his regular practice.
**Later:** patients get their own view, then other doctors and clinics.

## 2. Core concepts

These ideas shape the whole design, so they come before the stack.

### 2.1 The live queue and projected times
Each surgery day is a **queue**, not a fixed grid. Every appointment has two times:
- `scheduled_at` is what was booked.
- `projected_at` is when it will actually start, recomputed whenever something changes.

`projected_at` is derived from: whether the doctor has arrived, how long the current case is taking, and each procedure's typical duration. All the autonomy builds on this one number. When it drifts too far from `scheduled_at`, the system acts.

### 2.2 How the system knows what is happening (doctor signals)
The system can't read minds, so the doctor's input must be **one tap**:
- "Running late: +15 / +30 / +60"
- "Arrived", "Started case", "Finished case"
- Later, optionally: phone geolocation/ETA for automatic lateness detection

Missing signals are signals too. If the doctor hasn't tapped "Arrived" 10 minutes before the first case, the system assumes a delay and asks him.

### 2.3 Policies (the rules behind the autonomy)
Each behaviour is a rule that can be configured, not hard-coded:
| Policy | Example default |
|---|---|
| Delay notify threshold | Notify a patient if `projected_at` slips more than 20 min |
| Hysteresis | Don't re-notify unless it moves another 15 min (avoid SMS spam) |
| Patient response window | Wait 15 min for "wait / reschedule"; default = wait |
| Early-offer | When a slot opens, offer it to the next patients in order; first to accept wins; offer expires in 10 min |
| Follow-up tolerance | Each follow-up has a window (e.g. 1 month ± 5 days) |

### 2.4 Procedure templates and follow-up protocols
Each procedure type has:
- typical duration (plus buffer)
- a **follow-up protocol**, e.g. Operation A means 3 check-ups at +1, +2 and +3 months, each with a ±N-day window and a check-up duration

After a surgery is marked done (or booked), the scheduler **proposes the whole follow-up chain**. For each visit it finds the best free slot inside the window. If the window is full, it **suggests the nearest slot outside the window** and flags it for the doctor to approve.

### 2.5 Everything is an event
Every change ("doctor late 30m", "patient chose reschedule", "SMS delivered") is stored as an event. That gives:
- an **audit trail**, which medical software needs
- the project record of *what the system did and why* ("SMS sent because projected start moved +34 min")
- a clean trigger for automation

## 3. Recommended tech stack

Guiding principle: **one language and as few moving parts as possible.** Lean on managed services, because one person has to build and run this.

| Layer | Choice | Why |
|---|---|---|
| Language | **TypeScript** everywhere | One language for UI, API, scheduler and workflows. Shared types. |
| App (doctor and patient UI) | **Next.js** (App Router) as an installable **PWA** | Works on the doctor's phone like an app, with no app store. The patient portal lives in the same codebase. |
| UI kit | Tailwind and shadcn/ui | Fast to build, looks clean |
| Database | **PostgreSQL** (via **Supabase**) | Relational data, transactions, and an *exclusion constraint* on time ranges that makes double-booking **impossible at the database level** |
| Realtime | **Supabase Realtime** | The dashboard updates instantly when anything changes, with no polling |
| Auth | Supabase Auth: email or passkey for doctors, **signed magic links** for patients | Patients need no account. The SMS link is their access. |
| Autonomy engine | **Inngest** (durable workflows) | Built for "send SMS, then *wait up to 15 min for a reply*, else default". It has durable sleeps, `waitForEvent`, retries and idempotency. Workflows survive restarts and deploys. |
| SMS | Provider adapter: a **local aggregator** for the main country, Twilio as fallback | Local gateways are much cheaper and have better delivery. Keep it behind an interface so providers can be swapped. |
| Scheduling core | Plain TypeScript **pure functions** with heavy unit tests | For one doctor's calendar, a greedy slot-finder with scoring is enough. No solver is needed yet. |
| Hosting | Vercel (app) + Supabase (DB) + Inngest Cloud | Atomic deploys with instant rollback. Managed Postgres with point-in-time recovery. Nothing to babysit. |
| Monitoring | Sentry (errors) + an uptime check + a dead-letter alert for SMS that failed to send | You learn about failures before patients do |
| Testing | Vitest (scheduler logic), Playwright (key flows) | The scheduler and policies are where bugs hurt most |

### Alternatives considered
- **Self-hosted Postgres + Graphile Worker/pg-boss** instead of Supabase + Inngest. Cheaper at scale, with no vendor lock-in. Costs more ops work. A reasonable later migration because it is all still Postgres.
- **Temporal** for workflows. More powerful, but heavy for one person.
- **React Native / Expo app.** Not needed at first. A PWA plus SMS links reaches every patient without an install. Revisit when push notifications to patients matter.
- **Firebase.** Realtime is good, but a NoSQL store makes scheduling constraints and reporting harder. Postgres fits this domain better.

### About "zero downtime"
For a project this size that means:
1. Managed hosting with atomic deploys and instant rollback
2. **Backward-compatible database migrations** (add first, remove later: "expand/contract")
3. **Durable workflows**, so a pending "wait 15 min then send SMS" survives a deploy or crash
4. **Idempotent messaging**, so a retry never sends a patient two SMS
5. Fallback: if the system can't reach the SMS provider, it alerts the doctor or assistant

Realistic target: 99.9%+ for the app, and **no lost or duplicated notifications**. The second matters more.

## 4. Key flows

### Flow A: Doctor is late
1. Doctor taps "Running late +30" (or misses the arrival check-in and the system asks him).
2. The system recomputes `projected_at` for the whole day's queue.
3. For each patient whose delay is over the threshold: SMS with the new estimated time and a link offering **Wait** or **Reschedule**.
4. Workflow waits for the response (timeout = Wait).
5. If someone chooses Reschedule: they get slot suggestions, and the freed slot triggers Flow B.

### Flow B: Slot frees up (early-offer cascade)
1. Slot opens (cancellation, reschedule or no-show).
2. The system finds the next patients who could come earlier.
3. SMS offer: "A slot opened at 5:30. Come early?" First to accept gets it. The slot is held atomically in a DB transaction.
4. The offer expires. Then the next candidate gets it, or the gap stays as a buffer.

### Flow C: Follow-up chain
1. Surgery booked or completed, so the protocol for that procedure is loaded.
2. The scheduler places each follow-up inside its window, scoring for preferred times and the doctor's load.
3. Conflicts produce "best alternative" suggestions for the doctor to approve.
4. Patient gets a confirmation SMS with the full plan and later reminders.

## 5. Data model (first draft)

```
clinics            (multi-tenant from day 1, cheap now, painful to add later)
doctors            -> clinic
availability       (doctor, weekday, start, end; e.g. Fri/Sat sessions)
availability_exceptions (holidays, one-off changes)
procedure_types    (name, duration, buffer)
followup_protocols (procedure_type, step #, offset, window_before, window_after, duration)
patients           (name, phone, language preference)
appointments       (doctor, patient, type: surgery|followup, procedure_type,
                    scheduled_range tstzrange, projected_at, status, parent_appointment_id)
                    EXCLUDE USING gist (doctor_id WITH =, scheduled_range WITH &&)
day_sessions       (doctor, date, actual_arrival, state)
events             (append-only: what happened, who/what caused it, payload)
notifications      (appointment, channel, template, status, provider_id, idempotency_key)
offers             (slot, patient, expires_at, status)
policies           (per-doctor overrides of defaults)
```

## 6. Roadmap

Each phase ends with something the doctor can actually use.

| Phase | Deliverable | Done when |
|---|---|---|
| **0. Foundations** | Repo, docs, CI, Supabase project, deploy pipeline, empty app live | A deploy to prod works and can be rolled back |
| **1. Manual scheduling** | Doctor login, Fri/Sat availability, procedure types, patient list, book/move/cancel, day and week view (realtime) | Dad uses it instead of paper or his current method for one weekend |
| **2. Follow-up autopilot** | Protocols per procedure, auto-proposed follow-up chains, conflict suggestions | Booking a surgery generates the full follow-up plan |
| **3. SMS basics** | Provider adapter, confirmations and reminders, delivery tracking, Bangla/English templates | Patients get reliable reminders |
| **4. Live day mode** | One-tap doctor signals, projected times, delay SMS with Wait/Reschedule link | A late arrival notifies patients without anyone lifting a finger |
| **5. Early-offer cascade** | Slot-release offers, atomic acceptance, expiry | A reschedule auto-fills from the queue |
| **6. Patient portal** | Magic-link page: my appointments, live status, follow-up plan, reschedule | Patients self-serve |
| **7. Multi-doctor** | Onboarding, per-clinic settings, billing if needed | A second doctor uses it |
| **Later** | AI parsing of free-text SMS replies (incl. Bangla), geolocation ETA, WhatsApp channel, analytics (typical overrun per procedure, used to improve projections) | |

## 7. Risks and open questions

**Open questions (need answers before Phase 1 and Phase 3):**
1. **Country and SMS provider.** The Fri/Sat weekend suggests Bangladesh or the Gulf. This decides the SMS provider, sender-ID rules, Unicode (Bangla) SMS cost, and data-protection rules.
2. Does dad have an **assistant** who should also have access?
3. How does booking happen today: phone calls, walk-ins, referrals? Who enters patients into the system?
4. What are the real procedures, durations and follow-up protocols? Get the list from dad. It becomes the seed data.
5. Are surgeries at one location or several hospitals? This affects travel time and buffers.
6. Can patients reply to SMS, or only tap links? **Links are more reliable.** Two-way SMS needs extra setup in many countries.

**Risks:**
- **Medical privacy.** Keep diagnoses and procedure names *out of SMS*. Use "your appointment with Dr. X". Encrypt at rest, keep an audit log, and give patients signed, expiring links.
- **Over-automation.** Early on, run new autonomous behaviour in **"suggest" mode** (the doctor approves) before switching it to **"auto" mode**. Build trust first.
- **SMS spam and cost.** Hysteresis and per-patient rate limits.
- **Doctor adoption.** The one-tap UX must be faster than calling the patient himself, or it won't be used.

## 8. Record keeping

- `docs/PLAN.md`: this file, the living plan. Update it when scope changes.
- `docs/DECISIONS.md`: Architecture Decision Records (why X over Y).
- `docs/LOG.md`: dated progress journal (what was done, what's next).
- Git history: small commits with clear messages.
- In-app `events` table: a record of everything the *system* did at runtime.
