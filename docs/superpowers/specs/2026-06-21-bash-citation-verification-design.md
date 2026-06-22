# Citation content verification — design

Date: 2026-06-21
Status: approved (brainstorming) — ready for implementation plan
Scope: `scripts/grounding_spec.py`, `scripts/grounding-verifier.py`, tests

Covers two citation kinds via **one** verbatim-quote (backtick) convention:
- **Bash output** — move `Bash(<cmd>) — <output>` from *asserted* toward
  *verified* (the original focus of this spec).
- **File-line content** — add an opt-in content check on top of the existing
  `Read`/`Edit`/`Write`/`MultiEdit` pointer check.

## Problem

- Bash citations are treated as **asserted (unchecked)** — honest, but the most
  common recorded-output citation goes entirely unverified.
- File citations are **pointer-only**: the verifier checks the file exists, the
  line is in range, and the file/line was opened — but never compares the cited
  prose to the actual content at that line.

We want to ground both, deterministically, without re-executing anything, using
one shared mechanism.

## Key realization

For both kinds, **the evidence is the content, not the locator.** For Bash the
locator is the command (a label); for a file it's `path:line` (a pointer). In
both cases the trustworthy check is: *does the quoted content actually appear in
the real source?*

The hard part is telling a *verbatim claim* apart from the author's *prose*. We
solve it with an explicit **backtick convention**: the author wraps spans they
claim are verbatim in backticks, and the verifier checks exactly those. This
moves "what is a checkable claim" from a fragile guess to an explicit signal —
and the same rule serves both sources. The transcript already holds Bash
commands + full output + `is_error`; files are re-readable on disk; no cache file
is needed.

### Relationship to the existing Read/Edit check (updated)

| Dimension | Read/Edit before | Read/Edit after | Bash |
|---|---|---|---|
| Locator exists | `FABRICATED` | `FABRICATED` | — |
| In range | `BAD_LINE` | `BAD_LINE` | — |
| Happened this session | `UNREAD_FILE`/`UNREAD_LINE` | same | backticked output in recorded outputs |
| **Cited content accurate** | **not checked** | **checked when backticked (opt-in)** → `CONTENT_MISMATCH` | backticked output in recorded output → `BASH_OUTPUT_MISMATCH` |

The existence guarantee is the same idea across both; the content check is the
new, shared capability.

## The mechanism — verbatim-quote grounding (one rule, two sources)

The verifier has **one behavior**: for a footnote, extract every backticked span
(`` `…` ``) from its content portion and check each is an **exact substring** of
the relevant source.

- **Bash** (`Bash(<cmd>) — <output>`): match each backticked span against the
  **union of all recorded Bash `tool_result` outputs** this session. (The
  command is an unchecked label.)
- **File** (`Read(path:line) — `…``): match each backticked span against the
  **current file at the cited line/range**. Runs only after the pointer checks
  pass. Indentation is handled for free (substring match: `` `def foo():` `` is a
  substring of `    def foo():`).

Decision (identical shape for both):
- **No backticked spans** → nothing claimed verbatim → **skip**; citation keeps
  its current tier (Bash → *asserted*; file → *pointer-verified*). Paraphrase is
  never checked, so it never false-positives.
- **All backticked spans present** → Bash becomes **output-verified**; file stays
  **pointer-verified** (the content check adds confidence, not a new tier).
- **Any backticked span absent** → a warn-only finding:
  - Bash → **`BASH_OUTPUT_MISMATCH`**
  - File → **`CONTENT_MISMATCH`**

No token classification, no numbers heuristic, no normalization beyond exact
substring.

### Bonus for files: catches stale line numbers

`BAD_LINE` only catches a line that is now *out of range*. The content check also
catches a line number that **drifted** — `Read(app.py:42) — `def foo():`` where
line 42 still exists but the code moved — which `BAD_LINE` cannot. Warn-only, so
active-editing churn never blocks.

## File content check: opt-in + light policy mention (cost control)

The file content check is **purely opt-in**:
- The **verifier** checks backticked file content *if present* — this costs
  nothing until used.
- The **policy** gets only a *light* mention: "you *may* backtick the exact line
  content for a stronger check — only if that content is already in your context;
  do **not** re-read a file just to quote it." No mandate.

Rationale (token cost, measured): the injected policy is ~1,516 tokens/turn
today. The backtick convention is already introduced for Bash, so extending it
to files is ~one sentence (~20 tokens, cached). Per-citation output cost is a
small snippet, and only when used. The real cost to avoid is **induced re-reads**
(re-opening a file just to quote it) — hence the explicit "only if already in
context" guidance.

