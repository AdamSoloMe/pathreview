# PathReview Contribution Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/147

**Issue title:** Resume section detection fails on text with leading whitespace

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
`ResumeParser._detect_sections()` in `ingestion/parsers/resume_parser.py` looks for
section headers (Experience, Education, Skills, etc.) using regex patterns anchored
directly to the start of a line or right after a newline. Text extracted from PDFs,
and indented markdown resumes, commonly has leading whitespace before each heading,
so the anchors never match and `detected_sections` comes back empty even though the
resume clearly has sections. While reproducing this locally I found the same
anchoring bug also breaks `_strip_markdown()`'s header-stripping regex, which is why
5 unit tests fail rather than the 3 named in the issue. A successful fix relaxes both
regexes to tolerate leading whitespace without changing what they match for
already-working (non-indented) input.

**Branch name:** fix/147-resume-parser-whitespace

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

---

### Selection notes (issue-fit checklist reasoning)

**Part 1 — Understanding:** Confirmed by running `pytest tests/unit/test_resume_parser.py -v`
before making any change: 5 tests fail (`test_parse_single_column_resume_text`,
`test_parse_resume_no_work_experience`, `test_detect_sections`,
`test_parse_markdown_resume`, `test_strip_markdown_syntax`). Before/after is concrete:
indented resume text currently returns `detected_sections: []`; after the fix it
should return the correct section names regardless of indentation.

**Part 2 — Tier fit:** First open-source contribution, so Tier 1 is the right call per
the checklist. Confirmed the fix is fully contained to one file
(`ingestion/parsers/resume_parser.py`), two methods.

**Part 3 — Codebase readiness:** Read `_detect_sections()` and `_strip_markdown()` in
full, not just the file. An existing, well-structured test file
(`tests/unit/test_resume_parser.py`) is available as a pattern to extend — read at
least one full test end-to-end before starting. Root cause for both methods is the
same class of bug (regex anchors with no allowance for leading whitespace), so the
fix is predictable and low-risk: relaxing `^`/`\n` to `^\s*`/`\n\s*` only adds matches,
it cannot break an already-passing (non-indented) case.

**Part 4 — Scope and time:** Checked issue comments — heavily claimed (~30 comments)
but claims are non-exclusive per the module rules, and one open PR (#178) exists but
is unmerged, so no blocker. Estimated 3-5 hours given the fix spans two methods
instead of one. No open blockers/dependencies referenced in the issue.
