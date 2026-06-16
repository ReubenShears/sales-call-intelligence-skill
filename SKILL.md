---
name: sales-call-intelligence
description: >
  Turn a single sales/discovery call transcript into a fully structured row in the
  Optimally Baserow "Sales Call Data" table plus linked "Objection Data" rows. Use
  whenever a Fathom (or any) transcript of a SALES or DISCOVERY call needs to be
  analysed and logged for sales-intelligence KPIs. The input is a transcript (+ light
  metadata); the output is one parent Sales Call row, N linked Objection rows, and a
  link to the owning client in Client Data. Skips internal/fulfilment/webinar-logistics
  calls. Accepts a Fathom URL and/or a raw transcript, and is idempotent/update-based —
  re-running on the same call (keyed on Recording ID) updates the existing row instead of
  duplicating. Two soft metrics (Rep Talk %, Framework Adherence) are computed by FIXED
  deterministic formulas so every call scores consistently.
---

# Sales Call Intelligence

Extract one sales/discovery call into Baserow as structured, KPI-ready data.

## Inputs

The routine fires once per finished sales call and passes ANY of:
- `transcript` — speaker-labelled text (`[MM:SS] Speaker Name: ...`); may be raw/auto-diarised.
- `fathom_url` and/or `recording_id` — a Fathom call link or numeric id.
- `call_title`, `call_date` (ISO), `owner_hint` — optional metadata.

**Resolve the input to a transcript + metadata before doing anything else:**
- If a `fathom_url` is given → `get_recording_by_url` (returns recording_id, title, date, url) → then `get_meeting_transcript(recording_id, url)`.
- If only a bare numeric id → `get_recording_by_call_id` (fallback: retry the number as a recording_id) → `get_meeting_transcript`.
- If only raw `transcript` text is passed → use it directly with the supplied `call_title` / `call_date`.
Always capture the `recording_id` and canonical recording URL — they are the **idempotency key** and the `Call Recording Link`.

Tools to load (ToolSearch): Fathom `get_recording_by_url`, `get_recording_by_call_id`, `get_meeting_transcript`; Baserow `list_table_rows`, `create_rows`, `update_rows`, `delete_rows`.

## Baserow targets (workspace 453125)

- **Sales Call Data** — table `1028550` (parent, one row per call)
- **Objection Data** — table `1028590` (child, one row per objection; link field `Sales Call` → parent)
- **Client Data** — table `1000911` (link target for `Client Data` on the parent)

**Field-type rule (critical):** every formerly-single/multi-select field is now **plain text**
(the Baserow MCP cannot create select options). Write strings, NOT option objects. Always
write values from the **fixed vocabularies** below so rollups stay clean. Dates are ISO
(`YYYY-MM-DDTHH:MM:SSZ`). Booleans are real `true`/`false`. Numbers are plain integers.

---

## Step 0 — Idempotency / upsert (run BEFORE writing)

The routine is **update-based** and safe to fire repeatedly on the same call. The unique key is
`Recording ID` (stable per Fathom call). Before creating anything:

`list_table_rows(1028550, search="<recording_id>")`
- **No match** → this is a new call: CREATE (Step 9 create path).
- **Match found** → this is a re-run: UPDATE the existing parent row in place via
  `update_rows(1028550, [{ "id": <existing id>, ...all fields }])`, and REFRESH its objections —
  fetch the objection rows linked to it (`list_table_rows(1028590, search="<recording_id or Call Name>")`),
  `delete_rows(1028590, [those ids])`, then recreate from the current analysis. Never create a
  second parent row for a Recording ID that already exists.

If `Recording ID` is unavailable (raw-transcript-only input with no id), fall back to matching on
`Call Name` + `Call Date`; if still ambiguous, create and flag for human dedup.

## Step 0a — Is this call in scope?

Only process calls where **someone is SELLING a service to a prospect** (discovery or sales).
**SKIP and report "out of scope"** if the call is any of:
- Internal team / strategy / partner sync (e.g. recurring "X Call 1/2" working sessions)
- Client fulfilment / onboarding / check-in with an *existing* client
- Webinar delivery, breakout-room logistics, or waiting-room routing
- Pure tech setup / software-access calls

Decision test: *Is there a prospect being qualified and/or pitched an offer, with a buy/no-buy
question in play?* If no → skip. If unsure → skip and flag for human review (do not write a row).

## Step 1 — Identify the Owner Account (who the pipeline belongs to)

The owner is **the selling side**, not the prospect.
- Optimally's own sales call → owner = **Optimally Systems**.
- A client's sales call we're ingesting as a service → owner = **that client**.

Resolve to a Client Data row id: `list_table_rows(1000911, search="<owner name>")`, take the
matching row `id`, and set the parent `Client Data` link to `[that_id]`. Optimally Systems is
currently row **38**. If no client match is found, do not guess — flag for human review.

