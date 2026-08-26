---
name: keep-warm
description: Weekly keep-warm nurture engine for your sales pipeline. Scans fresh material (via the last30days research engine) for each contact in the keep-warm funnel, then drafts contextual, no-ask share notes in your voice ("here's what this is and why it's interesting to you"). Draft-only, nothing ever auto-sends. Use for "/keep-warm", "keep warm run", "warm the pipeline", "nurture the prospects", "weekly nurture pass".
---

# Keep Warm

Weekly nurture engine for prospects and quiet clients. It fuses three sources:

1. **The last30days research engine** (github.com/mvanhorn/last30days-skill) finds genuinely fresh, relevant material (YouTube, articles, posts, discussions) per contact.
2. **Roman Olivera's pipeline boundaries** (Orban Labs): one core pipeline (roster → research → script → deliver), each stage with its own persisted artifact so it can be checked and retried alone, evidence kept separate from the message, and **render-from-approval**: nothing goes out that a human has not approved.
3. **Sam McKenna's nurture play** (#samsales "Next Level Nurturing"): educate while they navigate internal meetings, give them ammo they can forward, use multiple channels, **make no ask whatsoever**, and put in effort other sellers won't.

Every run produces a **draft pack for review**. This skill NEVER sends anything: no emails, no DMs, no comments, no posts. Mail drafts may be created only when the user explicitly asks after reviewing a pack. **A run that ends with anything sent is a failed run.** Note: if your CRM key is write-capable, "read-only on the CRM" is a behavioral rule of this skill, not a credential guarantee. Keep it absolute.

> **ADAPT MARKERS.** Sections marked `[ADAPT]` reference one specific stack (Attio CRM, Gmail, local client folders). Swap in your own CRM query, mailbox, and paths. Everything else is stack-agnostic.

## Commands

All commands operate on the state files defined below. If state files are missing, run bootstrap first (see State).

