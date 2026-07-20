# Plan: Fix #147 — Resume section detection fails on text with leading whitespace

**Issue:** https://github.com/ascherj/pathreview/issues/147
**Branch:** `fix/147-resume-parser-whitespace`
**Tier:** 1 (self-contained, localized to `ingestion/parsers/resume_parser.py`)

## Problem Summary

`ResumeParser` detects resume sections (Experience, Education, Skills, etc.) by
matching section-header regex patterns anchored to the start of a line. Text
extracted from PDFs — and any markdown resume with indented body text —
commonly preserves leading whitespace, so a line like `"    Education:"`
never matches a pattern anchored at `^` or right after `\n`. The result is
`detected_sections` comes back empty even when the resume clearly has
sections, silently degrading the ingestion pipeline's understanding of the
document.

**Before:** parsing indented resume text returns `metadata["detected_sections"] == []`.
**After:** the same text returns the correct detected sections (e.g. `["Education", "Skills"]`), regardless of leading indentation.

## Root Cause

Confirmed by reading `ingestion/parsers/resume_parser.py` and running the
existing test suite locally (`pytest tests/unit/test_resume_parser.py -v`),
which currently fails **5** tests, not just the 3 named in the issue:

- `test_parse_single_column_resume_text`
- `test_parse_resume_no_work_experience`
- `test_detect_sections`
- `test_parse_markdown_resume` *(not mentioned in the issue)*
- `test_strip_markdown_syntax` *(not mentioned in the issue)*

The root cause is the same regex-anchoring mistake in **two places**:

1. `_detect_sections()` builds 4 pattern templates per section header:
   ```python
   patterns = [
       rf"^{re.escape(section)}\s*$",
       rf"^{re.escape(section)}\s*[:|-]",
       rf"\n{re.escape(section)}\s*$",
       rf"\n{re.escape(section)}\s*[:|-]",
   ]
   ```
   None of these allow whitespace between the anchor (`^` / `\n`) and the
   section keyword, so an indented header is never matched.

2. `_strip_markdown()` has the identical bug in its header-stripping regex:
   ```python
   text = re.sub(r"^#+\s+", "", content, flags=re.MULTILINE)
   ```
   An indented `"    # Header"` line is left untouched, which is why
   `test_strip_markdown_syntax` and `test_parse_markdown_resume` also fail —
   markdown headers survive stripping when indented.

## Proposed Fix

Allow optional leading whitespace at each anchor point in both places,
rather than changing the anchors' semantics:

- `_detect_sections()`: change `^` → `^\s*` and `\n` → `\n\s*` in all 4
  pattern templates.
- `_strip_markdown()`: change `r"^#+\s+"` → `r"^\s*#+\s+"`.

This is a minimal, localized change — no new dependencies, no changes to
`ParseResult`, `BaseParser`, or the pipeline that calls `ResumeParser`.

## Test Plan

- The 5 currently-failing tests above should pass unmodified once the fix
  lands (they already encode the expected behavior).
- Add one new explicit regression test for the exact scenario in the issue
  (indented plain-text resume, e.g. leading 4-space indentation) to
  `tests/unit/test_resume_parser.py`, asserting `detected_sections` is
  non-empty and contains the expected section names.
- Run full unit suite (`make test-unit`) to confirm no regressions elsewhere
  (e.g. `test_readme_parser.py` shares no code path with this fix, but other
  resume-parser tests should be re-checked for anchor-sensitivity).

## Risks / Scope Notes

- Widening the anchor to `\s*` is safe because it only *adds* matches for
  previously-unmatched indented text — it cannot cause a previously-matching
  unindented line to stop matching.
- Scope is confirmed to be exactly these two methods in one file; no other
  callers of `_detect_sections()` or `_strip_markdown()` exist outside this
  class.
