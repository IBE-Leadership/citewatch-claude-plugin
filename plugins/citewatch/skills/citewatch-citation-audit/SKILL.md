---
name: citewatch-citation-audit
description: Conduct a thorough, correctly-scoped academic citation audit using the CiteWatch MCP tools -- verify references against real bibliographic data, check citation/reference balance, detect referencing-style inconsistencies, and handle credit limits and extraction uncertainty correctly. Use whenever the user asks to audit, check, verify, or fact-check the citations/references in a manuscript, thesis, dissertation, essay, or paper, or mentions CiteWatch by name.
license: MIT
compatibility: Requires a connected CiteWatch MCP server (any connector name -- this skill does not assume a specific tool-name prefix). See https://citewatch.app/setup to connect one.
metadata:
  author: CiteWatch
  version: "2.6"
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

Treat this the same way step 4 treats the credit budget: decide the scope
*before* running the verification batches, not after, whenever full
coverage isn't practical in one session -- ask the user, with real
options, rather than silently deciding for them:

1. Full coverage -- extract and submit `claim_text` for every reference
   used to support a specific finding (the materially larger task above).
2. A targeted subset -- `claim_text` only for the highest-value
   references: core theoretical/framework claims, specific statistics
   quoted from grey-literature or non-peer-reviewed sources, and any
   reference already flagged elsewhere (a metadata mismatch, low
   confidence, web-search-only match) where a wrong or misused source
   would matter most.
3. Existence/accuracy verification only -- skip claim-support/usage
   checking entirely for this pass.

If there's no chance to ask (an unattended/automated run), reason it
through yourself, pick the option the manuscript's size and stakes
actually justify, and **say so explicitly in the report** -- see step
6.5's Contextual Misuse Flags section, which now requires this decision
to be stated outright rather than left to be inferred from an absent
section. A reader should never have to guess whether usage/claim-support
checking was considered and deliberately scoped down, or simply never
occurred to you.

## 2. Locate the reference list reliably, and track long documents as you go

### The mandatory procedure: eat the elephant one bite at a time

Never treat a manuscript as one undifferentiated pass -- extract
everything, then verify everything, then report. That's exactly how
entries get dropped, merged, or silently skipped on anything longer than
a few pages. Follow these steps, in this order, every time:

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
reach a given reference, verify it and record the result in this table;
every later unit that cites the same reference reads the already-recorded
result instead of calling the tool again -- never re-verify a reference
just because it recurs later in the document, that spends credits on a
call whose answer you already have. Keep orphaned/missing-from-bibliography
candidates in their own clearly separate rows (or a separate file) from
genuine bibliography rows in this same table -- both get sent to
CiteWatch per Step C.5 below, but they must never be blended together
when you build a bibliography tool-call payload, or in the report's own
figures later.

If you have file-write access in this environment, save and update both
the todo list and the tracking table on disk as you go, rather than
holding them only in the conversation.

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
   to this unit's `verify_manuscript_references` batch (or read the
   already-recorded result from the tracking table if another unit
   verified it first).
5. **If no matching entry is found (an orphaned in-text citation)**, send
   it to CiteWatch anyway, using whatever text the in-text citation
   itself gives you (e.g. `"Smith, 2020"`) as the `reference_string` --
   do not just note it locally and skip the server entirely. CiteWatch's
   own database counts and tracks these attempts regardless of whether a
   full reference existed to send; leaving them out means CiteWatch never
   learns about them, and the audit's own record of what was attempted is
   incomplete. This still costs 1 credit per attempt like any other
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

**On batch sizing** (independent of the unit-by-unit plan above): a
single chapter or section can have anywhere from a handful to 100+
references, so batch the actual `verify_manuscript_references` **calls**
by size, not by unit boundary -- send **~10 references per call**
(smaller, ~5, if you're also sending `claim_text` for most entries in the
batch -- the automatic claim-vs-abstract check adds a real extra step per
reference, so a batch that size takes meaningfully longer than the same
size did before that feature existed), not a single huge call for an
entire chapter or the whole bibliography (a very large batch can run
several minutes, especially if several references need the automatic
web-search/scrape escalation or claim check, and a long-running single
call has more that can interrupt it along the way). If a batch call
errors out or the connection drops partway through, just call it again
with the same batch (or continue with the remaining references) -- the
server detects anything already verified and not yet certified and
returns it for free (`resumed_from_earlier_call: true`) instead of
redoing the work and charging twice, so retrying is always safe.

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
| Verified, high/moderate confidence | `matched: true` with `match_confidence` high or moderate, excluding `match_method: "web_search"` |
| Verified, low confidence | `matched: true` with `match_confidence` low |
| Verified via web search only (not index-corroborated) | `match_method: "web_search"` -- see below |
| Grey literature / non-academic source | `escalation.grey_literature` present -- see **[GL]** below |
| Unverifiable (no match found) | `matched: false` and no `escalation.grey_literature` |
| Retracted | `retraction.is_retracted: true` |
| Metadata/completeness mismatches (title/authors/year/venue/volume/issue/pages) | at least one `metadata_checks` field has `status: "mismatch"` -- percentaged against `matched` count, not total, since an unmatched entry has nothing to compare against |
| Duplicate reference entries | from `generate_verification_certificate`'s `duplicate_reference_groups` (only available after that tool has been called -- see its own section below) |
| In-text citations missing from bibliography | `orphaned_citations` count |
| Reference entries never cited | `unused_references` count |
| Not checked (credit limit / scope decision) | explicit count, never omitted |

