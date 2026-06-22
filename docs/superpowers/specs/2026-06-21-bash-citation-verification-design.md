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
| Happened this session | `UNREAD_FILE`/`UNREAD_LINE` (opened) | backticked output present in recorded outputs |
| Cited content accurate | **not checked** (pointer only) | backticked output present in recorded output |

So Layer 1's existence guarantee **mirrors** Read's `UNREAD_FILE` ("did this
actually happen this session?"), and its output-match is a **new** capability
with no Read equivalent — in the content dimension it is *stronger* than the Read
check, because it validates the quoted evidence itself.

## Goals / non-goals

Goals:
- Catch **fabricated commands/results** — a `Bash(...)` footnote whose quoted
  output was never produced this session.
- Catch **invented specifics** — a quoted value that does not appear in the
  recorded output.
- **Do not** false-positive on paraphrasing.
- Deterministic, no re-execution, standard library only.

Non-goals (for this build):
- Verifying the command string itself (accepted tradeoff: a mislabeled command
  with a real output passes — inventing *results* is the dangerous lie;
  mislabeling a command is cosmetic).
- Judging whether the model's *claim* follows from the output — that's the
  semantic Layer 2, kept opt-in (see below).
- Re-running commands. Never.

## Layer 1 — verbatim-quote grounding (build now)

We use an explicit **verbatim-quote (backtick) convention** rather than a
token-classification heuristic. The author marks the spans they claim are
verbatim by wrapping them in backticks; the verifier checks exactly those.
This moves the decision of "what is a checkable claim" from a fragile guess the
verifier makes to an explicit signal the author gives.

### The check (verifier — one behavior)
For each `Bash(<cmd>) — <output text>` footnote:
1. Parse off `<cmd>` (ignored as a label) and `<output text>`.
2. Extract every **backticked span** from `<output text>` (`` `…` ``).
3. Decide:
   - **No backticked spans** → nothing claimed verbatim → **skip**; citation
     stays **asserted** (unchecked, as today). This is what makes paraphrase
     safe — un-backticked prose is never checked.
   - **Every backticked span** is an exact substring of the recorded Bash
     outputs → **output-verified**.
   - **Any backticked span** is absent from the recorded outputs → finding
     **`BASH_OUTPUT_MISMATCH`** (warn-only).

Matching is an exact substring test against the **union** of all recorded Bash
`tool_result` outputs this session (we do not tie a footnote to one specific
command — consistent with "command is just a label"). No token classification,
no numbers heuristic, no normalization.

### Authoring guidance (injected policy — "robust" style)
The verifier treats every backticked span identically; how much to backtick is
the author's choice. The policy text will recommend the **robust** discipline:
backtick the **stable, meaningful** spans of output (e.g. `` `Ran 5 tests` ``,
`` `OK` ``, `` `12 passed` ``) and leave **volatile** values (timings, memory
addresses, PIDs, run-specific ids) as ordinary prose. Exact-quoting a whole line
is allowed and still verifies, but backticking a volatile value is needlessly
brittle (it changes every run). Un-backticked text stays *asserted* — honest, no
penalty, no credit.

### Tiers and findings
- New tier **`output-verified`** (symbol `✓`), counted with pointer-verified in
  the trust summary and listed in the report alongside pointer-verified (the
  report's "list only verified" rule generalizes from `pointer-verified` to the
  set `{pointer-verified, output-verified}`).
- New finding code **`BASH_OUTPUT_MISMATCH`** — warn-only by default (not in
  `BLOCK_CODES`); shown in the "Grounding check:" section like other findings.
  May be opted into blocking later, once trusted.
- A Bash footnote with no backticked spans remains **asserted** (`~`), as today.

## `grounding_spec.py` changes (single source of truth)

The tool taxonomy and the policy text both derive from `grounding_spec.py`, so
Bash's new status is encoded there:
- Add a backticked-span extraction pattern (e.g. `BACKTICK_SPAN`) and an
  "output-checked" marker on the Bash row, so the policy text can tell the model
  to put verbatim output in backticks (and that those are now checked) and the
  verifier imports the same pattern. Keep `FILE_CITE`/`CHECKED` (file-pointer
  specific) untouched — Bash uses a separate verification path, not the
  file-pointer path.
- Update the emitted policy text with the verbatim-quote convention and the
  robust authoring guidance.
- Extend `--check` self-consistency assertions to cover the new pattern.

## `grounding-verifier.py` changes
- `collect()`: in addition to the `reads` map, gather `bash_outputs` — the list
  (or concatenation) of Bash `tool_result` contents this session (and
  `is_error`, reserved for Layer 2). Return it alongside the existing values.
- `verify()`: when a footnote's leading atom is `Bash(...)`, extract its
  backticked spans and run the substring check, assigning `output-verified` /
  `BASH_OUTPUT_MISMATCH` / `asserted`. Non-Bash recorded-output atoms
  (Web/Task/MCP) remain asserted.
- `report()`/`summary_line()`: include `output-verified` in the verified set and
  in the summary counts.

## Examples

| Footnote (output part) | Recorded output | Verdict |
|---|---|---|
| `` `Ran 5 tests`, `OK` `` | `Ran 5 tests in 0.523s … OK` | output-verified — both spans are exact substrings |
| `` `Ran 5 tests in 0.523s` `` | `Ran 5 tests in 0.523s` | output-verified — full-line exact quote |
| `ran the suite, `Ran 5 tests in 0.5s`` | `Ran 5 tests in 0.523s` | `BASH_OUTPUT_MISMATCH` — `0.5s` ⊄ `0.523s` (rounding caught) |
| `all tests pass` (no backticks) | `12 passed` | asserted — nothing claimed verbatim → skipped (paraphrase safe) |
| `` `99 passed` `` | `12 passed` | `BASH_OUTPUT_MISMATCH` — span absent (invented) |
| `Bash(git status) — `pushed: 0`` | `pushed: 0` exists | output-verified — command label wrong but output real (accepted tradeoff) |

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
- Bash footnote parsing (`<cmd>` / `<output>` split; backticked-span extraction).
- `verify()`: output-verified when all backticked spans are present;
  `BASH_OUTPUT_MISMATCH` when a span is absent; asserted when there are no
  backticked spans.
- Exact-substring semantics (a rounded value like `0.5s` fails against `0.523s`).
- Report/summary include `output-verified`; mismatch appears under "Grounding
  check:"; warn-only (no block by default).
- `grounding_spec.py --check` still passes.

## Risks / edge cases
- **Transcript truncation**: if Claude Code truncates very large Bash outputs in
  the transcript, a real backticked span could be missing → false
  `BASH_OUTPUT_MISMATCH`. Mitigation: warn-only; revisit a dedicated
  `PostToolUse:Bash` cache only if this proves to bite in practice.
- **Author must backtick to earn verification**: an un-backticked Bash footnote
  stays *asserted* (no regression, but no credit). This is intended — the
  convention is opt-in honesty.
- **Coincidental substring**: an invented span that happens to appear elsewhere
  in some unrelated output passes. Accepted — Layer 1 is a grounding check, not a
  semantic one.
- **Separator ambiguity**: a command containing `—`/`-` could confuse the
  cmd/output split; parse the command greedily to the last `)` before the
  separator.
- **Command label unverified**: documented tradeoff above.

## Rollout
- Warn-only by default (`BASH_OUTPUT_MISMATCH` not in `BLOCK_CODES`), matching
  the project's "watch the warnings first" stance.
- README "Scope" note updated: backticked Bash output is now grounded (the quoted
  span really appears in the recorded output), still not semantically judged.