- `/keep-warm` or `/keep-warm run` — full weekly pass, produces the draft pack.
- `/keep-warm status` — table of the roster: contact, company, track, cadence position, last confirmed touch, unanswered count, links shared, opt-out flag. Read-only.
- `/keep-warm sent <name> [date] [link]` — record that the user actually sent a draft (defaults: today, the links from that contact's latest pack draft). Advances `next_touch_index`, appends links to `links_shared`, sets `last_touch_at`, increments `consecutive_unanswered`. Does NOT change `last_real_conversation_at`: a send is one-way. A duplicate `sent` for the same contact and date is a no-op with a warning; an ambiguous name lists the matches and asks. This command and Stage 0 auto-confirmation are the ONLY ways cadence advances.
- `/keep-warm unsent <name> [date]` — revert a mistaken confirmation: append a reversal event to history (never silently rewrite it), decrement `next_touch_index` and `consecutive_unanswered` (both floored at 0), and remove that send's links from `links_shared`.
- `/keep-warm add <name> | <company> | <email or channel> | <context>` — add a manual roster contact. Match on normalized email first, else full name + company; if a matching contact exists, update it instead of duplicating and say so. Context text may include a deal event ("just had discovery call"), which sets `next_touch_index: 0`.
- `/keep-warm remove <name>` — set `opted_out: true` (never delete history). If the name matches more than one contact, list the matches and ask; never guess.
- `/keep-warm track <name> active|warm` — move a contact between tracks (same name-matching rules as remove).

## Tracks

Every contact is on one of two tracks:

- **`active`** — live proposals and live conversations. Touched on the weekly rhythm.
- **`warm`** — pure keep-warm: no live thread, just staying on their radar. Touched roughly monthly: a warm contact is **due** when their `last_touch_at` is 28+ days ago (or null).

Default assignment at roster derivation: a company with an open proposal, or any two-way activity in the last 21 days, defaults to `active`; everything else defaults to `warm`. A `track` value in roster.json always overrides the default, and `/keep-warm track` is how the user flips someone. Every weekly run processes all `active` contacts plus whichever `warm` contacts are due; warm contacts not yet due are listed one-line in the pack ("warm, next due <date>") so the funnel stays visible.

## State

Home: `~/.claude/state/keep-warm/` (create with `mkdir -p` on first use; runtime state never lives in the skill folder).

`roster.json` — manual adds, overrides, opt-outs:

```json
{ "version": 1,
  "contacts": [
    { "key": "jane.doe@example.com",
      "name": "Jane Doe", "company": "Example Advisory",
      "email": "jane.doe@example.com", "channel": "email",
      "source": "manual", "opted_out": false, "aliases": [], "track": "active",
      "topics": ["AI in advisory workflows"],
      "notes": "Referral intro Aug 2026" } ] }
```

`channel` is an enum, `email` or `linkedin-dm`, and roster.json is the single source of channel identity (a `linkedin-dm` contact means the user has an active DM thread with them). Never place a real prospect's details in this skill file; examples stay fictional.

`state.json` — per-contact cadence state, keyed by the same `key` (normalized primary email; if no email is known yet, `name|company` lowercased):

```json
{ "version": 1,
  "contacts": {
    "jane.doe@example.com": {
      "next_touch_index": 1,
      "last_real_conversation_at": "2026-08-24",
      "last_touch_at": "2026-08-24",
      "consecutive_unanswered": 0,
      "deferred_since": null,
      "links_shared": ["https://example.com/a"],
      "topics_used": ["AI in advisory workflows"],
      "history": [ {"date": "2026-08-24", "type": "sent", "play": "thankyou", "links": []} ] } } }
```

Rekey rule: when a real email becomes known for a `name|company` key, atomically rename the key in BOTH files, keep all history and `links_shared`, and record the old key in the roster entry's `aliases` so nothing strands under the dead key.

Field semantics: `next_touch_index` = position in the cadence ladder for the NEXT touch (0-based). `last_real_conversation_at` = last two-way exchange, updated ONLY by verified two-way activity (an inbound reply found in reconciliation, a meeting) or an explicit conversation event supplied via `/keep-warm add` — never by a send. `consecutive_unanswered` = confirmed sends since the last inbound reply; ANY detected inbound resets it to 0 and updates `last_real_conversation_at`. These are three independent fields; never derive one from another.

Bootstrap (first run, or a contact with no state entry): do NOT default to touch 0. Always seed at `next_touch_index: 1`; the evidence (latest call-note date in the client folder, the mailbox reconciliation below) decides only `last_real_conversation_at` — the best-evidenced date, or null plus a "history unknown" flag in the pack. `next_touch_index: 0` is reachable ONLY when the user supplies a deal event via `/keep-warm add`. If a state file exists but fails to parse, stop, report it, and ask before touching it; never regenerate over a corrupt file.

Run artifacts: each run writes to `~/.claude/state/keep-warm/runs/<YYYY-MM-DD>/` — `roster-snapshot.json` (Stage 1), `cadence-decisions.md` (Stage 1), `evidence.md` (Stage 2), `drafts.md` (Stage 3), `review.html` (Stage 4), and `manifest.json` (`{run_id, stages: {s0..s4: pending|done|failed}, timestamps}`). If the folder already exists, do not overwrite: use `<YYYY-MM-DD>-2`, `-3`, etc. A failed stage is retried alone from its inputs; never redo research because drafting failed.

## The funnel roster

The funnel = **live CRM open opportunities + manual roster**, minus opt-outs.

### CRM query `[ADAPT]` (Attio shown as the worked example; swap for your CRM's API)

1. Verify you are in the right workspace first (`GET https://api.attio.com/v2/self`, header `Authorization: Bearer $YOUR_CRM_KEY`); stop with a diagnostic on a mismatch.
2. Fetch pipeline entries: `POST https://api.attio.com/v2/lists/<your-pipeline-list>/entries/query` with body `{"limit": 50, "offset": <n>}`; loop while `len(data) == limit`. Per entry read the parent company id, stage title, close date, and comments.
3. Resolve stage titles dynamically from the CRM's attribute options rather than hardcoding them; match case-insensitively; an unrecognized title → include the entry and flag it in the pack rather than silently dropping it.
4. Resolve company names from the company records.

### Roster derivation

1. **Group ALL entries by company first** (a company may hold multiple linked entries, e.g. an entry plus a forecast placeholder; a placeholder is not a separate relationship).
2. **Inclusion policy per company:** include if it has ≥1 open-stage entry AND no won engagement in active delivery. A company with only won entries is a client, excluded by default (delivery IS the relationship touch), but flag it so the user can `/keep-warm add` them deliberately. Manual roster entries and opt-outs always override this policy.
3. **Resolve the recipient (per company → per person).** A roster row is a PERSON, not a company. Resolution order: (a) `roster.json` entry for that company; (b) named contacts in your client-context folder for that company `[ADAPT]`; (c) names in the CRM entry comments. Required to draft: name + at least one channel identity (email address, or an active DM thread). **If no recipient can be verified, produce no draft for that company; list it in the pack under "needs a contact" instead.** Multiple valid contacts at one company: pick the relationship owner (the person the user actually talks to); never draft to two people at the same company in one run.
4. **Context load per contact:** read your client-context folder for that company `[ADAPT]`. If none exists, fall back to the CRM entry fields your key can actually read plus mailbox thread history with that contact, and mark the contact **"thin context — verify before sending"** in the pack. Never claim to have checked a source your credentials cannot read.

## Weekly run

Run order, exactly: create the run folder + manifest → derive the **provisional roster** (CRM query + manual roster + every contact in the previous pack) → Stage 0 reconciliation over that provisional roster → Stage 1 cadence decisions persist the reconciled snapshot → Stages 2-4.

### Stage 0: Reconciliation (mandatory, before any drafting)

The skill cannot see what the user does manually, so every run starts by syncing state with reality `[ADAPT: Gmail shown; use your mailbox API]`:

1. **Unconfirmed drafts:** for each draft in the most recent pack not yet confirmed via `/keep-warm sent`, search the mailbox. **Auto-confirm ONLY when a Sent message to that recipient, dated after the pack was generated, contains at least one of the draft's normalized link URLs.** Then record it as sent (as `/keep-warm sent` would) and note "auto-confirmed from mailbox" in the pack. Any other outbound is just a manual touch (step 2), and the draft stays in the pack-header question: "did these go out? (`/keep-warm sent <name>` or ignore)". Never advance cadence on an unconfirmed draft.
2. **Activity sweep (idempotent):** per provisional-roster contact, search the mailbox both directions over a window starting at the later of that contact's `last_touch_at` and the previous run's timestamp (fall back to 14 days on first run). Process unique message IDs in chronological order, skipping any message already recorded in `history` or auto-confirmed in step 1 of this run. Outbound from the user → a manual touch: set `last_touch_at` AND increment `consecutive_unanswered` (a touch without an answer counts, however it was sent). Inbound from them → reset `consecutive_unanswered` to 0 and update `last_real_conversation_at`. Append every reconciled message to `history` with its date and message ID, so a repeated reconciliation is a no-op.
3. **Coverage honesty:** contacts with no email or whose thread lives outside the mailbox (LinkedIn DMs) get `activity: unknown` — see the Stage 1 guardrail; their state counters are treated as untrusted.

### Stage 1: Roster + cadence decisions

Build the roster (above). For each contact decide this week's **play** from the ladder using `next_touch_index`:

| next_touch_index | Play |
|---|---|
| 0 (only via user-supplied deal event) | Customised thank-you / understood note referencing the specific conversation the user named. Not a one-liner. |
| 1-2 | **Educational share** (the core play): one or two genuinely useful pieces, framed for them, zero ask. |
| 3 | **Proof touch**: a blind, verified proof point. Still no ask. |
| 4+ | Rotate: educational share, or a **channel switch** (a LinkedIn comment-tag script). |

Guardrails (checked in this order, each recorded in `cadence-decisions.md` with a reason):
- `opted_out` → skip.
- `warm` track and not due (last touch under 28 days ago) → skip with the "warm, next due <date>" line.
- `last_touch_at` within 5 days (confirmed or reconciled) → skip this week.
- `activity: unknown` → draft, but place it in a separate **"pending your call"** section of the pack that the user must explicitly clear; it never sits in the main send list. (Checked before the unanswered rule: an untrusted counter must not fire the gone-quiet flag.)
- `consecutive_unanswered >= 3` → do NOT draft; flag "gone quiet — the user decides" in the pack. (The unanswered counter is independent of the ladder position; a replied-to proof touch at index 3 continues to index 4, it is not blocked.)

Write `roster-snapshot.json` and `cadence-decisions.md`.

### Stage 2: Research (evidence pack)

1. **Topic allocation with guaranteed coverage.** Each active contact carries 2-3 nurture topics (from their context load or roster.json). Dedupe overlapping topics across contacts, then select up to 5 distinct topics per run such that **every drafting contact is covered by at least one selected topic**. If the cap forces exclusions, drop the topic whose contacts are already covered elsewhere; if a contact still ends up uncovered, defer that contact, record it under "deferred — no research this run" in the pack, and set `deferred_since: <run date>` on them in state.json. The next run must cover contacts with `deferred_since` set before any others, clearing the field once covered. A contact may also end a run with **"no qualified asset this week"** — that is a valid outcome; never draft from weak evidence to fill a slot.
2. **Pre-flight for the research engine:** check `~/.config/last30days/.env` exists and contains `SETUP_COMPLETE=true`. If not, stop Stage 2 and tell the user to run last30days setup; do not improvise research from plain web search without flagging it as a degraded run.
3. **Per-topic runs:** delegate each topic to a subagent (a cheaper model tier is fine for this) whose prompt requires it to (a) load the last30days skill and follow that skill's contract end-to-end as written, including its WebSearch pre-flight steps and its engine invocation with the `--agent` flag so it never pauses to ask the user questions; (b) after the last30days run completes (the subagent's own emitted response obeys that skill's contract in full), separately write a compact machine-consumed item list (title | URL | source | date | why it matters for this audience) to `runs/<date>/topic-<slug>.md` and report the path. keep-warm reads that file downstream; the last30days contract governs the subagent's run, the file is a distinct post-skill artifact. A subagent that returns a question, an error, or nothing = a Stage 2 failure for that topic; retry that topic only.
4. **The user's own material:** check your published-content folder `[ADAPT]` for anything shipped in the last 14 days that fits a contact's topics. Sam's rule: your own posts, articles and podcast clips are first-class nurture assets.
5. Write `evidence.md`: every candidate item with URL, source, date, one-line summary, and which contact(s) it could serve. Evidence is internal; source records never get pasted wholesale into a message.

**Selection bar:** an item earns a slot only if it is (a) recent — verify the date; undated vendor posts must be flagged as undated, never presented as "this week"; (b) specific to the contact's actual problem or industry; (c) something the user would genuinely find interesting themselves. One strong item beats three okay ones.

**Link dedupe (checked at FOUR points):** at evidence selection, at drafting, at pack assembly, and at mail-draft creation — each time against fresh `state.json` `links_shared` for that contact AND the current run's allocations. Normalize URLs before comparing (strip tracking params, trailing slashes). Never re-share a link to the same contact; avoid the same link to two contacts at the same company.

### Stage 3: Script (the contextual share note)

This is Roman's contextual-sharing move: the share arrives already explained. For each contact, draft in **the user's voice** (read your voice/style reference files first, every run `[ADAPT]`):

- **Shape:** 1-2 sentences of personal context (why this made me think of you / your situation), then the link(s), each with one or two sentences of **what it is + why it's interesting to them specifically**. Nothing generic.
- **Register:** plaintext email register. Short. Signed with the user's first name. Under 120 words.
- **The no-ask rule is absolute** (Sam): no meeting request, no proposal nudge, no calendar link in a nurture touch. The message must survive the test "could they read this and owe me nothing?"
- **Banned:** "chasing", "just following up", "quick question", "touch base", "circle back", markdown in email bodies, engagement-bait questions, any invented detail about the contact.
- **Truth gates:** Proof touches must quote an entry from your verified proof-points file `[ADAPT]` (each entry sourced and dated), blinded (no client name without recorded permission). No qualifying entry → do an educational share instead; never fabricate or estimate. Backward references ("the one I mentioned last week") are allowed ONLY when state.json shows a confirmed prior send containing that reference; otherwise script with no backward reference.
- **Channel fit:** per the roster `channel` field. Email default, with a subject line on every email draft. Comment-tag play: script the comment text. `linkedin-dm`: DM-length, under 60 words, no subject.

Run a de-slop pass on every draft (cut AI tells, filler, and anything the user wouldn't say out loud). Write `drafts.md`: per contact — name, company, play, channel, subject (email drafts), the draft, link(s), and one line of reasoning ("why this item for this person"). The subject travels unchanged into `review.html` and into any later mail-draft verification.

### Stage 4: Deliver-for-approval

Assemble `review.html`: run header (reconciliation questions from Stage 0 first), roster table (contact, track, play, status, flags), the drafts, flagged contacts (needs-a-contact, thin context, gone quiet, deferred, pending-your-call), and unused strong evidence. Show it to the user alongside the plaintext drafts.

Then STOP. Approval = the user sends manually and records it with `/keep-warm sent`, or explicitly instructs mail-draft creation: verify recipient address and subject against the pack before each draft-creation call, and after creation verify the result is a draft, not a sent message. Never call any send/reply/forward tool.

Cadence state advances ONLY through `/keep-warm sent` or Stage 0 auto-confirmation. Drafts that were never sent change nothing.

### Master doc (Sam's bonus move)

Maintain a nurture library file `[ADAPT: pick a home in your sales docs]`. Every confirmed-sent nurture note gets appended as a reusable pattern after this blinding transform: strip recipient name, company, industry-identifying details, deal stage, and dates; replace the personal-context sentences with a one-line structural description ("opens from a detail in their last call"); tag with play type and a coarse industry bucket. Before appending, check: could someone holding your pipeline list identify the recipient from this entry? If yes, blind further or skip. Before drafting from scratch, check the library for a pattern that fits. After a strong draft ships, ask Sam's scaling question in the pack: "who else in the funnel should get a version of this?"

## Weekly cadence

Designed to run weekly (suggested: Friday afternoon, so educational touches are ready to send Saturday morning per Sam's quiet-inbox rule). Do not arm any scheduled task for this skill without the user's explicit approval. Until scheduled, the user triggers it manually with `/keep-warm`.

## Hard boundaries

- Draft-only. Nothing sends, ever, from this skill. No exceptions.
- Behavioral read-only on the CRM (even if the key can write; this skill must not).
- Never invent facts about a contact, their company, or your results.
- Never re-share a link to the same contact.
- Respect opt-outs immediately and permanently.
- An empty roster or a zero-draft run is reported as such with reasons, never padded.
- State in `~/.claude/state/keep-warm/`, run artifacts under its `runs/`, the reusable library in your sales docs. Nothing mutable inside the skill folder.