The "Verified via web search only" and "Grey literature / non-academic
source" rows exist so a reader can see at a glance how much of the
bibliography rests on a weaker or different kind of verification than a
direct bibliographic-index match -- never fold these into the plain
"Verified"/"Unverifiable" counts above them, and never omit them even
when their count is zero (write `0 (0%)` explicitly, same discipline as
"Not checked").

If any `claim_text` was submitted anywhere in the audit, add two more
rows immediately after the table above, percentaged against the number of
claims actually **checked** (i.e. had `claim_support.checked: true`), not
against total bibliography entries -- most entries typically won't have
had a claim submitted at all, so "% of total" would understate things:

| Metric | Count (% of claims checked) |
|---|---|
| Cited claims checked against abstract | total `claim_support.checked: true` count (this row alone is % of total bibliography entries, not of itself) |
| Flagged: unsupported, contradicted, or methodology mismatch | `claim_support.verdict` in `NOT_SUPPORTED`/`CONTRADICTED`, or `claim_support.methodology_flag` set |

Omit both rows entirely if no claims were checked at all (rather than
writing `0 (0%)`) -- unlike the escalation rows above, which reflect
something the server does automatically for every reference, this only
ever reflects a scope decision you made about which claims to submit, so
its absence needs no explicit zero to be self-explanatory.

Follow with a short **Critical Issues** list (numbered, most severe
first) -- confirmed retractions, foundational/heavily-cited sources that
are orphaned, and any entry where the reference-list text itself is too
malformed to identify (no title, no year, etc.) belong here first.

### 2. Full Verification Table

Legend:
- **[OK]** `matched: true`, no `metadata_checks` mismatches, not retracted, no journal-quality concern.
- **[!!]** `matched: true` but a `metadata_checks` field (title/authors/year/venue/volume/issue/pages) shows `mismatch`, or `match_confidence` is low, or journal quality is flagged as a concern, or `match_method` is `"web_search"` (see below). A volume/issue/pages mismatch can mean the wrong paper was matched, but just as often means the reference-list entry itself is incomplete or has a typo (e.g. a missing volume number) -- present it as "check this entry's completeness," not as an accusation that the wrong source was found.
- **[??]** `matched: false` -- genuinely unverifiable against open bibliographic data. This is **not** an accusation of fabrication -- say so explicitly, matching the header's methodology note.
- **[GL]** `escalation.grey_literature` is present -- not an academic paper (a government report, industry white paper, standard, etc. with no index entry to match against), but the server captured its full text and wrote a summary directly from the cited source. Present `escalation.grey_literature.summary` here, and don't mark it "Unverifiable" -- it's a different, more informative status than a plain no-match.
- **[XX]** Confirmed critical: `retraction.is_retracted: true`, or the reference-list entry itself is too malformed to identify (not a matching judgment call -- an observable fact about the entry as written).

**Automatic web-search/scrape escalation.** The server automatically falls
back to a paid web search/scrape when the free sources (Crossref/OpenAlex/
PubMed/Unpaywall) can't resolve something on their own -- a missing
abstract with a known link, grey literature, or no match at all. This is
fully automatic; there is nothing you need to do differently to trigger
it. It surfaces in the response as:
- `match_method: "web_search"` -- matched, but only via an independent web
  search, never corroborated against a structured bibliographic index.
  Always treat as **[!!]**, never as a plain **[OK]**, regardless of
  `match_confidence`.
- `escalation.verification_note` -- a short explanation to carry into your
  `Notes` column whenever present (e.g. why a web-search match should be
  treated with extra caution, or that a second opinion corroborated an
  originally medium-confidence match).
- `escalation.grey_literature` -- see **[GL]** above.
- `credits_charged` may include a small fractional amount on top of the
  flat per-reference cost (e.g. `1.0072` instead of `1`) when one of these
  steps fired, or when the claim-vs-abstract check ran (see step 6.5).
  Just report the actual number returned -- nothing about the
  insufficient-credits handling above changes.

