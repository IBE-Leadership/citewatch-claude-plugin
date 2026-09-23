---
name: citewatch-citation-audit
description: Conduct a thorough, correctly-scoped academic citation audit using the CiteWatch MCP tools -- verify references against real bibliographic data, check citation/reference balance, detect referencing-style inconsistencies, and handle credit limits and extraction uncertainty correctly. Use whenever the user asks to audit, check, verify, or fact-check the citations/references in a manuscript, thesis, dissertation, essay, or paper, or mentions CiteWatch by name.
license: MIT
compatibility: Requires a connected CiteWatch MCP server (any connector name -- this skill does not assume a specific tool-name prefix). See https://citewatch.app/setup to connect one.
metadata:
  author: CiteWatch
  version: "2.20"
---

# CiteWatch citation audit workflow

This skill governs how to run a citation audit well once a CiteWatch MCP
connector is available. It exists because getting this wrong looks
plausible but produces exactly the kind of confidently-wrong output an
academic citation audit must never produce: fabricated or incomplete
extraction, findings presented as confirmed when they were never
verified, and an audit that silently keeps going after running out of
credits by quietly substituting the model's own guesses.

## 0. Check you're running the latest version of this skill

This skill is installed as a static local file with no automatic update
mechanism, so your copy can silently go stale. Once, at the start of the
first citation-audit task in a session (not on every tool call), if you
have a web-fetch capability available: fetch
`https://citewatch.app/skill/citewatch-citation-audit/SKILL.md` and
compare its frontmatter `metadata.version` against this skill's own
`metadata.version` declared in the frontmatter above. If they differ, tell the
user plainly, before doing anything else: "A newer version of the
CiteWatch citation-audit skill is available (yours: X, latest: Y) --
update with: `curl -o .claude/skills/citewatch-citation-audit/SKILL.md
https://citewatch.app/skill/citewatch-citation-audit/SKILL.md`", then
continue the audit with the (possibly outdated) instructions you already
have loaded -- don't block the task on this. If no web-fetch capability
is available or the fetch fails, skip this check silently and proceed
normally; it's a courtesy notice, not a requirement.

## 1. Read the whole document before extracting anything

Extract in-text citations and reference-list entries only from text
you've actually read into context in full -- never from a partial read,
and never invent a placeholder entry for a section you haven't read yet.
If the document is long, read it completely across as many turns as it
takes before extraction, rather than extracting from what's been read so
far and treating that as final.

Re-read your own extraction against the source text once before calling
any tool. Common real extraction failures worth specifically checking
for: stray text pulled in from tracked-changes/comments, merged or
truncated year ranges, dropped entries, and citations split across a
page/line break that got joined incorrectly.

Never let categories of extracted data bleed into each other when
assembling a tool-call payload. In particular, the bibliography/
reference-list array you send to a tool must contain *only* entries that
appear verbatim in the document's own reference list -- never a source
you've identified as orphaned or missing from the bibliography, a
foundational/theory source you recognize from general knowledge, or a
leftover test entry from earlier in the session, even briefly, even just
to test a tool call. This has happened in practice: a foundational-theory
source correctly flagged as an orphaned in-text citation (missing from
the bibliography) got carried over into the bibliography array itself
when building a `check_referencing_style_consistency` payload, silently
contaminating the input. Before calling any tool that takes a references/
bibliography list, confirm every single entry has a literal, verbatim
location in the document's own reference-list section -- if you can't
point to where it appears, remove it before calling the tool, not after.
This class of error is easy to make and hard to catch afterward, because
these tools trust the input list completely and have no way to
distinguish a genuine entry from a contaminating one.

### Extract the citing claim text alongside each reference

Whenever a reference is used to support a specific finding, statistic, or
conclusion in the document's body text (not just a bare mention), extract
that sentence too and send it as `claim_text` on that reference's own
call. If supplied and an abstract is found, the server automatically
checks the claim against the abstract server-side -- including for
methodological over-generalization (see step 6's Contextual Misuse Flags
section) -- so this is genuinely worth doing across the manuscript, not
just for a few spot-checked claims, wherever the body text makes an
attributable claim to a reference.

**Also send `claim_section`, and don't hand-edit `claim_text`.** Name
which part of the document the citing sentence is in --
`"introduction"`, `"literature_review"`, `"methods"`, `"results"`,
`"discussion"`, `"conclusion"`, `"abstract"`, or `"other"` -- as
`claim_section` alongside `claim_text` on the same call. This is a cheap,
mechanical lookup (which heading is this sentence under), never a
judgment call about the sentence's wording, and it matters: in the
introduction/literature review, a citation typically attributes a finding
directly to the source ("Smith (2020) found X"), but in the discussion,
conclusion, and often results (where a study interleaves its own numbers
with literature comparisons), a citation is conventionally used to situate
the AUTHOR'S OWN result against prior work ("our finding of X is
consistent with Smith, 2020") -- the specific statistic there belongs to
the manuscript being audited, not the cited source. Submitting `claim_text`
verbatim from a discussion/conclusion sentence like that, with no
`claim_section`, used to get misread as an attribution and could flag it
`claim_not_supported`/`claim_contradicted` simply because Smith's abstract
doesn't happen to contain the author's own number -- a false fabrication
flag on a citation that never claimed the source reported that figure.
Do NOT try to fix this yourself by rewriting or trimming `claim_text` down
to "just the attributable part" -- that reintroduces the same fragile,
error-prone parsing this field exists to avoid. Send the sentence
unedited, set `claim_section` correctly, and the server applies the right
tolerance for that section automatically (lenient about a missing exact
figure in discussion/conclusion/results, strict attribution elsewhere).
Omit `claim_section` only when you genuinely can't tell which section a
claim falls in.

**When one sentence cites two or more references together** (e.g. "Prior
work has found X (Smith, 2020; Jones, 2019; Lee, 2021)"), include the
*same* `claim_text` on **each** of those references' own entries in the
tool call -- not just the first one. Each cited source gets checked
independently against the identical claim; a reference silently dropped
from this because it wasn't first in the list never gets checked at all.
If the sentence instead attributes *different, source-specific* findings
to each reference (e.g. "Smith (2020) found X while Jones (2019) found
Y"), extract each reference's own specific portion as its `claim_text`
rather than sending the whole combined sentence to all of them -- send
the most specific attributable text for each source, falling back to the
full shared sentence only when the text doesn't distinguish per-source
attribution.

### Locate the sentence within its own paragraph -- never just "the paragraph's longest sentence"

A paragraph often carries more than one citation, each supporting its own
distinct point. Picking "the longest/most substantive sentence in the
paragraph containing this reference's citation" and using that as
`claim_text` is not a safe shortcut -- that sentence can easily belong to a
*different* citation in the same paragraph rather than the one it's being
attached to. This has happened in practice: across one audit, 148 of 169
references (88%) were submitted with `claim_text` that was actually about a
neighboring citation, not the reference it was filed under -- every one of
those came back as an apparent "not supported by abstract" finding that was
really just wrong input, not a genuine flag.

Do it mechanically instead: within the citation's own paragraph, find the
specific in-text citation string for *this* reference (the literal
"(Smith, 2020)" or "Smith (2020)" occurrence), then extract only the
sentence bounded by the nearest periods that actually contains that
occurrence -- not the paragraph's first sentence, not its longest sentence,
not whichever sentence looks the most substantive. Before submitting,
confirm the reference's own surname (or, for a source without a clear
surname, the first significant word of its citation) literally appears in
the `claim_text` you're about to send. If it doesn't, the extraction picked
the wrong sentence -- go back and find the right one rather than submitting
it anyway. This check is cheap and mechanical, and it catches exactly the
failure mode above before it ever reaches CiteWatch.

### Parse and pass the reference's own cited metadata -- not just `reference_string`

`verify_reference`/`verify_manuscript_references` accept `cited_title`,
`cited_authors`, `cited_year`, and `cited_venue` as separate arguments,
parsed out of the reference-list entry itself. These are not just cosmetic
inputs for the metadata-mismatch report line (`detail.metadata_checks`) --
`cited_year` and `cited_title` feed directly into server-side matching:
title similarity is scored against `cited_title` when it's supplied,
instead of falling back to fuzzy-matching the whole raw `reference_string`,
and `cited_year` is checked against the resolved candidate's own year to
decide whether the match is confident. Omitting them doesn't just weaken
the report -- it can let the wrong source through completely undetected.
This has happened in practice: a reference to "Newman et al. (2018)" was
matched to a 1999 book review by the same author surname, because
`cited_year` was never passed and nothing forced a year check against the
edition actually found. Passing `cited_year: 2018` would have surfaced a
stark year mismatch automatically, catching the wrong-edition match before
it reached the report.

Parse and pass all four fields on every verify call whenever the
reference-list entry gives you the information to do so -- which is nearly
always true for a properly formatted entry. Only fall back to
`reference_string` alone for a field the entry is genuinely too malformed
to parse -- and when that happens, the malformation itself is worth
flagging per step 6's **[XX]** category, not silently worked around.

`cited_venue` specifically also feeds a server-side safety net against a
different, riskier failure than a year mismatch: several real references
(Babbie 2020, Donabedian 1988, Kline 2016, Field 2018) confidently
attached to a clearly DIFFERENT work that just happened to share a similar
or generic title -- a different edition, an unrelated paper the fuzzy
title scorer over-credited. The server now checks whether a candidate
disagrees with the cited authors AND year AND venue all at once before
accepting a title-only match, and routes it to "no confident match"
instead when all three disagree -- but that check only has data to work
with when you actually supplied `cited_venue` in the first place. Omitting
it doesn't just lose one metadata field from the report; it removes one of
the three signals this specific safety net depends on.

### On a large manuscript, this is a real scope decision -- make it out loud, not silently

Full `claim_text` coverage means a *second* complete pass over the body
text, independent of extracting the reference list itself: finding, for
every reference used to support a finding, the exact sentence that uses
it, and hand-matching it to that reference's own verification call. For a
short document this barely adds work. For a dissertation/thesis/book-
length manuscript with 50+ references, it is a materially bigger task
than verifying the references' existence and attribution -- easily
comparable to reading the whole document a second time. Existence/
accuracy verification (does this source exist, is it correctly
attributed, is it retracted) never requires this second pass; the
claim-support/usage check (does the source actually say what the
document claims it says) always does. These are two different-sized
jobs, not one job with an optional extra field.

### Always ask the user, before running any paid verification -- this is not your call to make

Ask this, verbatim in substance, before the first `verify_reference`/
`verify_manuscript_references` call of the audit -- not after, not only
when the manuscript happens to be large, and not as a decision you reason
through yourself when there's "a chance to ask":

1. Check claims against abstracts for accuracy (full coverage).
2. Check only a random sample of claims for accuracy.
3. Check only suspicious claims for accuracy (references already flagged
   elsewhere -- a metadata mismatch, low confidence, or a web-search-only
   match).
4. Check no claims against abstracts (existence/accuracy verification
   only).

**If the user doesn't answer, the default is always option 1 (full
coverage)** -- never option 3 or 4, and never something you pick for
them because the document is large. Full coverage is a genuinely bigger
task (see below), but "the user didn't respond" must never quietly become
"so I'll do the cheap thing" -- that's the exact failure this skill has
hit repeatedly: an unstated scope decision the user never actually agreed
to.

