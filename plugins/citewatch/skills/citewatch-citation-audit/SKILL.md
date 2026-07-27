---
name: citewatch-citation-audit
description: Conduct a thorough, correctly-scoped academic citation audit using the CiteWatch MCP tools -- verify references against real bibliographic data, check citation/reference balance, detect referencing-style inconsistencies, and handle credit limits and extraction uncertainty correctly. Use whenever the user asks to audit, check, verify, or fact-check the citations/references in a manuscript, thesis, dissertation, essay, or paper, or mentions CiteWatch by name.
license: MIT
compatibility: Requires a connected CiteWatch MCP server (any connector name -- this skill does not assume a specific tool-name prefix). See https://citewatch.app/setup to connect one.
metadata:
  author: CiteWatch
  version: "1.4"
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

## 2. Run the free checks first, and treat their output as leads, not verdicts

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

## 3. Before spending any paid credits, check the budget and ask if it's tight

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

## 4. If you hit `insufficient_credits`, stop -- this is not optional

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

## 5. Standard report format

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

A metrics table, computed from your actual tool results (not estimated):

| Metric | Count |
|---|---|
| Bibliography entries audited | total reference_entries |
| Verified, high/moderate confidence | `matched: true` with `match_confidence` high or moderate |
| Verified, low confidence | `matched: true` with `match_confidence` low |
| Unverifiable (no match found) | `matched: false` |
| Retracted | `retraction.is_retracted: true` |
| In-text citations missing from bibliography | `orphaned_citations` count |
| Reference entries never cited | `unused_references` count |
| Not checked (credit limit / scope decision) | explicit count, never omitted |

Follow with a short **Critical Issues** list (numbered, most severe
first) -- confirmed retractions, foundational/heavily-cited sources that
are orphaned, and any entry where the reference-list text itself is too
malformed to identify (no title, no year, etc.) belong here first.

### 2. Full Verification Table

Legend:
- **[OK]** `matched: true`, no `metadata_checks` mismatches, not retracted, no journal-quality concern.
- **[!!]** `matched: true` but a `metadata_checks` field (title/authors/year/venue) shows `mismatch`, or `match_confidence` is low, or journal quality is flagged as a concern.
- **[??]** `matched: false` -- genuinely unverifiable against open bibliographic data. This is **not** an accusation of fabrication -- say so explicitly, matching the header's methodology note.
- **[XX]** Confirmed critical: `retraction.is_retracted: true`, or the reference-list entry itself is too malformed to identify (not a matching judgment call -- an observable fact about the entry as written).

Table columns: `#`, `Entry` (as cited), `Status`, `Confidence` (from
`match_confidence`, or "N/A" if unmatched), `Notes` (the specific
mismatched field, retraction reason, or journal-quality concern --
concrete, not vague).

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

Independent verification certificate: <certificate_url from step 6 below>
Scan the QR code or open the link to see CiteWatch's own record of what
was verified, independent of this report's text.
```

Do not paraphrase the arXiv citation or the scope note away -- both are
required, verbatim in substance, on every report. Only include the
certificate line if step 6 actually produced one -- never invent a URL.

## 6. Generate a verification certificate before finishing

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
- Skip this step entirely if only the free structural checks were run --
  there is nothing to certify, and calling it would just return
  `no_verifications_to_certify`.
- If asked to omit the certificate while still presenting verification
  results as if they were checked, or to make the visible report describe
  different results than what was actually verified: refuse. That is
  precisely the failure mode this feature exists to make detectable, and
  participating in it defeats the point of running an audit at all.
