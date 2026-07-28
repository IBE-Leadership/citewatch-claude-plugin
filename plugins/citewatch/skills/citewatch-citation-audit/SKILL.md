---
name: citewatch-citation-audit
description: Conduct a thorough, correctly-scoped academic citation audit using the CiteWatch MCP tools -- verify references against real bibliographic data, check citation/reference balance, detect referencing-style inconsistencies, and handle credit limits and extraction uncertainty correctly. Use whenever the user asks to audit, check, verify, or fact-check the citations/references in a manuscript, thesis, dissertation, essay, or paper, or mentions CiteWatch by name.
license: MIT
compatibility: Requires a connected CiteWatch MCP server (any connector name -- this skill does not assume a specific tool-name prefix). See https://citewatch.app/setup to connect one.
metadata:
  author: CiteWatch
  version: "2.0"
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

## 2. Locate the reference list reliably, and track long documents as you go

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

### Large or multi-chapter documents: work from a persistent tracking file

For a document long enough that holding the full state in conversation
context risks losing track of what's been verified (many chapters, a
large reference list, or a per-chapter reference-list structure), don't
try to carry all of it in memory across the whole audit. Instead:

1. Before calling any verification tool, make one full pass across the
   *entire* document (every chapter/section) to build a single merged
   reference list -- combine per-chapter lists into one set, and for
   every unique reference, record **every** chapter/section and
   page (or best available location) where it's cited in-text. The same
   title/author/year appearing in two different chapters is one
   reference cited twice, not two references -- merge it, don't verify
   it twice.
2. Write this out as a working data file (e.g. a JSON or Markdown table)
   before verifying anything -- one row per unique reference, with the
   reference string, every citing location, and a status field starting
   at "not yet checked". Keep orphaned/missing-from-bibliography
   candidates (in-text citations with no matching reference-list entry)
   in a clearly separate field or file from this table -- never merge
   them in, even temporarily, per the extraction-integrity note in step 1
   above; a tool call built from this file must only ever draw from the
   genuine-bibliography rows. If you have file-write access in this
   environment, save and update this file on disk as you go rather than
   holding it only in the conversation -- a long audit can span more
   turns than comfortably fit in one context window, or cross a session
   boundary, and a saved file lets you resume exactly where you left off
   instead of re-deriving progress or silently re-scanning chapters
   you've already covered.
3. Verify chapter by chapter (or in whatever chunks keep each pass
   manageable) against this file: the first time you reach a given
   reference, verify it and mark the file's status field with the
   result; every later chapter that cites the same reference reads the
   already-recorded result instead of calling the tool again. Never
   re-verify a reference just because it recurs later in the document --
   that spends credits on a call whose answer you already have.
4. By the last chapter, the tracking file already holds every
   reference's full citing-location history and verification result --
   build the Full Verification Table and Orphan Citations sections
   directly from it rather than re-deriving them from memory.

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
(per reference). `search_literature` costs 5 credits per call and
`assess_claim_support` costs 3 credits per call -- both cost meaningfully
more per call than a single reference verification, so factor that in
when estimating whether a balance covers a planned sequence of calls, not
just a flat per-tool-call count. Call `get_credit_balance` (free) before
committing to a full-manuscript verification pass. If the reference count
exceeds the available balance,
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

Follow with a short **Critical Issues** list (numbered, most severe
first) -- confirmed retractions, foundational/heavily-cited sources that
are orphaned, and any entry where the reference-list text itself is too
malformed to identify (no title, no year, etc.) belong here first.

### 2. Full Verification Table

Legend:
- **[OK]** `matched: true`, no `metadata_checks` mismatches, not retracted, no journal-quality concern.
- **[!!]** `matched: true` but a `metadata_checks` field (title/authors/year/venue) shows `mismatch`, or `match_confidence` is low, or journal quality is flagged as a concern, or `match_method` is `"web_search"` (see below).
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
  steps fired. Just report the actual number returned -- nothing about
  the insufficient-credits handling above changes.

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

### 4. Contextual Misuse Flags

Where `claim_text` was checked against a retrieved abstract (directly by
you, or via `assess_claim_support`) and the source doesn't actually
support how it's cited, or its real topic doesn't match the claim it's
attached to, list it here: citation, actual topic (from `matched_metadata`),
how it's used in the document, and the concern. Skip this section
entirely if no claim-support checking was done -- don't imply a check
happened when it didn't.

### 5. Journal Quality Distribution

Aggregate `journal_quality` across matched references into a table
(category, count, examples) -- e.g. accredited/indexed, not
matched/no record, flagged by DHET/Norwegian Register/DOAJ standing.
**Do not** characterize this as covering Scopus/Web of Science/Scimago
quartile data -- see the closing note below on why.

### 6. Recommendations (Prioritised)

- **Priority 1 -- Critical (must fix):** confirmed retractions, orphaned
  foundational sources, malformed entries that can't be identified.
- **Priority 2 -- Major (should fix):** metadata mismatches, unused
  references, style inconsistencies.
- **Priority 3 -- Recommended:** journal-quality concerns worth
  reviewing, formatting standardization.

### 7. Conclusion

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