Whichever option applies, pass it to `generate_verification_certificate`'s
`claim_check_scope` argument at the end (`"full"`, `"random_sample"`,
`"suspicious_only"`, or `"none"`) -- this is a required, validated field
(defaults to `"full"` server-side too, matching the rule above), not
optional. The certificate also shows its own server-computed count of
matched, abstract-available references that never got a claim check, as
neutral context next to your note -- this is normal even under `"full"`
scope (bare-mention and methods citations never need a claim check), not
evidence something was missed.

Why this is a real decision worth asking about, not a formality: full
`claim_text` coverage means a *second* complete pass over the body text,
independent of extracting the reference list itself: finding, for every
reference used to support a finding, the exact sentence that uses it, and
hand-matching it to that reference's own verification call. For a short
document this barely adds work. For a dissertation/thesis/book-length
manuscript with 50+ references, it is a materially bigger task than
verifying the references' existence and attribution -- easily comparable
to reading the whole document a second time. Existence/accuracy
verification (does this source exist, is it correctly attributed, is it
retracted) never requires this second pass; the claim-support/usage check
(does the source actually say what the document claims it says) always
does. These are two different-sized jobs, not one job with an optional
extra field -- which is exactly why the user, not you, should be the one
deciding whether to pay for the bigger job.

If there's genuinely no chance to ask (an unattended/automated run with
no user to respond), the default from above still applies: proceed as if
option 1 was chosen, and **say so explicitly in the report** -- see step
6.5's Contextual Misuse Flags section, which requires this decision to be
stated outright rather than left to be inferred from an absent section.
A reader should never have to guess whether usage/claim-support checking
was considered and deliberately scoped down, or simply never occurred to
you.

## 2. Locate the reference list reliably, and track long documents as you go

### The mandatory procedure: eat the elephant one bite at a time

Never treat a manuscript as one undifferentiated pass -- extract
everything, then verify everything, then report. That's exactly how
entries get dropped, merged, or silently skipped on anything longer than
a few pages. Follow these steps, in this order, every time:

### Capture `audit_session_id` from your very first verify call, then reuse it for the whole audit

The first `verify_reference`/`verify_manuscript_references` call of this
audit won't have one yet -- omit it, and the server generates one,
returned as `audit_session_id` in that response (top-level for
`verify_manuscript_references`, alongside the other fields for
`verify_reference`). Record it (in the tracking file, if you have one)
and pass that exact same value as `audit_session_id` on every later call
for this same audit -- every batch, every retry, every resumed call --
right through to `generate_verification_certificate` at the end.

This exists because CiteWatch's certificate used to sweep up "everything
uncertified on the account" with no way to tell one audit's results apart
from another's -- confirmed live to actually mix an unrelated call into a
real certificate. Forgetting to propagate the id isn't silently dangerous
any more (each call would just start its own session, visible immediately
as a low `entries_included` count when you generate the certificate), but
getting it right the first time avoids that friction entirely.

**If you need to redo part of an already-certified audit -- resubmitting
corrected `claim_text`, re-checking a reference, anything -- start a fresh
`audit_session_id` rather than reusing the one from the certified session.**
Reusing a session id after `generate_verification_certificate` has already
been called on it has been observed to make CiteWatch silently return
`"unmatched"` for references that had verified perfectly fine originally --
not an error, nothing to signal the session is stale, just a wrong-looking
result. Omit `audit_session_id` on the first call of the redo pass so the
server issues a new one, track it separately from the original, and
generate a separate certificate for the corrected pass at the end (per step
7) rather than trying to fold corrected results back into the
already-certified session.

**If the user wants a genuinely independent re-verification of a
manuscript you've already audited -- a clean-room re-test, checking
whether a fix or a newer skill/prompt version changes the result, a
second opinion -- pass `force_refresh: true` on every call in that pass.**
A fresh `audit_session_id` (above) is necessary but **not sufficient** for
this: by default, if a reference_string from the earlier audit was left
**uncertified** (an audit that was interrupted, only partially certified,
or never finished `generate_verification_certificate`), CiteWatch silently
plays back that old, uncertified result instead of re-verifying anything
-- a fresh session id changes which session the NEW result gets filed
under, but does nothing to stop the old one from being reused in the first
place. Confirmed in practice: a 169-reference re-audit, run specifically
to independently confirm a prior audit's findings, only freshly
re-verified 29 of the 169 -- the other 140 silently replayed uncertified
results left over from the earlier run, undercutting the entire point of
re-running it. `force_refresh: true` discards any existing uncertified
result for each reference first and forces a genuinely fresh pipeline run
-- a full credit charge per reference, same as any other fresh
verification, not free. Only set it for a deliberate re-verification pass;
leave it false (the default) for an ordinary resume/retry within the SAME
audit, where reusing an uncertified result is exactly the intended,
free behavior.

**Step A -- Size up the document before touching it.** Decide which of
three cases you're in, and commit to the matching unit of work before
extracting anything:

1. **Large, with chapters** (a thesis, dissertation, or book-length
   manuscript divided into numbered/titled chapters) -- the unit of work
   is **one chapter at a time**.
2. **Large, with sections but no chapters** (a long report or article
   divided into major headed sections but not full chapters) -- the unit
   of work is **one section at a time**.
3. **Small, with no meaningful internal division** (a short paper, essay,
   or report where reading it in one pass is genuinely manageable) -- the
   unit of work is **the entire document**.

Say out loud which case applies and why before extraction begins -- this
decision drives everything that follows, so make it deliberately.

**Step B -- Plan it before extracting anything.** Build an explicit,
ordered todo list of the work: one item per unit from Step A (one per
chapter, one per section, or a single item for an undivided small
document), in document order, plus trailing items for this audit's
remaining stages (free structural checks, certificate generation, final
report). If a native todo/task-tracking tool is available in this
environment, use it directly; otherwise write the plan as the opening
section of the tracking file described below. Mark a unit's item
complete only once every citation inside it has gone all the way through
Step C below -- never mark one done partway through, and never jump
ahead to a later unit out of order. This is what turns a 100+ reference,
multi-chapter audit into a visible, resumable checklist instead of an
implicit mental model that's easy to lose track of -- especially
important since a long audit can span more turns than comfortably fit in
one context window, or cross a session boundary.