## `grounding_spec.py` changes (single source of truth)
- Add a backticked-span extraction pattern (e.g. `BACKTICK_SPAN`), shared by both
  sources. Mark Bash as "output-checked"; keep `FILE_CITE`/`CHECKED`
  (file-pointer specific) as-is — content checking is a separate path layered on
  top of the pointer path.
- Update emitted policy text: the verbatim-quote convention for Bash (encouraged)
  and the light, opt-in mention for file content (with the no-re-read caveat).
- Extend `--check` assertions to cover the new pattern.

## `grounding-verifier.py` changes
- `collect()`: also gather `bash_outputs` (list of Bash `tool_result` contents,
  plus `is_error` reserved for Layer 2). Keep the existing `reads` map.
- `verify()`:
  - Bash atom → extract backticked spans, match against `bash_outputs` →
    `output-verified` / `BASH_OUTPUT_MISMATCH` / `asserted`.
  - File atom → after the existing pointer checks, if backticked spans are
    present, read the cited line/range from the resolved file and match → keep
    `pointer-verified` or add `CONTENT_MISMATCH`.
- `report()`/`summary_line()`: include `output-verified` in the verified set and
  counts; surface `CONTENT_MISMATCH`/`BASH_OUTPUT_MISMATCH` in "Grounding check:".

## Examples

File:

| Footnote | Current line 42 | Verdict |
|---|---|---|
| `Read(app.py:42)` | (any) | pointer-verified — no backtick, content not checked (today's behavior) |
| `Read(app.py:42) — `def foo():`` | `    def foo():` | pointer-verified — span is a substring (indentation tolerated) |
| `Read(app.py:42) — `def foo():`` | `    def bar():` | `CONTENT_MISMATCH` — span absent at line 42 (drift/misquote caught) |

Bash:

| Footnote (output part) | Recorded output | Verdict |
|---|---|---|
| `` `Ran 5 tests`, `OK` `` | `Ran 5 tests in 0.523s … OK` | output-verified |
| `ran the suite, `Ran 5 tests in 0.5s`` | `Ran 5 tests in 0.523s` | `BASH_OUTPUT_MISMATCH` — `0.5s` ⊄ `0.523s` |
| `all tests pass` (no backticks) | `12 passed` | asserted — nothing claimed verbatim |
| `Bash(git status) — `pushed: 0`` | `pushed: 0` exists | output-verified — command label wrong but output real (accepted tradeoff) |

## Layer 2 — semantic intent check (documented, NOT built now)

Whether the model's *claim* follows from the content (e.g. "all tests pass" vs
output `12 passed, 3 failed`) is a semantic judgment needing a model — the
existing **OPTIONAL ESCALATION** in `grounding-verifier.py` (a second model call).
Stays opt-in and out of the deterministic core; not implemented here.

## Testing (TDD)
- `collect()` gathers Bash outputs from a synthetic transcript.
- Backticked-span extraction from a footnote's content portion.
- Bash: output-verified when all spans present; `BASH_OUTPUT_MISMATCH` when a span
  is absent; asserted when no spans.
- File: pointer-verified + content match passes; `CONTENT_MISMATCH` when a
  backticked span is not at the cited line; pointer-only (no backtick) unchanged;
  indentation tolerated; stale/drifted line caught.
- Exact-substring semantics (`0.5s` fails against `0.523s`).
- Report/summary include `output-verified`; mismatches under "Grounding check:";
  all warn-only by default.
- `grounding_spec.py --check` still passes.

## Risks / edge cases
- **Induced re-reads (files)**: model re-opens a file just to quote it → extra
  tokens/latency. Mitigation: policy says quote only what's already in context.
- **Transcript truncation (Bash)**: very large outputs may be truncated → false
  `BASH_OUTPUT_MISMATCH`. Mitigation: warn-only; revisit a `PostToolUse:Bash`
  cache only if it bites.
- **Staleness during active edits (files)**: a line edited after being read may
  fail the content check. Partly the feature (drift detection); warn-only.
- **Coincidental substring**: an invented span that appears elsewhere passes.
  Accepted — this is a grounding check, not a semantic one.
- **Separator ambiguity (Bash)**: a command containing `—`/`-`; parse the command
  greedily to the last `)` before the separator.
- **Locator-not-verified (Bash)**: command label unchecked — documented tradeoff.

## Rollout
- All new findings warn-only by default (`CONTENT_MISMATCH`,
  `BASH_OUTPUT_MISMATCH` not in `BLOCK_CODES`).
- File content check is opt-in with only a light policy mention.
- README "Scope" note updated: backticked Bash output and backticked file-line
  content are now grounded (the quoted span really appears in the source), still
  not semantically judged.
