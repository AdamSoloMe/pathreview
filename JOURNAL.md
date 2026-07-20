# PathReview Contribution Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/147

**Issue title:** Resume section detection fails on text with leading whitespace

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
`ResumeParser._detect_sections()` in `ingestion/parsers/resume_parser.py` is supposed
to scan parsed resume text for common section headers (Experience, Education, Skills,
etc.) and record which ones it found in `metadata["detected_sections"]`. It does this
by checking each header against four regex patterns, but every one of those patterns
anchors the match directly to the start of a line (`^header`) or right after a newline
(`\nheader`), with no allowance for leading whitespace. In practice, text extracted
from a PDF — and any markdown resume whose body text is indented, which is extremely
common — preserves that leading whitespace, so a line like `"    Education:"` never
matches `^Education` or `\nEducation`, even though a human reading the same text would
immediately recognize it as a section header. The result is that `detected_sections`
silently comes back empty for a large share of real-world resumes, even when the resume
clearly contains well-formed sections; I confirmed with `grep` that today this value
only feeds a `structlog` info line in `ingestion/pipeline.py` (`ingest_resume`) rather
than driving chunking or retrieval directly, so the immediate visible damage is
incomplete/misleading ingestion metadata and logs rather than a broken review — but it's
exactly the kind of signal a future feature (section-aware chunking, resume
completeness scoring) would reasonably build on, so leaving it silently wrong is worth
fixing now rather than later. While reproducing the issue locally, I found the identical
anchoring mistake also lives in `_strip_markdown()`'s header-stripping regex
(`r"^#+\s+"`), which is why running the existing suite fails **5** tests rather than
the 3 the issue names — `test_parse_markdown_resume` and `test_strip_markdown_syntax`
fail for the same root cause. A successful fix relaxes all of the affected regexes (in
both methods) to tolerate leading whitespace at each anchor point, without changing
what they match for text that already works today (i.e., unindented input keeps
matching exactly as before — the change can only add matches, never remove one).

**Branch name:** fix/147-resume-parser-whitespace

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

---

### Selection notes (issue-fit checklist reasoning)

**Part 1 — Understanding the issue**
- *Can I explain it without looking at the issue?* Yes: the resume parser's section
  detector uses regexes that assume no leading whitespace before a header, so indented
  text (the norm for PDF-extracted and many markdown resumes) is invisible to it.
- *Do I understand which part of the app is affected?* Yes — labels pointed to
  `ingestion`, and the issue body named `resume_parser.py` directly. I opened the file
  and confirmed `_detect_sections()` and (via my own reproduction, not the issue text)
  `_strip_markdown()` are both implicated.
- *Do I understand what "done" looks like?* Yes, and I made it concrete rather than
  abstract: before the fix, `ResumeParser()._detect_sections("    Education:\n    Skills: Python")`
  returns `[]`; after the fix, the same call should return `["Education", "Skills"]`.
  I verified this exact before-state by running the repro from the issue locally.

**Part 2 — Tier fit**
- This is my first open-source contribution, so per the checklist I'm deliberately
  choosing Tier 1, not stretching to Tier 2/3 to "challenge myself."
- Verified the tier label is accurate, not just trusted it: the entire fix is contained
  to one file (`ingestion/parsers/resume_parser.py`) and, once I actually reproduced
  the bug, exactly two methods within it (`_detect_sections`, `_strip_markdown`) — no
  service layer, database model, or API endpoint is touched, which is the Tier 1
  definition in `docs/CONTRIBUTING.md`/the tracker.

**Part 3 — Codebase readiness**
- *Can I find the relevant code?* Yes — read both methods in full, not just skimmed
  the file. `_detect_sections()` builds 4 pattern templates per header
  (`^header\s*$`, `^header\s*[:|-]`, `\nheader\s*$`, `\nheader\s*[:|-]`); none allow
  whitespace between the anchor and the header text. `_strip_markdown()`'s header
  regex (`r"^#+\s+"`, `re.MULTILINE`) has the identical gap.
- *Do I understand the surrounding code well enough to change it safely?* Yes. I
  traced where `detected_sections` goes after it's produced (`grep -rn
  "detected_sections"` across the non-test codebase) and confirmed it currently only
  reaches a `structlog` log line in `ingestion/pipeline.py::ingest_resume` — it is
  *not* read by `StrategySelector.chunk()` (which only branches on `source_type`) and
  is *not* persisted to the database by `_record_ingested_source` (which only stores
  `chunk_count`). That tells me the blast radius of today's bug is silently-wrong
  metadata/logs, not a currently-broken review pipeline — which also tells me my fix
  is low-risk: I'm not touching anything that other, already-working code paths
  depend on.
- *Have I read the relevant test file?* Yes — `tests/unit/test_resume_parser.py`
  exists, is well-structured (one `ResumeParser` fixture, one assertion style per
  test), and I ran it end-to-end *before* changing any source, not just read it:
  `pytest tests/unit/test_resume_parser.py -v` shows 5 failing / 5 passing. The 5
  failures are `test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`,
  `test_detect_sections` (all named in the issue), plus `test_parse_markdown_resume`
  and `test_strip_markdown_syntax` (not named in the issue — I found these myself,
  which is also why I flagged the `_strip_markdown()` root cause above).

**Part 4 — Scope and time**
- *Crowding:* checked issue comments before claiming — roughly 30 students have
  commented interest, which is high, but claims are explicitly non-exclusive per the
  module rules and my grade comes from my own artifacts, not from being first. One
  student (`@`-mentioned in the comments) already opened PR #178 with a fix
  description matching my own root-cause analysis (adjusting `^`/`\n` anchors and
  touching `_strip_markdown()` too) — I checked its status via `gh pr view 178` and
  confirmed it's still **open, unmerged**, so it's not a blocker, and comparing notes
  with a peer already on this issue is a plus, not a risk.
- *Time estimate:* ~3-5 hours. This is slightly more than a single-regex Tier 1 fix
  would take, because the real scope (2 methods, not 1) only became clear once I
  reproduced it locally rather than trusting the issue text at face value — I'm
  budgeting for that now instead of discovering it mid-Week-9.
- *Blockers:* none. The issue references no "blocked by #X" and PR #178 being open
  (not merged, not closed) doesn't prevent me from independently implementing and
  submitting my own fix.