Alongside the unit todo list, if the same reference is likely to recur
across multiple chapters/sections (common in theses, edited volumes,
compiled reports), also start a reference-tracking table now: one row
per unique reference, with its reference string, a status field starting
at "not yet checked", and a citing-locations list. Populate and update
this table incrementally as you complete each unit in Step C below --
**not** via a separate whole-document read done first. The first time you
reach a given reference, verify it and record the result in this table.
A later unit that cites the same reference again should normally just
reuse that recorded result instead of calling the tool again -- never
re-verify a reference just because it recurs later in the document, that
spends credits on a call whose answer you already have.

**Exception: a later citation with its own new claim.** If THIS later
citation attributes a specific finding, statistic, or conclusion
(`claim_text`) that the first call never had -- the common case where a
reference gets a bare mention early on and a substantive claim only later
-- call `verify_reference`/`verify_manuscript_references` again with the
same reference string and this new `claim_text`. The server recognizes
it's the same reference, does **not** re-charge the base per-reference
credit, and runs the claim-vs-abstract check fresh against the abstract
already on file (a small extra charge only for that check, same as a
first-time check) -- update this table's row with the new `claim_support`
result. Only skip the call when the recurring citation has no claim_text
of its own, or repeats a claim already checked on an earlier call for the
same reference.

Keep orphaned/missing-from-bibliography candidates in their own clearly
separate rows (or a separate file) from genuine bibliography rows in this
same table -- both get sent to CiteWatch per Step C.5 below, but they
must never be blended together when you build a bibliography tool-call
payload, or in the report's own figures later.

If you have file-write access in this environment, save and update both
the todo list and the tracking table on disk as you go, rather than
holding them only in the conversation.

**Prefer persisting each tool call's response near-verbatim over a
hand-condensed summary.** It's tempting, especially in a long audit, to
compress each result down to a short human-readable line when writing it
into the tracking table (verdict + a phrase, not the full JSON) -- readable
in the moment, but it destroys traceability: confirmed in practice, one
audit's condensed tracking files preserved full per-claim detail for only
72 of 114 checked claims, so roughly a third of the claim verdicts in the
final report couldn't be traced back to the raw tool output at all without
re-spending credits to re-run verification. Where you have the room for
it, persist the response body (or at minimum every field the report
actually draws on: `flags`, `detail.matched_metadata`,
`detail.metadata_checks`, `detail.claims`, `detail.escalation`,
`detail.retraction`, `detail.journal_quality`) rather than a condensed
paraphrase. If space or environment constraints genuinely force
condensing for some or all entries, that's an acceptable tradeoff -- but
disclose it as an explicit limitation in the final report (how many
entries have only a condensed record, not full per-claim/per-field
detail) rather than leaving a reader to discover it only if they try to
audit a specific entry and find nothing to check it against.

**Step C -- Work through one unit at a time, in document order.** For
cases 1 and 2, process chapters/sections strictly in order, one at a
time -- never skip ahead "to get a feel for the whole document" first,
and never merge several chapters/sections into a single extraction pass.
Within each unit, before marking its todo item complete and moving to the
next:

1. Read the unit fully (per step 1: never extract from a partial read).
2. Extract every in-text citation in that unit, together with its
   `claim_text` wherever the citation supports a specific finding,
   statistic, or conclusion (see step 1's claim-text guidance above).
3. For each citation extracted this way, look it up against the
   document's reference list (see "Finding the reference list" below) to
   find its full bibliographic entry.
4. **If a matching reference-list entry is found**, pair the full
   reference string with that citation's `claim_text` (if any) and add it
   to this unit's `verify_manuscript_references` batch -- or, if the
   tracking table already has a recorded result from an earlier unit,
   either reuse that result unchanged (nothing new to check) or submit a
   follow-up call with just this citation's new `claim_text` per Step B's
   exception above (safe, no duplicate base charge, runs the claim check
   fresh).
5. **If no matching entry is found (an orphaned in-text citation)**, send
   it to CiteWatch anyway, using whatever text the in-text citation
   itself gives you (e.g. `"Smith, 2020"`) as the `reference_string`,
   **and set `is_orphaned_citation: true`** on that entry -- do not just
   note it locally and skip the server entirely, and never omit this flag.
   CiteWatch's own database counts and tracks these attempts regardless of
   whether a full reference existed to send; leaving them out means
   CiteWatch never learns about them, and the audit's own record of what
   was attempted is incomplete. The flag matters specifically because the
   server uses it to keep this entry out of the certificate's bibliography
   table and statistics entirely (there is no genuine reference-list
   attribution to verify completeness/accuracy against, so it would
   misrepresent both the table and the numbers to include it there) --
   omitting the flag makes an orphaned citation look like a real
   bibliography entry on the certificate, exactly the confusion this flag
   exists to prevent. This still costs 1 credit per attempt like any other
   reference -- factor the manuscript's likely orphaned-citation count
   into the budget check in step 4, not just the bibliography's own entry
   count. Keep its result in the report's Orphan Citations section (step
   6.3), never folded into the Full Verification Table or Executive
   Summary counts that describe the bibliography itself -- it's a
   different category of entry (there is no genuine reference-list
   attribution to verify against), and blending the two would misstate
   what fraction of the actual bibliography was checked.
6. Only after every citation in the unit has gone through steps 3-5,
   update the tracking table, mark the unit's todo item complete, and
   move on to the next one.

Case 3 (small, undivided documents) follows the same six sub-steps --
there's simply one unit, the whole document, instead of many.

By the last unit, the tracking table already holds every reference's full
citing-location history and verification result -- build the Full
Verification Table and Orphan Citations report sections directly from it
rather than re-deriving them from memory.

**Reconcile the found/orphaned judgment before generating the certificate.**
Step 5 above has you decide found-vs-orphaned per citation in real time,
one at a time, as you walk through each unit -- a necessary shortcut so
you can batch-verify incrementally instead of waiting for the entire
document to be read first, but it's still just your own ad hoc comparison,
made without the complete in-text-citation and reference-list side by
side. `check_citation_reference_balance` (step 3) is the tool built to do
this same comparison more carefully, once every unit's citations and the
full reference list have actually been extracted -- nothing above ties
the two together automatically, so do it explicitly: once you've reached
the end of the document (the last unit on your todo list), call
`check_citation_reference_balance` with the complete accumulated
`in_text_citations` and `reference_entries` lists, before generating the
certificate, and reconcile its `orphaned_citations` output against what
you actually submitted with `is_orphaned_citation: true` during the walk:

- If it lists additional orphaned citations you never submitted at all
  (missed during the walk -- easy to happen in an early chapter, or if a
  reference-list lookup was rushed), submit those now with
  `is_orphaned_citation: true` before generating the certificate. The
  certificate must reflect every orphaned citation the free check
  independently confirms, not just the ones caught in real time.
- If something you submitted as orphaned is absent from its
  `orphaned_citations` list (i.e. it found a plausible reference-list
  match your real-time lookup missed), don't silently trust either
  judgment -- note the discrepancy in the report's Orphan Citations
  section as a possible extraction inconsistency worth the user's own
  check, per that tool's own `extraction_disclaimer`.

**On batch sizing** (independent of the unit-by-unit plan above): a
single chapter or section can have anywhere from a handful to 100+
references, so batch the actual `verify_manuscript_references` **calls**
by size, not by unit boundary -- send **~15-20 references per call**
(smaller, ~10, if you're also sending `claim_text` for most entries in the
batch -- the automatic claim-vs-abstract check adds a real extra step per
reference, so a batch that size takes meaningfully longer than the same
size did before that feature existed), not a single huge call for an
entire chapter or the whole bibliography (a very large batch can run
several minutes, especially if several references need the automatic
web-search/scrape escalation or claim check, and a long-running single
call has more that can interrupt it along the way). These are larger
numbers than earlier guidance gave, on purpose -- see "Responses are
compact by default" below: only references that actually have something
wrong return full detail, so a typical batch's response size no longer
scales anywhere near as fast with batch size as it used to; the limiting
factor is now wall-clock time and how many entries in a given batch turn
out flagged, not response size on its own. If a batch call errors out or
the connection drops partway through, just call it again with the same
batch (or continue with the remaining references) -- the server detects
anything already verified and not yet certified and returns it for free
(`resumed_from_earlier_call: true`) instead of redoing the work and
charging twice, so retrying is always safe.

### Responses are compact by default -- read `flags`, not raw fields

