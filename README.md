# sales-call-intelligence (skill)

Public skill for the Optimally sales-call-intelligence routine. Given a Fathom call URL
and/or a raw transcript, it extracts a structured Sales Call row + linked Objection rows
into Baserow, with deterministic Rep Talk % and Framework Adherence scoring. Idempotent
(keyed on Recording ID). The routine downloads `SKILL.md` from this repo at runtime.

See `SKILL.md` for the full specification.
