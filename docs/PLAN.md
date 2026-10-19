# Surgery Schedule: Project Plan

> Working name. Status: **Planning** (2026-10-19)

## 1. Vision

An appointment system that **runs the logistics by itself**. The doctor makes clinical decisions and taps a button now and then. The system handles delays, rescheduling, follow-up planning and patient communication.

**First user:** a surgeon in **Bangladesh** who operates on Fridays and Saturdays at **one location**, outside his regular practice. His **assistant** also uses the system.
**Later:** patients get their own view, then other doctors and clinics.

**Locale:** timezone `Asia/Dhaka` (UTC+6, no daylight saving). Phone numbers in `+880` format. UI and SMS in Bangla and English.

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
**Nothing here is hard-coded.** The doctor creates, edits and archives procedure types and their follow-up protocols in a settings screen. The app ships with no fixed list.

Each procedure type has:
- typical duration (plus buffer)
- a **follow-up protocol**, e.g. Operation A means 3 check-ups at +1, +2 and +3 months, each with a ±N-day window and a check-up duration

After a surgery is marked done (or booked), the scheduler **proposes the whole follow-up chain**. For each visit it finds the best free slot inside the window. If the window is full, it **suggests the nearest slot outside the window** and flags it for the doctor to approve.

**Editing a procedure never changes existing patients' plans.** When a surgery is booked, the system saves a *snapshot* of that procedure's protocol with the appointment. Later edits apply only to new bookings. The doctor can still re-plan a specific patient by hand.

Protocol editor needs:
- offsets in days, weeks or months ("every 2 weeks", "monthly ×3")
- number of follow-ups
- tolerance window per step
- check-up duration
- an optional note per step ("remove stitches")

### 2.5 Roles: doctor and assistant
| Action | Doctor | Assistant |
|---|---|---|
| View schedule, book/move/cancel appointments | ✅ | ✅ |
| Manage patients | ✅ | ✅ |
| Send "doctor running late" / "arrived" signals | ✅ | ✅ (on the doctor's behalf) |
| Approve suggestions in "suggest" mode | ✅ | ✅ (configurable) |
| Edit procedures, protocols, availability, policies | ✅ | ❌ by default (doctor can grant) |
| Invite or remove staff | ✅ | ❌ |

Every action records **who** did it, so the audit trail shows whether the doctor, the assistant or the system made a change.

The assistant is also the **human fallback** for the autonomy. Patients who don't respond to SMS or links (for example on feature phones) show up in an "Needs a call" list for the assistant.

### 2.6 Everything is an event
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
| SMS | Provider adapter with a **Bangladeshi SMS gateway** as primary (see §3.1) | Local gateways are far cheaper than Twilio for `+880` numbers and deliver better. Keep it behind an interface so providers can be swapped. |
| Scheduling core | Plain TypeScript **pure functions** with heavy unit tests | For one doctor's calendar, a greedy slot-finder with scoring is enough. No solver is needed yet. |
| Hosting | Vercel (app) + Supabase (DB) + Inngest Cloud, in the **Singapore region** | Closest major region to Bangladesh, so the app feels fast. Atomic deploys with instant rollback. Managed Postgres with point-in-time recovery. Nothing to babysit. |
| Monitoring | Sentry (errors) + an uptime check + a dead-letter alert for SMS that failed to send | You learn about failures before patients do |
| Testing | Vitest (scheduler logic), Playwright (key flows) | The scheduler and policies are where bugs hurt most |

### 3.1 Bangladesh specifics
- **SMS gateway.** Compare local providers (e.g. SSL Wireless, Alpha SMS, BulkSMSBD, MiMSMS) on price, API quality, delivery reports and reliability before Phase 3. Test real delivery to Grameenphone, Robi, Banglalink and Teletalk numbers.
- **Sender ID.** A branded sender name ("masking") needs registration through the gateway and takes time. Start the paperwork early. Non-masking (sent from a number) is cheaper and works for a pilot.
- **Bangla SMS cost.** Unicode SMS fits about 70 characters per part, versus 160 for English. Keep Bangla templates short, or a single message can cost 2–3 parts.
- **Feature phones.** Not every patient can open a link. Every SMS must make sense on its own ("Dr. X is running ~30 min late. New time ≈ 5:30 PM."). The link is only for actions. Patients who don't act are passed to the assistant (§2.5).
- **Privacy.** Follow Bangladesh's data-protection rules as they stand at launch. Check their current status before onboarding real patients.
- **Payments (later).** If billing other doctors ever matters, bKash or SSLCommerz, not Stripe.

### 3.2 Alternatives considered
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
staff              (user, clinic, role: doctor|assistant, permissions)
availability       (doctor, weekday, start, end; e.g. Fri/Sat sessions)
availability_exceptions (holidays, one-off changes)
procedure_types    (name, duration, buffer, archived_at)       <- doctor-editable
followup_protocols (procedure_type, step #, offset + unit (days|weeks|months),
                    window_before, window_after, duration, note) <- doctor-editable
patients           (name, phone, language preference)
appointments       (doctor, patient, type: surgery|followup, procedure_type,
                    protocol_snapshot jsonb,   <- frozen copy at booking time
                    scheduled_range tstzrange, projected_at, status, parent_appointment_id,
                    created_by)
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
| **1. Manual scheduling** | Doctor and assistant logins with roles, Fri/Sat availability, **procedure types editor**, patient list, book/move/cancel, day and week view (realtime) | Dad and his assistant use it instead of their current method for one weekend |
| **2. Follow-up autopilot** | **Follow-up protocol editor**, auto-proposed follow-up chains, protocol snapshots, conflict suggestions | Booking a surgery generates the full follow-up plan from protocols dad defined himself |
| **3. SMS basics** | Bangladeshi gateway adapter, confirmations and reminders, delivery tracking, Bangla/English templates, "Needs a call" list for the assistant | Patients get reliable reminders |
| **4. Live day mode** | One-tap doctor signals, projected times, delay SMS with Wait/Reschedule link | A late arrival notifies patients without anyone lifting a finger |
| **5. Early-offer cascade** | Slot-release offers, atomic acceptance, expiry | A reschedule auto-fills from the queue |
| **6. Patient portal** | Magic-link page: my appointments, live status, follow-up plan, reschedule | Patients self-serve |
| **7. Multi-doctor** | Onboarding, per-clinic settings, billing if needed | A second doctor uses it |
| **Later** | AI parsing of free-text SMS replies (incl. Bangla), geolocation ETA, WhatsApp channel, analytics (typical overrun per procedure, used to improve projections) | |

## 7. Risks and open questions

**Answered (2026-10-19):**
- Country: **Bangladesh**. See §3.1.
- Assistant: **yes**, with their own login. See §2.5.
- Locations: **one**. No travel-time logic needed. (`clinics` still allows more later.)
- Procedures: **set by the doctor in the app**, not hard-coded. See §2.4.

**Still open (to discuss with dad):**
1. How does booking happen today: phone calls, walk-ins, referrals? Who enters patients: dad, the assistant or both?
2. What should the assistant **not** be able to do or see? Check the defaults in §2.5.
3. Roughly what share of patients use smartphones versus feature phones? This decides how much we rely on links.
4. How late does he typically run, and how far ahead should patients be told? This sets the default policy thresholds.
5. Bangla, English, or both in SMS? Per patient?
6. A few example procedures with their follow-up schedules, so the protocol editor can be tested against real cases.

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