`verify_reference`/`verify_manuscript_references` used to return every
field for every reference -- full abstract text, a complete per-field
metadata comparison, the works -- regardless of whether anything was
actually wrong with that reference. Confirmed live: this is exactly what
forced a real audit's calling model to artificially ration how many
references it verified in one session, not a credit limit -- a clean
reference's full abstract text alone was measured at ~4,300 of ~6,300
response characters (68%) for one real reference, with **no use at all**
under this skill's own instructions when `claim_text` wasn't submitted
for it. That's now fixed at the source: **a reference with nothing wrong
returns only what's needed to write one clean report line.**

Each entry now comes back with: `matched`, `match_confidence`,
`match_method`, `doi`, `title`, `journal_matched` (bool), `quartile`,
`abstract_available` (bool, not the abstract text itself), `flags` (a
list of short strings), a compact `claim_support`
(`checked`/`verdict`/`methodology_flag`/`skipped_reason` -- no
`rationale`/`confidence`/`methodology_note` at this level), and the usual
`credits_charged`/`credit_balance(_after)`.

**`flags` is the signal to read, not any of the raw fields it used to
take their place.** An empty list means exactly that: nothing to report
beyond matched/confidence, full stop -- don't go looking for a
`metadata_checks` or `abstract.text` field that isn't there anymore.
A non-empty list -- e.g. `["retracted"]`, `["metadata_mismatch:pages"]`,
`["low_confidence"]`, `["unmatched"]`, `["web_search_only"]`,
`["claim_contradicted"]`, `["claim_methodology_flag"]`,
`["claim_unverifiable"]`, `["claim_check_parse_error"]`,
`["orphaned_citation"]`, or a
`journal_quality_concern` flag for a blacklisted/flagged journal -- means
the entry ALSO includes a `detail` object with everything needed to write
that report entry: full `matched_metadata`, `metadata_checks` (per-field
cited/found), `abstract`, `escalation` (including
`verification_note`/`grey_literature`), `claims` (see below), `retraction`,
and `journal_quality`. No second call needed for anything actually worth
reporting -- this is exactly the case the old verbose-by-default shape
was solving for, just without paying the cost for every clean reference
too.

**One citation, many claims.** The same reference can be cited more than
once with a different attributable claim each time, so `claim_support` in
every response (compact or detail) is a *count*, not a single verdict:
`{"claims_checked": N, "claims_flagged": N, "claims_unverifiable": N,
"claims_parse_failed": N}` -- "how many claims were checked against this
reference, and how many of those held up." The full per-claim breakdown --
each claim's own text, verdict, confidence, rationale, methodology
flag/note -- is in `detail.claims` (a list, one entry per claim actually
submitted) whenever `flags` is non-empty, or via `get_reference_detail`
otherwise. Don't look for a single `claim_support.verdict`/
`claim_support.checked` field anymore -- that shape is gone; iterate
`detail.claims` instead.

**`claims_parse_failed` is a fourth, distinct state from
`claims_unverifiable` -- never fold them together.** Confirmed live: ~13%
of a real audit's claim checks came back as a fabricated-looking
`CANNOT_ASSESS` verdict with the rationale "unparseable model response" --
indistinguishable from a genuine inconclusive judgment unless you happened
to read that exact rationale text. The server now detects this itself
(retries once internally before giving up) and reports it as `checked:
false`, `skipped_reason: "claim_check_parse_error"` in `detail.claims`,
with its own `claim_check_parse_error` flag and its own `claims_parse_failed`
count -- **this means the tool broke on that claim, not that there was
genuinely nothing to check it against** (that's `claims_unverifiable`/
`no_abstract_available`, an entirely different, permanent condition -- see
step 6.5's Contextual Misuse Flags section for how to report the two
differently). Never write a `claims_parse_failed` entry into the report as
if it were a checked-and-inconclusive claim.

For a reference that came back clean (`flags: []`) but you still want the
full detail for -- to read its abstract yourself independent of an
automatic claim check, or to double-check something not surfaced by
default -- call `get_reference_detail` with that entry's exact
`reference_string`. It's free (reads back what's already stored, doesn't
re-verify or spend credits) and returns the complete record. Most
references never need this; reach for it only when you specifically want
more than a clean summary line requires.

Batches don't have to be sent one at a time, either -- if your
environment lets you make multiple tool calls concurrently, you can fire
several `verify_manuscript_references` batches in parallel to cut the
audit's total wall-clock time, rather than waiting for each batch to
finish before starting the next. The server handles this safely: if two
concurrent calls ever verify the exact same reference at once, a
database-level check on its side means only one is actually recorded and
charged, and the other automatically gets refunded and reflects that same
result instead of creating a conflicting duplicate.

### Finding the reference list

Don't search for a single heading text and stop. Reference lists appear
under many different headings depending on discipline and referencing
style -- check for all of: "References", "Bibliography", "Works Cited",
"Reference List", "Literature Cited", "Sources", "Sources Cited", and
"Notes" (some humanities/Chicago-style documents fold citations into a
notes section rather than a separate list) -- case-insensitively, and
without assuming singular vs. plural. If a search for one heading comes
back empty, that is evidence the heading is different, not evidence the
document has no reference list -- broaden the search before concluding
otherwise.

Also check explicitly whether the document has **one master list at the
end, or a separate reference list per chapter/section** (common in
theses, edited volumes, and compiled reports). Skimming only the very
end of the document will miss per-chapter lists entirely. Confirm which
structure you're dealing with before extraction, not after.

## 3. Run the free checks first, and treat their output as leads, not verdicts

### Before trusting either free check's output, confirm the extraction it's built from is actually complete

`check_citation_reference_balance` and `check_referencing_style_consistency`
are pure local matching over the citation lists *you* extracted -- they
have no access to the source document at all, so they cannot tell a
genuinely missing citation apart from one that simply never made it into
your own extraction list. Confirmed in practice: a real audit's in-text-
citation list silently lost several real citations (a mix of author-name
and organization-name citations) across a context-compaction event
mid-session -- the extraction file simply never picked them back up after
the session resumed -- and the free balance check, built from that
incomplete list, then flagged bibliography entries as "unused" partly
because the citations that would have matched them were never in its
input to begin with. Neither free check has any way to detect this; both
trust the list you hand them completely, the same way a verify call trusts
its `reference_string` input (see step 1's warning about contaminated
payloads).

Before calling either free check, and again immediately after resuming
from a context-compaction event or a new session picking up an
in-progress audit, spot-check the extraction list against the actual
source text: pick several paragraphs at random -- and specifically
whatever was read right before the point compaction happened or the prior
session ended, since that's exactly the boundary where an entry silently
drops -- and confirm every citation in them made it into the list. This is
a cheap, mechanical check. It matters precisely because the failure mode
is silent: the list looks complete (nothing errors, nothing looks
obviously short), while quietly missing entries from whatever wasn't
re-confirmed after resuming.

### Read the results as leads, not verdicts

`check_citation_reference_balance` and `check_referencing_style_consistency`
are free (no credits) but are pure local matching over the citation lists
*you* extracted -- they have no access to the source document and cannot
tell a genuine orphaned citation or style inconsistency from one caused
by your own extraction error. Both responses include an
`extraction_disclaimer` field explaining this; read it and reflect its
substance in how you present results. Never state an `orphaned_citations`,
`unused_references`, or `flagged_entries` item as a confirmed finding --
frame each one as a candidate the user should verify against their own
document.

### Every orphan/unused candidate must be reconciled before it can appear in the report

