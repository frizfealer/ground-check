# Bash citation verification — design

Date: 2026-06-21
Status: approved (brainstorming) — ready for implementation plan
Scope: `scripts/grounding_spec.py`, `scripts/grounding-verifier.py`, tests

## Problem

Today the verifier treats `Bash(<cmd>) — <output>` citations as **asserted
(unchecked)** — it takes the model's word for them. Asserted is honest (we can't
re-run Bash safely or deterministically), but it leaves the most common
recorded-output citation entirely unverified. We want to move Bash from
*asserted* toward *verified* without re-executing anything.

## Key realization (the framing this design rests on)

A Bash citation has two parts: the **command** (a human-readable label) and the
**output** (the evidence). The command string is the weakest thing to match —
footnotes cite a fragment of a compound command, drop flags, reformat pipes — so
matching it is brittle and false-positive-prone.

**The evidence is the output, not the command.** So we key the check on the
output and treat the command as an unchecked label. The transcript already holds
every Bash command, its full output, and an `is_error` flag, and the verifier
already reads the transcript — so no separate cache file is needed.

### Relationship to the existing Read/Edit check

A `Read(path:line)` citation is a pure **pointer**; the verifier checks the
pointer (file exists → `FABRICATED`, line in range → `BAD_LINE`, file/line
actually opened this session → `UNREAD_FILE`/`UNREAD_LINE`) and never compares
the cited prose to the file's content. Content correctness is deliberately left
to the optional semantic layer.

Bash is different because its output is **ephemeral** — it can't be re-opened
from disk later, so the citation embeds the output and the only way to ground it
is to match that embedded text against the single recorded copy in the
transcript.

| Dimension | Read/Edit (today) | Bash Layer 1 (this design) |
|---|---|---|
| The thing exists | `FABRICATED` (file on disk) | — (no file/line) |
| In range | `BAD_LINE` | — |
| Happened this session | `UNREAD_FILE`/`UNREAD_LINE` (opened) | output present in recorded outputs |
| Cited content accurate | **not checked** (pointer only) | quoted output present in recorded output |

