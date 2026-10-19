# Surgery Schedule

Autonomous appointment system for a surgeon (Fri/Sat surgery sessions). See `docs/PLAN.md`.

## Record keeping (always)
- After any meaningful work, add a dated entry at the top of `docs/LOG.md` (done / decisions / next).
- Architectural choices go in `docs/DECISIONS.md` as a new ADR. Never rewrite an accepted ADR. Supersede it instead.
- If scope or roadmap changes, update `docs/PLAN.md`.

## Principles
- Scheduling logic lives in pure, unit-tested functions.
- Every autonomous action writes an `events` row explaining why it happened.
- No diagnoses or procedure names in SMS text.
- Migrations must be backward compatible (expand/contract).
