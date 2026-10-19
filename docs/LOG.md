# Project Log

Newest entries at the top. Each entry: what was done, decisions made, what's next.

---

## 2026-10-19: Requirements update
**Decided**
- Launch market is **Bangladesh** (ADR-006).
- The doctor's **assistant** gets their own login with a restricted role (ADR-007).
- **One** operating location.
- Procedure types and follow-up protocols are **configurable by the doctor in-app**, with snapshots so edits don't affect existing patients (ADR-008).

**Updated**
- `docs/PLAN.md`: locale, roles table, Bangladesh specifics (§3.1), data model, roadmap phases 1–3, open questions.

**Next**
- Discuss the remaining open questions in PLAN.md §7 with dad.
- Shortlist and price-compare Bangladeshi SMS gateways. Start the sender-ID registration.
- Start Phase 0.

## 2026-10-19: Project kickoff and planning
**Done**
- Defined the vision: autonomous appointment logistics for a surgeon working Fri/Sat sessions.
- Wrote `docs/PLAN.md` (concepts, stack, flows, data model, roadmap, risks).
- Recorded the initial architecture decisions in `docs/DECISIONS.md` (ADR-001 to ADR-005, all *Proposed*).

**Next**
- Answer the open questions in PLAN.md §7 (country/SMS provider, assistant access, current booking process, procedure list with follow-up protocols).
- Accept or revise the ADRs.
- Start Phase 0: scaffold the Next.js app, Supabase project, CI and first deploy.