Reconciliation is not optional and not a nice-to-have -- an orphan/unused
candidate may never go into the report as raw tool output. Confirmed in
practice: of 14 candidates one real audit's free check flagged as
"orphaned in-text citations," 13 turned out to have an exact match
already sitting in the bibliography once checked directly -- the free
check's own surname+year matching simply failed on ordinary formatting
differences. The 14th failed to match purely because of an
apostrophe-encoding difference between the in-text citation and the
bibliography entry (a curly vs. straight apostrophe in a name like
Dall'Ora). None of the 14 were genuine gaps. The honest characterization
of a result like that is "essentially all false positives," not "most are
likely artifacts" -- get the reconciliation done so you can say the
stronger, more accurate thing instead of hedging around a result you never
actually checked.

Before any `orphaned_citations` or `unused_references` candidate appears
in the report:

- For a candidate **orphaned in-text citation**, do an independent
  surname+year search directly against the actual bibliography text (not
  the extracted list you built -- the source document's own reference list
  itself), checking for the kind of formatting differences fuzzy matching
  misses: a different citation style, an initial vs. full first name, a
  reordered author list.
- For a candidate **unused reference**, do a direct search of the source
  text for that reference's surname, specifically checking accent and
  apostrophe-encoding variants (curly `'` vs. straight `'`, accented vs.
  unaccented letters, e.g. `é`/`e`) -- these are exactly the class of
  difference the free check's fuzzy matching misses and a plain
  case-sensitive search would too.
- Only a candidate that still doesn't resolve after this direct check may
  be reported as a genuine orphaned-citation or unused-reference finding.

State the reconciliation itself in the report, not just the survivors:
"the free check flagged N raw candidates; M were resolved as false
positives via direct surname/accent-variant matching against the source
text, leaving K genuine [orphaned in-text citations / unused references]."
Reporting the raw tool count alone, as if it were the finding, overstates
the problem by exactly the amount this reconciliation step catches -- see
step 6.3's Orphan Citations section and the Executive Summary table, both
of which require this same raw-vs-reconciled breakdown, not a bare count.

`check_journal_quality` and `check_retraction_status` are also free and
can be run per matched source without affecting the credit budget.

## 4. Before spending any paid credits, check the budget and ask if it's tight

`verify_reference` and `verify_manuscript_references` cost 1 credit each
(per reference), plus a small fractional add-on whenever the automatic
web-search/scrape escalation or claim-vs-abstract check actually fires
for that reference (a few thousandths of a credit each -- see `Notes`
guidance below; never a whole extra credit). `search_literature` costs 5
credits per call -- meaningfully more than a single reference
verification, so factor that in when estimating whether a balance covers
a planned sequence of calls, not just a flat per-tool-call count. Call
`get_credit_balance` (free) before committing to a full-manuscript
verification pass. Estimate the reference count from the *whole*
document, not just the bibliography: per step 2's mandatory procedure,
every orphaned in-text citation also gets sent to CiteWatch and costs 1
credit, same as a genuine bibliography entry -- a manuscript with a messy
or incomplete reference list can easily have meaningfully more orphaned
citations than usual, so don't estimate off the bibliography's entry
count alone. If the reference count exceeds the available balance,
stop and ask the user how to proceed rather than silently deciding for
them -- for example:

1. Verify only the highest-risk subset (no-DOI entries, entries the free
   checks flagged, or claims central to the argument) within budget.
2. Verify what's affordable now, and note exactly what's left for later.
3. Wait for the user to add credits, then verify everything.
4. Skip paid verification and rely on the free checks alone.

Presenting this choice *before* spending credits is good practice -- keep
doing it. What must never happen is spending down to zero and then
continuing anyway on the model's own knowledge without the user having
agreed to that scope.

## 5. If you hit `insufficient_credits`, stop -- this is not optional

Every metered tool returns a response containing `"error":
"insufficient_credits"` and `"action_required": "STOP"` when the account
is out of credits (the batch tool additionally sets
`stopped_early_insufficient_credits: true` and reports exactly how many
references were processed vs. requested). When this happens:

- Do **not** verify, search, or assess the remaining items using your own
  knowledge, a web search, or any other tool as a substitute. That
  produces unverified guesses presented as if they were checked against
  real bibliographic data -- the exact failure mode credit metering
  exists to prevent, and it has been observed happening live.
- Tell the user immediately and clearly: exactly how many items were
  verified vs. not checked at all, and the purchase link
  (`https://citewatch.app/billing`) from the tool response.
- Do not fold any unverified item into the audit's findings. If you've
  already produced partial results, present them as partial and stop
  there -- don't pad the rest of the report with guesses to make it look
  complete.

## 6. Standard report format

When the user asks for a written audit report (as opposed to a quick
chat answer), use this structure every time, so output is consistent
across sessions and users. Keep verified, flagged, and unchecked
findings visually and textually distinct throughout -- never blur them
into one undifferentiated list.

### Header

```
# Citation Audit Report — <Document Title> (<status, e.g. Draft/Revised>, <date>)

**Document:** "<title, as given by the user>"
**Author/Candidate:** <name, if given>
**Auditor:** CiteWatch (https://citewatch.app), via Claude
**Date:** <today's date>
**Methodology:** Systematic verification against OpenAlex, Crossref, PubMed,
and Unpaywall (open bibliographic data). Confidence levels assigned per
entry. Entries marked "Unverifiable" are NOT assumed fabricated — they
require manual follow-up, not automatic suspicion.
```

### 1. Executive Summary

A metrics table, computed from your actual tool results (not estimated).
Every row is a count **and its percentage of total bibliography entries
audited** (e.g. `12 (11%)`), not a bare count -- this matters specifically
for the non-academic/other-source rows below, since a report reader
needs to immediately see what share of the whole bibliography those
categories represent, not just an absolute number:

| Metric | Count (% of total) |
|---|---|
| Bibliography entries audited | total reference_entries |
| Verified, high/moderate confidence | `matched: true`, `match_confidence` high or moderate, `flags` has no `web_search_only` |
| Verified, low confidence | `matched: true`, `"low_confidence" in flags` |
| Verified via web search only (not index-corroborated) | `"web_search_only" in flags` -- see below |
| Grey literature / non-academic source | `"grey_literature" in flags` -- see **[GL]** below |
| DOI corrected (cited DOI didn't resolve; automated repair found and verified the right one) | `"doi_corrected" in flags` -- see **[DC]** below |
| Unverifiable (no match found) | `"unmatched" in flags` and no `"grey_literature"` |
| Retracted | `"retracted" in flags` |
| Metadata/completeness mismatches (title/authors/year/venue/volume/issue/pages) | any flag starting with `metadata_mismatch:` -- percentaged against `matched` count, not total, since an unmatched entry has nothing to compare against |
| Duplicate reference entries | from `generate_verification_certificate`'s `duplicate_reference_groups` (only available after that tool has been called -- see its own section below) |
| In-text citations missing from bibliography | genuine count **after** reconciliation (step 3's mandatory reconciliation, not the raw `orphaned_citations` count) -- state both, e.g. "2 (raw: 14)" |
| Reference entries never cited | genuine count **after** reconciliation (step 3's mandatory reconciliation, not the raw `unused_references` count) -- state both, same as above |
| Not checked (credit limit / scope decision) | explicit count, never omitted |

The "Verified via web search only", "Grey literature / non-academic
source", and "DOI corrected" rows exist so a reader can see at a glance how
much of the bibliography rests on a weaker or different kind of
verification than a direct bibliographic-index match -- never fold these
into the plain "Verified"/"Unverifiable" counts above them, and never omit
them even when their count is zero (write `0 (0%)` explicitly, same
discipline as "Not checked"). Unlike the other two, a nonzero "DOI
corrected" count is a genuine, positive finding about the CORRECTED source
(it verified cleanly once the right DOI was used) rather than a weaker kind
of verification -- but it still belongs in its own row rather than folded
into plain "Verified," because it means the manuscript's own bibliography
has a DOI that needs fixing, which is worth surfacing on its own even
though the underlying source itself checked out fine.

If any `claim_text` was submitted anywhere in the audit, add two more
rows immediately after the table above, percentaged against the number of
claims actually **checked** (sum every reference's `claim_support.claims_checked`),
not against total bibliography entries -- most entries typically won't
have had a claim submitted at all, so "% of total" would understate
things. A reference can contribute more than one claim to these totals
(one citation, many claims):

| Metric | Count (% of claims checked) |
|---|---|
| Cited claims checked against abstract | sum of every reference's `claim_support.claims_checked` (this row alone is % of total bibliography entries, not of itself) |
| Flagged: unsupported, contradicted, or methodology mismatch | sum of every reference's `claim_support.claims_flagged` -- see `detail.claims` for which specific claim(s) on a given reference triggered it |
| Not assessed: no abstract/summary existed to check against (permanent) | sum of every reference's `claim_support.claims_unverifiable` |
| Not assessed: tool error, response unparseable even after retry | sum of every reference's `claim_support.claims_parse_failed` -- see the note below; never merge this row with the one above |

The last two rows are both "not assessed," but for entirely different
reasons a reader needs to be able to tell apart: "no abstract/summary
existed" is a **permanent, structural** limit (an older monograph or
textbook genuinely has no indexed abstract anywhere -- re-running the
audit will never resolve it), while "tool error" is a **transient**
failure of this specific run (the claim-check call itself broke) that a
later re-run could plausibly fix. Never report a `claims_parse_failed`
entry the same way you'd report a `claims_unverifiable` one -- if the
parse-failed count is nonzero, say so explicitly and suggest a
`force_refresh` re-run for just that handful of references rather than
treating the result as final.

Omit all four rows entirely if no claims were checked at all (rather than
writing `0 (0%)`) -- unlike the escalation rows above, which reflect
something the server does automatically for every reference, this only
ever reflects a scope decision you made about which claims to submit, so
its absence needs no explicit zero to be self-explanatory. Once any claim
has been checked, though, write out all four rows explicitly, `0 (0%)`
included for whichever of the last two are zero -- a reader needs to see
that the parse-failure count was checked and came back zero, not just
never mentioned.

Follow with a short **Critical Issues** list (numbered, most severe
first) -- confirmed retractions, foundational/heavily-cited sources that
are orphaned, and any entry where the reference-list text itself is too
malformed to identify (no title, no year, etc.) belong here first.

**Reserve "CRITICAL" for a genuine topical mismatch central to the
manuscript's own argument -- never apply it automatically just because a
claim came back `NOT_SUPPORTED`/`CONTRADICTED`.** Confirmed in practice: a
flagged claim sitting in an ordinary discussion-section paragraph -- the
student contextualizing their own finding against prior literature ("our
finding is consistent with Smith, 2020"), exactly the pattern step 1's
`claim_section` guidance describes -- was labeled CRITICAL in one real
report, when the actual sentence, checked directly against the manuscript,
turned out to be conventional literature-contextualizing, not a direct
misattribution of a specific result to the cited source. That's a real
weakness worth flagging (see step 6.5's Contextual Misuse Flags section),
but it is not the same severity as a fabricated author list, a retracted
source presented as sound, or a citation whose claim is centrally load-
bearing for the paper's own argument and demonstrably contradicted by the
source. Before labeling anything CRITICAL here, check its `claim_section`
and re-read the actual sentence in context: a `discussion`/`conclusion`/
`results`-section claim doing ordinary literature-contextualizing gets
calibrated language ("worth reviewing," "a weaker citation than ideal")
instead of an automatic CRITICAL label, even if the automatic claim check
flagged it. Reserve CRITICAL for cases where the topical mismatch is
central to what the manuscript is actually arguing, not merely present
somewhere in a flagged claim.

### 2. Full Verification Table

Legend (read from `flags` -- see "Responses are compact by default" in
step 2 for the full shape; each symbol below checks `flags` first, so
check them in this order -- retracted, grey-literature, and DOI-corrected
take priority over a plain "something's flagged"):
- **[XX]** `"retracted" in flags`, or the reference-list entry itself is
  too malformed to identify (not a matching judgment call -- an
  observable fact about the entry as written).
- **[GL]** `"grey_literature" in flags` -- not an academic paper (a
  government report, industry white paper, standard, etc. with no index
  entry to match against), but the server captured its full text and
  wrote a summary directly from the cited source
  (`detail.escalation.grey_literature.summary`). Don't mark it
  "Unverifiable" -- it's a different, more informative status than a
  plain no-match.
- **[DC]** `"doi_corrected" in flags` -- the DOI as printed in the
  reference list didn't resolve to anything, but the server ran a bounded,
  free repair search (small mechanical edits to the DOI's trailing digits
  -- a dropped digit, an extra digit, a single misread digit) and found a
  corrected DOI whose title/metadata closely match this reference. Read
  `detail.escalation.doi_correction` for the exact `cited_doi` (as printed)
  and `corrected_doi` (what the server verified against) -- state both in
  the report line. This is matched, not unmatched: the abstract,
  retraction, and journal-quality checks below all ran against the
  corrected record, so claim-support analysis works normally against it
  too. Report this as a likely transcription error in the manuscript's own
  bibliography, not as evidence of a fabricated or nonexistent source --
  the source itself was found and verified; only the DOI as printed was
  wrong. If the server's repair search couldn't find or validate a
  correction (most DOI typos more complex than a single trailing-digit
  slip), the reference falls through to **[??]**/**[GL]** below as usual --
  **[DC]** only appears when a correction was actually found and verified,
  never as a "we suspect a typo" guess of your own; don't try to manually
  guess or propose a corrected DOI yourself when this flag is absent, since
  an unverified guess is exactly the kind of unchecked claim this audit
  exists to avoid.
- **[??]** `"unmatched" in flags` -- genuinely unverifiable against open
  bibliographic data. This is **not** an accusation of fabrication -- say
  so explicitly, matching the header's methodology note. **Never blend your
  own general knowledge into this row's report line as if CiteWatch had
  verified it.** Confirmed in practice: a report added commentary like
  "well-known real publication -- likely a tool limitation" onto several
  [??] entries, written in a way that read as though the source's existence
  had been confirmed, when in fact that was the model's own prior
  knowledge, never checked by CiteWatch at all. If you recognize a source
  and have your own opinion about why it didn't match (a preprint not yet
  indexed, a very new publication, a non-English venue), you may say so --
  but it must be visibly and explicitly your own judgment, not CiteWatch's
  finding, e.g. "CiteWatch could not verify this against open bibliographic
  data (my own assessment, not verified by the tool: this may be a recent
  or non-English-language publication not yet indexed)." Never phrase it in
  a way a reader could mistake for something the tool itself confirmed.
- **[!!]** `matched: true` but `flags` is non-empty and none of the above
  apply -- any of `metadata_mismatch:<field>`, `low_confidence`,
  `web_search_only`, `journal_quality_concern`, `claim_not_supported`,
  `claim_contradicted`, `claim_methodology_flag`, `claim_unverifiable`. A
  `metadata_mismatch:` flag can mean the wrong paper was matched, but just
  as often means the reference-list entry itself is incomplete or has a
  typo (e.g. a missing volume number) -- present it as "check this
  entry's completeness," not as an accusation that the wrong source was
  found.
- **[OK]** `matched: true`, `flags: []`.

**Automatic web-search/scrape escalation.** The server automatically falls
back to a paid web search/scrape when the free sources (Crossref/OpenAlex/
PubMed/Unpaywall) can't resolve something on their own -- a missing
abstract with a known link, grey literature, or no match at all. This is
fully automatic; there is nothing you need to do differently to trigger
it. It surfaces in the response as:
- `"web_search_only" in flags` -- matched, but only via an independent web
  search, never corroborated against a structured bibliographic index.
  Always treat as **[!!]**, never as a plain **[OK]**, regardless of
  `match_confidence`. Since this flag is always non-empty, `detail` is
  always present for these.
- `detail.escalation.verification_note` -- a short explanation to carry
  into your `Notes` column whenever present (e.g. why a web-search match
  should be treated with extra caution, or that a second opinion
  corroborated an originally medium-confidence match).
- `detail.escalation.grey_literature` -- see **[GL]** above.
- `credits_charged` may include a small fractional amount on top of the
  flat per-reference cost (e.g. `1.0072` instead of `1`) when one of these
  steps fired, or when the claim-vs-abstract check ran (see step 6.5).
  Just report the actual number returned -- nothing about the
  insufficient-credits handling above changes.

Table columns: `#`, `Entry` (as cited), `Status`, `Confidence` (from
`match_confidence`, or "N/A" if unmatched), `Notes` (the specific
mismatched field with its `detail.metadata_checks[field].cited`/`.found`
values, retraction reason, or journal-quality concern -- concrete, not
vague; pull these from `detail`, present whenever `flags` is non-empty).
For a document processed via the tracking file in step 2, add a
`Cited from` column listing every chapter/page location on record for
that reference, instead of just the first one encountered.

### 3. Orphan Citations

From `check_citation_reference_balance`'s `orphaned_citations` and
`unused_references`, **after** step 3's mandatory reconciliation --
never the raw tool output. Open this section with the reconciliation
itself, not just the survivors: "The free structural check flagged N raw
candidates; M were resolved as false positives (formatting/accent/
apostrophe-encoding differences the check's own matching missed) via a
direct search against the source document, leaving K genuine
[orphaned citations / unused references]." List only the K genuine
entries below that line -- a candidate that reconciled away does not
belong in this section at all, not even with a note that it was resolved
(the reconciliation summary line already covers that). Carry forward the
tool's `extraction_disclaimer` in substance for whatever remains: these
are still candidates for the user to give a final check against their own
document, not confirmed gaps, even after reconciliation. Flag
foundational/theory-defining sources specifically if orphaned -- an
examiner or reviewer notices those fastest.

Per step 2's mandatory procedure, each orphaned citation was also sent to
CiteWatch on its own (using the bare in-text citation as `reference_string`
and `is_orphaned_citation: true`, since no bibliography entry existed to
send instead) -- report that result here too, next to the citation itself
(e.g. "CiteWatch attempted an independent bibliographic search on this
citation alone and found no match" or, on the rare occasion the bare
citation is enough to resolve, whatever it did find). Never fold these
results into the Full Verification Table or Executive Summary counts
above -- they describe a different thing (no genuine reference-list
attribution exists to verify against) and belong only in this section.
The certificate (step 7) already does this separation automatically --
`is_orphaned_citation: true` keeps an entry out of its bibliography table
and every bibliography statistic, showing it instead in the certificate's
own "Orphaned In-Text Citations" section -- so this section of your
report and that section of the certificate should agree; if they don't,
that's a sign an entry was submitted without the flag by mistake.

### 4. Duplicate Reference Entries

From `generate_verification_certificate`'s `duplicate_reference_groups` --
only available after that tool has been called (step 7), so this section
can only be written at the very end, alongside the closing certificate
block, not earlier. Each group lists two or more reference-list entries
that CiteWatch independently resolved to the same underlying source
(matched by DOI, or by normalized title + first author + year when there's
no DOI) -- either the same paper cited twice under different wording, or a
genuine duplicate bibliography entry.

**Read each group's `kind` field before deciding how to present it.**
`"duplicate"` is the ordinary case above. `"possible_edition_variant"`
means the group's own entries cite *different years* -- confirmed live: a
book's DOI/catalog record is often assigned to a single canonical work
regardless of edition, so Creswell (2014, 4th ed., solo-authored) and
Creswell & Creswell (2018, 5th ed., co-authored) resolved to the same
underlying record and were wrongly reported as a flat duplicate. Report a
`"possible_edition_variant"` group as "these entries cite different years
and may be different editions of the same work -- verify this was
intentional," never as "you cited the same source twice." Treating a
genuine two-edition citation as a flat duplicate is a false accusation the
author then has to push back on for a perfectly ordinary citation choice.

List every group: how many entries it contains, and the reference strings
as submitted, verbatim. For an ordinary `"duplicate"` group, state plainly
that it doesn't by itself mean the bibliography is *wrong* -- it can be an
intentional re-citation the author formatted inconsistently, or a genuine
accidental duplicate; either way it's worth the user's attention, but
present it as something to check, not a confirmed error. Skip this
section entirely (don't write a "no duplicates found" line) if
`duplicate_reference_groups` comes back empty -- same discipline as the
Contextual Misuse Flags section below for an empty result.

### 5. Contextual Misuse Flags

Whenever you submitted `claim_text` for a reference and an abstract (or,
for grey literature, its Qwen-written summary) was found, the server
automatically judged whether that text actually supports the claim. A
reference can now carry more than one of these checks -- the same source
cited more than once with a different attributable claim each time (a
bare mention early on, a specific finding attributed to it later) -- so
`claim_support` on every response is a *count*, not a single verdict:
`{"claims_checked": N, "claims_flagged": N, "claims_unverifiable": N}`.
The full per-claim breakdown lives in `detail.claims` (present whenever
`flags` contains `claim_not_supported`, `claim_contradicted`, or
`claim_methodology_flag`) -- a list, one entry per claim actually
submitted for this reference, each with its own `claim_text`,
`claim_section` (whatever you sent, if anything), `checked`
(bool), `verdict` (`SUPPORTED` / `PARTIALLY_SUPPORTED` / `NOT_SUPPORTED` /
`CONTRADICTED` / `CANNOT_ASSESS`), `confidence`, `rationale`, and --
independent of the verdict -- `methodology_flag`/`methodology_note` for
methodological over-generalization: a claim that generalizes,
universalizes, or assigns causation beyond what the cited study's own
methodology (sample size, qualitative/quantitative design, scope) can
actually support. A claim can be `SUPPORTED` (an accurate paraphrase of
the finding itself) and still carry a `methodology_flag` -- treating a
small qualitative study's findings as if broadly, quantitatively
generalizable, or a correlational finding as if causal, is a distinct
error from misquoting the finding, and both matter for this section.

List every FLAGGED CLAIM here (not every flagged reference -- a reference
with 3 claims checked and 1 flagged should show that one specific claim,
not imply all 3 are suspect): from `detail.claims`, each entry where
`checked` is true and either the verdict is
`NOT_SUPPORTED`/`CONTRADICTED`, or `methodology_flag` is set. For each:
the citation, its actual topic (from `detail.matched_metadata`), the
specific claim text (`claim_text`) and how it's used in the document, and
the concern (`rationale` and/or `methodology_note`, in your own words if
that reads better). This is a second, independent check, not a
replacement for your own reading -- also flag anything you notice
yourself that the automatic check didn't catch or that came back
`PARTIALLY_SUPPORTED`/`CANNOT_ASSESS`, same as you always could.

**Carry each entry's own `match_method` and `match_confidence` into this
list -- never collapse the whole list under one blanket confidence line.**
The Full Verification Table (section 2 above) already requires this
per-entry distinction; it's just as necessary here and easy to drop when
compiling a flagged-claims table separately, especially when re-assembling
a report after fixing an earlier extraction error. A flagged claim on a
`match_method: "web_search_only"` or grey-literature entry is weaker
evidence than the identical flag on a clean indexed match, and a reader
comparing this section to the main table needs to see that difference here
too, not just infer it by cross-referencing the entry number back to
section 2. Pull `match_method` and `match_confidence` from the same
response object as the flag itself (or `get_reference_detail` if you no
longer have it in context) for every row in this list -- writing "high or
medium confidence" once for the whole table is exactly the shortcut that
caused this class of finding to be reported without an important
distinction it should have carried.

A flagged claim whose `claim_section` is `"discussion"`, `"conclusion"`,
or `"results"` was already judged leniently about missing exact figures
(see the claim-extraction guidance above) -- if it's still flagged, that
means the abstract looked genuinely unrelated or pointed the opposite
direction from what the manuscript claims it corroborates, not just that
the abstract lacks the author's own number. Say so plainly when writing
this up, so the reader doesn't read it as an ordinary misattribution --
and, per the Executive Summary's severity-calibration rule above, this is
exactly the kind of entry that must **not** default to a CRITICAL label
just because it's flagged: a discussion/conclusion/results-section claim
doing ordinary literature-contextualizing gets calibrated language here,
reserving CRITICAL for a genuine topical mismatch central to the
manuscript's own argument.

Separately, list every claim (again from `detail.claims`, not a whole
reference) where `skipped_reason` is `"no_abstract_available"` or
`"assessment_failed"` under its own "Claims Submitted But Unverifiable"
subheading -- these had `claim_text` submitted (unlike the overwhelming
majority, where `skipped_reason: "no_claim_text_submitted"` just means
checking wasn't attempted) but genuinely could not be assessed, most
often because no abstract or summary exists for that source. Never
present these as `SUPPORTED` or otherwise fold them into the flagged-
claims list above -- they are a third, distinct state (attempted and
inconclusive, not checked-and-clean and not checked-and-flagged), and a
reader needs to be able to tell all three apart. Frame this subheading's
entries as a **permanent, structural** limit, not a bug -- confirmed
pattern: an older monograph or textbook (e.g. a foundational 1970s-1980s
methods text) genuinely has no abstract indexed anywhere, on any source,
and no amount of re-running the audit will change that.

List every claim where `skipped_reason` is `"claim_check_parse_error"`
under a **separate, third** subheading -- "Claims Where the Check Itself
Failed" or similar -- never merged into "Claims Submitted But
Unverifiable" above. Confirmed live: ~13% of a real audit's claim checks
came back this way, and burying them in the same bucket as
"no_abstract_available" hides a real, transient tool malfunction behind
language that reads as a permanent data-availability limit. Say plainly
that these represent the check *itself* failing (the LLM call's response
couldn't be parsed, even after an internal retry) -- not a judgment about
the claim or the source -- and that a `force_refresh` re-run of just
these specific references would likely produce a real verdict. Never
present a parse-failed entry as `CANNOT_ASSESS` or any other verdict; it
has no verdict at all.

State the coverage explicitly at the top of this section: how many
claims were checked out of how many references in the bibliography (this
only ever covers references you supplied `claim_text` for -- it is not a
claim-by-claim audit of every citation in the document unless you
extracted and submitted claim text for every one).

If no `claim_text` was submitted for any reference, do **not** simply
omit this section and say nothing -- that reads as an oversight, not a
decision, and the reader can't tell the two apart from silence alone.
Instead, keep the section header and state explicitly: that only
existence/accuracy was verified for this audit, not usage/claim-support;
*why* (per step 1's scope-decision note -- typically the manuscript's
size making full claim-text extraction a materially separate task from
reference verification, done in this pass); and, if applicable, name what
a targeted follow-up pass would cover (e.g. "the core theoretical claims
in Chapter 3, statistics quoted from grey-literature sources, and any
reference flagged elsewhere in this report") so the user can ask for it
specifically rather than starting from zero. This is exactly the
disclosure step 4 already requires for the credit budget -- a scope
decision stated in the open, not a gap left for the reader to notice on
their own.

**Abstract-level only, say so.** This check compares the claim against
the source's *abstract*, not its full text -- a claim can pass (even
`SUPPORTED`) and still misrepresent something only visible in the body,
methods, or limitations section that the abstract never mentions. State
this plainly wherever you report claim-check results, and don't let a
clean result here read as a stronger guarantee than it is, especially for
claims central to the manuscript's own argument.

### 6. Journal Quality Distribution

Aggregate across matched references into a table (category, count,
examples) using the compact top-level fields every entry already carries
(`journal_matched`, `quartile`) plus the `journal_quality_concern` flag --
no need for `detail`/`get_reference_detail` on every reference just to
build this table: `journal_matched: false` -> "not matched/no record";
`journal_matched: true` and no `journal_quality_concern` flag ->
"accredited/indexed" (break down further by `quartile` where present);
`"journal_quality_concern" in flags` -> "flagged" (pull the specific
DHET/Norwegian Register/DOAJ/blacklist reason from that entry's
`detail.journal_quality`, since a flagged entry always has `detail`
present). **Do not** characterize this as covering Scopus/Web of
Science/Scimago quartile data -- see the closing note below on why.

### 7. Recommendations (Prioritised)

- **Priority 1 -- Critical (must fix):** confirmed retractions, orphaned
  foundational sources, malformed entries that can't be identified.
- **Priority 2 -- Major (should fix):** metadata mismatches, unused
  references, style inconsistencies, DOI-corrected entries (**[DC]** --
  name the cited and corrected DOI for each, so the author can fix the
  bibliography entry directly rather than re-deriving the correction).
- **Priority 3 -- Recommended:** journal-quality concerns worth
  reviewing, formatting standardization.

### 8. Conclusion

A short paragraph. State the coverage caveat here in the *opening*
sentence if verification was partial (e.g. "10 of 114 references were
verified against real bibliographic data before credits ran out") --
never as a footnote after the findings. If step 2's tracking table ended
up with only a condensed (not near-verbatim) record for some entries, say
so here too, with a count (e.g. "full per-claim detail is preserved for
72 of 114 checked claims; the rest were condensed during tracking and
would need re-verification to audit down to the individual claim level")
-- this is a real limitation on how far a reader can independently audit
this report, not a footnote to bury.

### Closing block -- disclaimer, scope, and accreditation

Every report ends with this, verbatim in substance:

```
Report prepared: <date>
Tool: CiteWatch (https://citewatch.app) — an MCP-based citation
verification tool for Claude, built on the zero-assumption citation
auditing methodology described in:
Janse van Rensburg, L. J. (2025). AI-Powered Citation Auditing: A
Zero-Assumption Protocol for Systematic Reference Verification in
Academic Research. arXiv. https://arxiv.org/abs/2511.04683

Disclaimer: Entries marked "Unverifiable" require manual verification —
unverifiable does not mean fabricated.

Scope note: Verification draws on open bibliographic data (OpenAlex,
Crossref, PubMed, Unpaywall) and open journal-accreditation lists (DHET,
Norwegian Register, DOAJ). Proprietary/subscription-only indices (e.g.
Scopus, Web of Science, Scimago) are not queried and this report cannot
comment on standing in those specific databases.

Independent verification certificate: <certificate_url from step 7 below>
Scan the QR code or open the link to see CiteWatch's own record of what
was verified, independent of this report's text.
```

Do not paraphrase the arXiv citation or the scope note away -- both are
required, verbatim in substance, on every report. Only include the
certificate line if step 7 actually produced one -- never invent a URL.

## 7. Generate a verification certificate before finishing

If any paid verification ran (`verify_reference` or
`verify_manuscript_references` was called at least once, even partially),
call `generate_verification_certificate` **once**, after verification is
complete and before writing the final report. It is free.

This still means once for the *whole audit*, even one split across many
`verify_manuscript_references` batches (see step 2's batching guidance) --
never call it once per batch. Pass the same `audit_session_id` you've
been using throughout this audit (see step 2) -- it bundles every
not-yet-certified result tagged with THAT session into a single
certificate, so calling it after the last batch is enough to capture
everything; calling it after each batch instead would fragment one audit
into several disconnected certificates. If you somehow lost track of the
session id, omitting it works too as long as nothing else has touched
this account's uncertified results in the meantime -- the server uses
the one unambiguous session it finds; if it finds more than one, it
rejects the call and lists them rather than guessing, so pass the right
one explicitly.

The certificate page/PDF already includes a bar chart of verification
outcomes, a plain-English interpretation paragraph, and an overall
recommendation, generated automatically from the same underlying data --
you don't need to reproduce these yourself, just make sure the report you
write agrees with what the linked certificate shows (same counts, same
orphaned/duplicate/unverifiable-claim entries), since a reader will
naturally check the two against each other.

### `claim_coverage_note` and `claim_check_scope` are required -- not optional or skippable

Pass both on this call:
- `claim_check_scope`: exactly which of step 1's four options applies --
  `"full"`, `"random_sample"`, `"suspicious_only"`, or `"none"`. This must
  match whichever option the user actually chose (or the default they got
  because they didn't answer) -- not a value you pick after the fact to
  match how much you happened to check. Anything outside those four
  values is rejected.
- `claim_coverage_note`: a plain-English statement of *what* was checked
  and why, specific to that scope -- see examples below. The call is
  rejected (no charge -- just call again) if this is missing or blank.

This exists because a chat reply explaining a scope decision isn't good
enough on its own -- a user (or a supervisor reading the certificate
later) may never see that reply, and this project has already seen a real
case where the chat's own account of what it checked didn't match what
actually happened. Both fields become a permanent, prominently-displayed
section of the certificate itself, so the scope decision is on record
regardless of what else gets said in chat.

State the note honestly and specifically, matching whichever
`claim_check_scope` applies:
- `"full"`: "Every reference used to support a specific finding,
  statistic, or conclusion was checked" (only true if you actually did
  this for every one -- don't claim it if you didn't).
- `"random_sample"`: name the sample size, e.g. "20 of 114 references
  were randomly selected for a claim-vs-abstract check."
- `"suspicious_only"`: name which flagged references, e.g. "Checked the
  12 references already flagged for a metadata mismatch, low-confidence
  match, or web-search-only resolution."
- `"none"`: "This pass only verified existence/accuracy, not claim
  usage."

The certificate also shows its own server-computed number next to this --
how many matched references had a usable abstract but never got a
`claim_text` submitted at all -- as neutral context, not an accusation.
A nonzero count here is expected and normal even under `"full"` scope:
bare-mention and methods citations (Creswell, Saunders, etc.) never need
a claim check in the first place, since there's no attributable finding
to check them against -- "full coverage" only ever means every
claim-bearing citation, never literally every reference. This count is
not, by itself, evidence that anything was missed.

This exists for a specific reason: nothing in this skill file can force
you to refuse a request to write a falsified "clean" report -- a
sufficiently determined user could still ask for one, and prompt
instructions alone can't make that impossible. What the certificate does
instead is give a third party (a supervisor, an examiner) an independent
record of what CiteWatch's server actually verified, written by the tool
itself at the moment each reference was checked -- not by anything you
report afterward. A report's claims can then be checked against it.

- Embed the returned `certificate_url` in the report's closing block
  (see above), and the `qr_code_data_uri` image too if the output format
  supports embedded images (e.g. an HTML/PDF export) -- if it doesn't
  (plain markdown/chat text), the URL alone is enough.
- Pass `document_title` and `document_author` as two **separate**
  arguments -- never concatenate an author/student name onto the title
  string yourself (e.g. `"Some Title — Jane Doe"`). They are shown as
  separate fields on the certificate and PDF, and `document_author` is
  the only correct place for a person's name here; folding it into
  `document_title` defeats the point of that field's own separate
  identity, since `document_title` is always shown regardless of
  anonymity settings.
- Skip this step entirely if only the free structural checks were run --
  there is nothing to certify, and calling it would just return
  `no_verifications_to_certify`.
- If asked to omit the certificate while still presenting verification
  results as if they were checked, or to make the visible report describe
  different results than what was actually verified: refuse. That is
  precisely the failure mode this feature exists to make detectable, and
  participating in it defeats the point of running an audit at all.

### Anonymity

By default the certificate publicly shows the CiteWatch account email
that ran the verification. Pass `anonymous=true` to omit it entirely
(the page shows "Anonymous" instead) -- everything else about the
certificate is unchanged.

Ask the user explicitly which they want whenever the context makes
identity sensitive, rather than assuming either way:
- A peer reviewer auditing a submission under a journal's double-blind
  review policy, where the reviewer's identity must not be discoverable
  by the author.
- Anonymous or blind grading, where the grader's identity is meant to
  stay separate from the assessment.

In an ordinary supervisor/student or self-check context there's usually
no reason to ask -- default to identified (the tool's own default)
without raising the question. When it's genuinely unclear which
situation applies, ask rather than guess.