## Step 2 — Core / identity fields

`Call Name` (title), `Prospect Name`, `Company`, `Role`, `Call Date` (ISO),
`Duration (min)` (last timestamp → minutes, rounded), `Rep / Closer` (the seller, text),
`Lead Source` (vocab), `Call Type` (vocab), `Call Recording Link` (the Fathom/recording URL),
`Recording ID`, `Raw Transcript` (store the full cleaned transcript — always).

## Step 3 — Prospect snapshot (extract from discovery; text fields)

`Annual Revenue`, `Profit`, `Current Offer`, `Client Acquisition Method`,
`Client Flow (per mo)`, `LTV`, `Company Age (yrs)`, `Market Sophistication` (vocab),
`Stated Goal`, `Qualified?` (vocab).
If a value is not stated, write `"Not stated"` — never invent figures.

## Step 4 — Stage-by-stage (maps to Optimally's 4-part framework)

Use these EXACT definitions so booleans are consistent.

**Intro**
- `Intro Notes` — rapport + framing summary.
- `Frame Set?` = true if the rep states the agenda / takes control of the call ("objective of the call is…").

**Discovery**
- `Why They Booked`, `VSL Watched?` (`Yes` / `No` / `Watched on call`), `VSL Takeaways`,
  `Discovery Notes`, `Core Problem`, `Secondary Problems`, `Problem Implications & Personal Stakes`.
- `Permission to Pitch Given?` = true if rep asks permission to present and prospect agrees.
- `Scarcity Used?` (text) — `"Yes — <how>"` or `"No"`.

**Offer / Demo**
- `Pitch Notes`, `Custom Demo Shown?` (true if a tailored demo/audit was presented),
  `Deliverables Presented`, `Pre-objection Handled?` (true if rep pre-empts a likely objection),
  `Pre-objection Details`.

**Close & Onboard**
- `Close Notes`, `Asked for the Sale?` (true if rep makes ≥1 explicit ask),
  `Times Asked for Sale` (count distinct explicit asks/closes),
  `Referral Asked?`, `Referrals Given`,
  `Payment Collected on Call?` — **true if a payment link is sent/accepted on the call OR
  the prospect commits to pay on the call** (link-sent-on-call counts as collected),
  `Onboarding Started on Call?` (Slack/onboarding form/next steps initiated live),
  `First Call Booked?` (true only if a concrete next call time is set, not just "send your link").

## Step 5 — Outcome

`Outcome` (vocab), `Deal Value`, `Cash Collected`,
`Prospect Temperature (0-10)` — use the prospect's self-rating if asked; else infer 0–10 from
buying signals, defaulting to a conservative read.
`Next Step`, `Next Step Owner`, `Next Step Date` (ISO).

## Step 5b — Multi-call deals (prevent double-counting)

Each call is its own row, grouped into a deal via these fields on `Sales Call Data`:
- `Related Calls` — self-link to the other call(s) in the same deal.
- `Call Number` — 1, 2, 3…
- `Deal Stage` — `Single-call close` · `Call 1 — discovery` · `Call 2 — close` · `Follow-up`.

**Continuation detection.** Treat the call as a continuation if EITHER:
- the API passes a `previous_call_id` (preferred — zero guessing), OR
- the same prospect (name/company, same owner) already has an open row AND the transcript
  references a prior conversation ("as we discussed last call", "since we last spoke").

**The anti-double-count rule (non-negotiable):**
- A deal's `Outcome` = `Closed-won`/`Lost`, plus `Deal Value` and `Cash Collected`, live ONLY
  on the call where money was actually committed/paid. Money was never paid on call 1 if they
  didn't close on call 1 — so call 1 must NOT carry a deal value or a won outcome.
- On creating a closing call (Call 2+): set this row's Outcome/Deal Value/Cash, link `Related Calls`
  to the earlier row(s), and **update the earlier row(s)** to `Outcome = Follow-up`, `Deal Value`
  empty, `Cash Collected` empty. Close-rate then counts each deal exactly once.
- The on-call booleans (`Payment Collected on Call?`, `Onboarding Started on Call?`,
  `First Call Booked?`) describe the call they happened on — leave them as-is per call.

**Single-call default:** if not a continuation, set `Call Number` = 1 and `Deal Stage` =
`Single-call close` (or `Call 1 — discovery` if it explicitly ends with a second call booked
and no close). Leave `Related Calls` empty.

**Adherence for multi-call deals:** score each call's `Framework Adherence Score` per Step 6 on
its own. For deal-level reporting, a checkpoint counts as hit if ANY call in the thread hit it
(a dedicated closing call legitimately has no discovery checkpoints, so never judge a deal on a
single call in isolation).

## Step 6 — Deterministic metrics (DO NOT eyeball these)