So Layer 1's existence guarantee **mirrors** Read's `UNREAD_FILE` ("did this
actually happen this session?"), and its output-match is a **new** capability
with no Read equivalent — in the content dimension it is *stronger* than the Read
check, because it validates the quoted evidence itself.

## Goals / non-goals

Goals:
- Catch **fabricated commands/results** — a `Bash(...)` footnote whose quoted
  output was never produced this session.
- Catch **invented specifics** — a real-looking output with a wrong number/code.
- **Do not** false-positive on paraphrasing.
- Deterministic, no re-execution, standard library only.

Non-goals (for this build):
- Verifying the command string itself (accepted tradeoff: a mislabeled command
  with a real output passes — inventing *results* is the dangerous lie;
  mislabeling a command is cosmetic).
- Judging whether the model's *claim* follows from the output — that's the
  semantic Layer 2, kept opt-in (see below).
- Re-running commands. Never.

## Layer 1 — deterministic output-grounding (build now)

### Inputs
- A Bash footnote: `Bash(<cmd>) — <output text>` (the `—` / `-` separator
  already used by the policy; `<output text>` is everything after it).
- The recorded Bash outputs: the union of all Bash `tool_result` contents in the
  transcript this session.

### The check
1. Parse the footnote into `<cmd>` (ignored as a label) and `<output text>`.
2. Extract **distinctive tokens** from `<output text>` — tokens that carry
   specific information rather than prose:
   - contain a digit (`12`, `0`, `0.5s`, `200`), or
   - are ALL-CAPS, length ≥ 2 (`OK`, `FAILED`, `ERROR`, `HTTP`), or
   - contain `/` or `:` adjoining alphanumerics (`req/s`, `app.py:42`, URLs).
   Ordinary lowercase prose words (`all`, `tests`, `pass`) are **not**
   distinctive.
3. Decide:
   - **No distinctive tokens** → the quote is pure paraphrase → **skip**; the
     citation stays **asserted** (unchecked, as today). This is what prevents
     paraphrase false-positives.
   - **All distinctive tokens present** (word-boundary match) in the recorded
     outputs → **output-verified**.
   - **Any distinctive token missing** → finding **`BASH_OUTPUT_MISMATCH`**
     (warn-only).

### Tiers and findings
- New tier **`output-verified`** (symbol `✓`), counted with pointer-verified in
  the trust summary and listed in the report alongside pointer-verified (the
  report's "list only verified" rule generalizes from `pointer-verified` to the
  set `{pointer-verified, output-verified}`).
- New finding code **`BASH_OUTPUT_MISMATCH`** — warn-only by default (not in
  `BLOCK_CODES`); shown in the "Grounding check:" section like other findings.
  It may be opted into blocking later, once trusted.
- A paraphrase-only Bash footnote remains **asserted** (`~`), exactly as today.

### Matching rules
- Match against the **union** of all recorded Bash outputs (we do not tie a
  footnote to one specific command — consistent with "command is just a label").
- Distinctive-token match is **word-boundary** (so cited `12` does not match
  `120`), case-sensitive for ALL-CAPS tokens.
- Require **all** distinctive tokens to be found; any miss → mismatch.

## `grounding_spec.py` changes (single source of truth)

The tool taxonomy and the policy text both derive from `grounding_spec.py`, so
Bash's new status is encoded there:
- Add a Bash citation pattern (e.g. `BASH_CITE`) and an "output-checked" marker
  on the Bash row, so the policy text can tell the model its Bash citations are
  now checked (encouraging accurate output quoting) and the verifier imports the
  same pattern. Keep `FILE_CITE`/`CHECKED` (file-pointer specific) untouched —
  Bash uses a separate verification path, not the file-pointer path.
- Extend `--check` self-consistency assertions to cover the new pattern.

## `grounding-verifier.py` changes
- `collect()`: in addition to the `reads` map, gather `bash_outputs` — the
  concatenation/list of Bash `tool_result` contents (and `is_error`, reserved for
  Layer 2). Return it alongside the existing values.
- `verify()`: when a footnote's leading atom is `Bash(...)`, run the Layer 1
  check and assign `output-verified` / `BASH_OUTPUT_MISMATCH` / `asserted`
  accordingly. Non-Bash recorded-output atoms (Web/Task/MCP) remain asserted.
- `report()`/`summary_line()`: include `output-verified` in the verified set and
  in the summary counts.

## Examples

Layer 1 (output-grounding):

| Footnote (output part) | Recorded output | Verdict |
|---|---|---|
| `Ran 5 tests … OK` | `Ran 5 tests in 0.5s … OK` | output-verified (`5`,`OK` present) |
| `pushed: 0` | `pushed: 0` | output-verified (`0` present) |
| `all tests pass` | `12 passed` | asserted — no distinctive token → skipped |
| `99 passed` | `12 passed` | `BASH_OUTPUT_MISMATCH` (`99` absent) |
| `1200 req/s` | (no such output) | `BASH_OUTPUT_MISMATCH` (`1200`,`req/s` absent) |
| `Bash(git status) — pushed: 0` | `pushed: 0` exists | output-verified — command label wrong but output real (the accepted tradeoff) |

## Layer 2 — semantic intent check (documented, NOT built now)

Deciding whether the model's *claim* follows from command+output (e.g. claim
"all tests pass" vs output "12 passed, 3 failed") is a semantic judgment that
needs a model. This is exactly the existing **OPTIONAL ESCALATION** noted at the
bottom of `grounding-verifier.py` (a second model call asking "does \<claim\>
follow from \<output\>"). It stays **opt-in and out of the deterministic core**.
This design only records it as the planned future layer; it is not implemented
here.

## Testing (TDD)
- `collect()` gathers Bash outputs from a synthetic transcript.
- Bash footnote parsing (`<cmd>` / `<output>` split).
- Distinctive-token extraction (numbers/ALL-CAPS/`/`:`; prose excluded).
- `verify()`: output-verified on present tokens; `BASH_OUTPUT_MISMATCH` on a
  missing/invented token; asserted on pure paraphrase.
- Word-boundary matching (`12` ≠ `120`).
- Report/summary include `output-verified`; mismatch appears under "Grounding
  check:"; warn-only (no block by default).
- `grounding_spec.py --check` still passes.

## Risks / edge cases
- **Transcript truncation**: if Claude Code truncates very large Bash outputs in
  the transcript, a real token could be missing → false `BASH_OUTPUT_MISMATCH`.
  Mitigation: warn-only; revisit a dedicated `PostToolUse:Bash` cache only if
  this proves to bite in practice.
- **Coincidental token presence**: an invented number that happens to appear in
  some unrelated output passes. Accepted — Layer 1 is a grounding check, not a
  semantic one.
- **Separator ambiguity**: a command containing `—`/`-` could confuse the
  cmd/output split; parse the command greedily to the last `)` before the
  separator.
- **Command label unverified**: documented tradeoff above.

## Rollout
- Warn-only by default (`BASH_OUTPUT_MISMATCH` not in `BLOCK_CODES`), matching
  the project's "watch the warnings first" stance.
- README "Scope" note updated: Bash output is now grounded (existence of the
  quoted output), still not semantically judged.