Table columns: `#`, `Entry` (as cited), `Status`, `Confidence` (from
`match_confidence`, or "N/A" if unmatched), `Notes` (the specific
mismatched field, retraction reason, or journal-quality concern --
concrete, not vague). For a document processed via the tracking file in
step 2, add a `Cited from` column listing every chapter/page location on
record for that reference, instead of just the first one encountered.

### 3. Orphan Citations

From `check_citation_reference_balance`'s `orphaned_citations` and
`unused_references`. Carry forward that tool's `extraction_disclaimer`
in substance: these are candidates for the user to check against their
own document, not confirmed gaps. Flag foundational/theory-defining
sources specifically if orphaned -- an examiner or reviewer notices
those fastest.

Per step 2's mandatory procedure, each orphaned citation was also sent to
CiteWatch on its own (using the bare in-text citation as `reference_string`,
since no bibliography entry existed to send instead) -- report that
result here too, next to the citation itself (e.g. "CiteWatch attempted
an independent bibliographic search on this citation alone and found no
match" or, on the rare occasion the bare citation is enough to resolve,
whatever it did find). Never fold these results into the Full
Verification Table or Executive Summary counts above -- they describe a
different thing (no genuine reference-list attribution exists to verify
against) and belong only in this section.

### 4. Duplicate Reference Entries

From `generate_verification_certificate`'s `duplicate_reference_groups` --
only available after that tool has been called (step 7), so this section
can only be written at the very end, alongside the closing certificate
block, not earlier. Each group lists two or more reference-list entries
that CiteWatch independently resolved to the same underlying source
(matched by DOI, or by normalized title + first author + year when there's
no DOI) -- either the same paper cited twice under different wording, or a
genuine duplicate bibliography entry.

List every group: how many entries it contains, and the reference strings
as submitted, verbatim. State plainly that a duplicate group doesn't by
itself mean the bibliography is *wrong* -- it can be an intentional
re-citation the author formatted inconsistently, or a genuine accidental
duplicate; either way it's worth the user's attention, but present it as
something to check, not a confirmed error. Skip this section entirely
(don't write a "no duplicates found" line) if `duplicate_reference_groups`
comes back empty -- same discipline as the Contextual Misuse Flags
section below for an empty result.

### 5. Contextual Misuse Flags

Whenever you submitted `claim_text` for a reference and an abstract was
found, the server automatically judged whether the abstract actually
supports the claim, returning this on that reference's result as
`claim_support`: `checked` (bool), `verdict` (`SUPPORTED` /
`PARTIALLY_SUPPORTED` / `NOT_SUPPORTED` / `CONTRADICTED` /
`CANNOT_ASSESS`), `confidence`, `rationale`, and -- independent of that
verdict -- `methodology_flag`/`methodology_note` for methodological
over-generalization: a claim that generalizes, universalizes, or assigns
causation beyond what the cited study's own methodology (sample size,
qualitative/quantitative design, scope) can actually support. A citation
can be `SUPPORTED` (an accurate paraphrase of the finding itself) and
still carry a `methodology_flag` -- treating a small qualitative study's
findings as if broadly, quantitatively generalizable, or a correlational
finding as if causal, is a distinct error from misquoting the finding,
and both matter for this section.

List every entry here where `claim_support.checked` is true and either
the verdict is `NOT_SUPPORTED`/`CONTRADICTED`, or `methodology_flag` is
set: citation, actual topic (from `matched_metadata`), how it's used in
the document, and the concern (`claim_support.rationale` and/or
`methodology_note`, in your own words if that reads better). This is a
second, independent check, not a replacement for your own reading --
also flag anything you notice yourself that the automatic check didn't
catch or that came back `PARTIALLY_SUPPORTED`/`CANNOT_ASSESS`, same as
you always could.

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

Aggregate `journal_quality` across matched references into a table
(category, count, examples) -- e.g. accredited/indexed, not
matched/no record, flagged by DHET/Norwegian Register/DOAJ standing.
**Do not** characterize this as covering Scopus/Web of Science/Scimago
quartile data -- see the closing note below on why.

### 7. Recommendations (Prioritised)

- **Priority 1 -- Critical (must fix):** confirmed retractions, orphaned
  foundational sources, malformed entries that can't be identified.
- **Priority 2 -- Major (should fix):** metadata mismatches, unused
  references, style inconsistencies.
- **Priority 3 -- Recommended:** journal-quality concerns worth
  reviewing, formatting standardization.

### 8. Conclusion

A short paragraph. State the coverage caveat here in the *opening*
sentence if verification was partial (e.g. "10 of 114 references were
verified against real bibliographic data before credits ran out") --
never as a footnote after the findings.

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
never call it once per batch. It automatically bundles every
not-yet-certified result across every prior call on the account into a
single certificate, so calling it after the last batch is enough to
capture everything; calling it after each batch instead would fragment
one audit into several disconnected certificates.

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