**Rep Talk %** (`Rep Talk %`, integer):
1. Split transcript into speaker segments.
2. Sum word counts per speaker. `rep_words` = total for the Optimally closer(s); `total_words` = all speakers.
3. `Rep Talk % = round(rep_words / total_words * 100)`.
(The 80/20 ideal is *prospect* 80 / rep 20, so a low Rep Talk % is good.)

**Framework Adherence Score** (`Framework Adherence Score`, integer 0–100):
`round(hits / 12 * 100)` over this FIXED 12-point checklist (each 1 if true, else 0):
1. `Frame Set?` = true
2. `Why They Booked` captured (non-empty)
3. VSL gate enforced (`VSL Watched?` = `Yes` or `Watched on call`)
4. `Stated Goal` quantified (contains a target figure/outcome)
5. `Core Problem` AND `Problem Implications & Personal Stakes` both non-empty
6. `Permission to Pitch Given?` = true
7. `Scarcity Used?` starts with "Yes"
8. `Custom Demo Shown?` = true
9. `Pre-objection Handled?` = true
10. `Asked for the Sale?` = true
11. `Times Asked for Sale` ≥ 2 (the "ask more than once" rule)
12. `Referral Asked?` = true

Adherence measures rep *behaviour*, independent of whether the deal closed.

## Step 7 — Coaching fields (qualitative)

`Summary` (3–4 sentences), `What Went Well`, `What to Improve` (call out missed checklist
items from Step 6), `Biggest Blocker` (the single thing most in the way of the sale),
`What Would Unblock` (and whether it was delivered), `Notable Quote` (verbatim, prospect).

## Step 8 — Objections → child rows

For every distinct objection/hesitation the prospect raises, create one row in table `1028590`:
- `Objection` (short label, primary), `Sales Call` = `[parent_row_id]`,
  `Objection Type` (vocab), `Verbatim Quote`, `Stage Arose` (vocab),
  `Severity (1-3)` (1 minor … 3 deal-threatening), `Resolved?` (bool), `How Handled`.
Negotiation asks (e.g. payment-plan requests) are NOT objections — note them in `Close Notes` instead.

## Step 9 — Write order

**New call (no existing Recording ID):**
1. `create_rows(1028550, [parent])` → capture returned `id`.
2. `create_rows(1028590, [objections with "Sales Call": [parent_id]])`.

**Re-run (existing row found in Step 0):**
1. `update_rows(1028550, [{ "id": existing_id, ...all fields }])`.
2. `delete_rows(1028590, [linked objection ids])`, then `create_rows(1028590, [...])` with `"Sales Call": [existing_id]`.

Never write select-option objects; always plain strings/booleans/numbers. Link fields take id arrays
(`"Client Data": [38]`, `"Sales Call": [parent_id]`).

---

## Fixed vocabularies (write these exact strings)

- **Call Type:** `Discovery` · `Sales` · `Re-engagement` · `Follow-up`
- **Lead Source:** `Webinar` · `Cold call` · `Cold email` · `Cold social/DM` (e.g. Instagram outreach) · `Referral` · `Organic/Inbound` · `Paid ad` · `Other`
  - **Default-to-cold rule:** Optimally does essentially no inbound. If a DM, email, or "reached out / got in touch" is mentioned as the source, classify it as COLD (`Cold social/DM` for DMs, `Cold email` for email) — NOT `Organic/Inbound`. Only use `Organic/Inbound` when the prospect unmistakably came in unprompted (e.g. "I found you and booked", "saw your content and reached out to you"). When in doubt, it's cold.
- **Outcome:** `Closed-won` · `Verbal yes` · `Follow-up` · `Lost` · `No-show` · `Disqualified`
  (append a short parenthetical for nuance, e.g. `Closed-won (paid on call)`)
- **Qualified?:** `Yes` · `No` · `Borderline`
- **VSL Watched?:** `Yes` · `No` · `Watched on call`
- **Market Sophistication:** `Low` · `Medium` · `High`
- **Objection Type:** `Price` · `No budget/cashflow` · `Trust & risk` · `Internal approval/compliance` ·
  `Timing/long cycle` · `Already have a solution` · `Will-it-work-for-me/niche` · `Effort/difficulty` ·
  `Need to consult someone` · `Results doubt` · `Other`
- **Stage Arose:** `Intro` · `Discovery` · `Pitch` · `Close`

## Golden reference (validated example)

`Megan Camille & Reuben Discovery Call` (Fathom 386929708, owner = Optimally Systems):
Closed-won (paid on call), Temperature 8, Adherence 83 (10/12 — missed VSL gate + referral ask),
biggest blocker = trust/female-proof (not price), 5 objections (Price, No budget/cashflow,
Trust & risk, Will-it-work-for-me/niche, Timing/long cycle), Call Number 1, Deal Stage
"Single-call close". Use this as the quality bar.
