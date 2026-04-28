# Walk-through: building a PDF→Markdown CLI

A complete example of running `codex-pair` on a real, end-to-end task. Shows what each of the 8 steps actually looks like in practice.

The user wants a CLI tool that takes a PDF and outputs Markdown. They invoke `codex-pair this`. Here's what happens.

---

## Step 1 — Architecture sketch (Claude → user)

> **You:** Make me a CLI tool: take a PDF path, output Markdown to stdout. Use `codex pair this` on it.

> **Claude:**
> Two reasonable architectures:
>
> **A) PyMuPDF + custom Markdown writer**
>   - Lightweight, ~1 dep, fast on text-mostly PDFs
>   - Tables come out OK but not great
>   - You own the Markdown layout decisions
>
> **B) Marker (`pip install marker-pdf`)**
>   - Heavier (~500MB model download), needs PyTorch
>   - Best-in-class on tables, formulas, complex layouts
>   - Less control — opinionated Markdown
>
> Lean: **A** unless you have heavy table/math content. Which one?

> **You:** A.

(User involvement at step 1: ~30 seconds.)

---

## Step 2 — Plan v1 with interface contract (Claude)

> **Claude:** Locked plan v1.
>
> **Interface contract:**
> ```
> parse_pdf(path: str) -> ParsedDoc
>   raises: FileNotFoundError, PDFParseError
>
> ParsedDoc: dataclass
>   - title: str | None
>   - pages: list[Page]
>
> Page: dataclass
>   - number: int  (1-indexed)
>   - blocks: list[Block]
>
> Block: union of TextBlock | HeadingBlock | ListBlock | TableBlock
>
> render_markdown(doc: ParsedDoc) -> str
>   returns: a UTF-8 Markdown string
> ```
>
> **Modules:**
> - `pdf_to_md/parser.py` — `parse_pdf` and the `ParsedDoc` family
> - `pdf_to_md/renderer.py` — `render_markdown`
> - `pdf_to_md/cli.py` — argparse glue, calls parser → renderer → stdout
>
> **Tests:**
> - Unit (Codex): each Block type round-trips through `render_markdown`
> - Integration (Codex): `parse_pdf` on a fixture PDF returns expected `ParsedDoc`
> - Behavior (Claude): run `pdf-to-md sample.pdf` on a real PDF, eyeball the Markdown
>
> **Implementation order:** parser → renderer → cli.

(Claude has not written any code yet. This is plan only.)

---

## Step 3 — Codex reviews (Codex)

> **Codex:**
> **Agreed:** module split, three-layer Block taxonomy, test plan structure.
>
> **Disagreed:**
>
> 1. `parse_pdf` should not raise raw `FileNotFoundError`. Wrap it in a domain error like `PDFNotFound(path)` so callers can distinguish "file missing" from a Python stdlib error from elsewhere in the call chain. Same for `PDFParseError` — make it a custom subclass with `path` and `cause`.
>
> 2. `Block` as a union type works in Python 3.11+ but typing it as `TextBlock | HeadingBlock | ...` everywhere is verbose. Suggest a `Block` ABC with a `kind: Literal["text" | "heading" | "list" | "table"]` field — easier to pattern-match in the renderer.
>
> 3. **Missing:** what about images? PDFs commonly have inline images. The plan doesn't mention them. Suggest: extract image references into the `ParsedDoc` (path or base64), let the renderer emit `![alt](path)`. Skip for v1 if user agrees, but call it out.
>
> **Missing tests:**
>
> - No test for empty PDF (0 pages)
> - No test for PDF with only images, no text
> - No test for malformed PDF (truncated, non-PDF magic bytes)

---

## Step 4 — Consensus loop (Claude ↔ Codex, round 1 of 3)

> **Claude:**
> 1. **Accept.** Custom `PDFNotFound` and `PDFParseError(path, cause)`.
> 2. **Accept.** ABC with `kind` literal — this will simplify the renderer's match statement.
> 3. **Accept the call-out, defer the work.** v1 skips images entirely (parser drops them). Image support is a follow-up. Updating contract: `parse_pdf` ignores image regions; `ParsedDoc` has no image field for now.
>
> Tests: accepting all three additions. Empty PDF returns `ParsedDoc(title=None, pages=[])`. Image-only PDF returns pages with empty blocks. Malformed PDF raises `PDFParseError`.
>
> Plan v2 locked. Anything else, Codex?

> **Codex:** No further objections. Plan v2 is consensus.

(Round 1 was enough. No tiebreak needed.)

---

## Step 5 — Codex implements (Codex, in sandbox)

Codex appends a heartbeat to `.claude/worklog.jsonl` every 3-5 minutes:

```jsonl
{"ts":"...", "agent":"codex", "phase":"start", "action":"impl parser.py"}
{"ts":"...", "agent":"codex", "phase":"end", "action":"parser.py done", "files":["pdf_to_md/parser.py"]}
{"ts":"...", "agent":"codex", "phase":"start", "action":"impl renderer.py"}
{"ts":"...", "agent":"codex", "phase":"end", "action":"renderer.py done", "files":["pdf_to_md/renderer.py"]}
{"ts":"...", "agent":"codex", "phase":"start", "action":"impl cli.py + tests"}
{"ts":"...", "agent":"codex", "phase":"end", "action":"all done, tests passing"}
```

Total wall time: ~12 minutes.

---

## Step 6 — Codex code-level tests (Codex)

Codex runs:
```
pytest tests/ -v
```

Output:
```
tests/test_parser.py::test_simple_text PASSED
tests/test_parser.py::test_empty_pdf PASSED
tests/test_parser.py::test_malformed_pdf_raises PASSED
tests/test_parser.py::test_image_only_pdf PASSED
tests/test_renderer.py::test_text_block PASSED
tests/test_renderer.py::test_heading_block PASSED
tests/test_renderer.py::test_list_block PASSED
tests/test_renderer.py::test_table_block PASSED
tests/test_cli.py::test_cli_outputs_markdown PASSED
9 passed in 0.84s
```

All green. Codex appends to `.claude/pending-commits.jsonl`:

```json
{
  "ts": "...",
  "agent": "codex",
  "files_to_stage": [
    "pdf_to_md/__init__.py",
    "pdf_to_md/parser.py",
    "pdf_to_md/renderer.py",
    "pdf_to_md/cli.py",
    "tests/test_parser.py",
    "tests/test_renderer.py",
    "tests/test_cli.py",
    "tests/fixtures/sample.pdf",
    "pyproject.toml"
  ],
  "commit_message": "Initial implementation: PDF to Markdown CLI\n\nparse_pdf parses to ParsedDoc with TextBlock / HeadingBlock /\nListBlock / TableBlock. render_markdown converts to UTF-8\nMarkdown. cli.py wraps both with argparse.\n\nTests cover empty, image-only, malformed, and all four\nBlock types. Image extraction deferred to v2.",
  "ready": true
}
```

---

## Step 7 — Claude verifies behavior (Claude)

> **Claude:** Pulling the diff. Reading parser.py — clean, matches the contract. renderer.py — match statement on `kind`, looks right. cli.py — argparse, no surprises.
>
> Running `pdf-to-md` on a real PDF (one of your saved Stripe receipts):
>
> ```
> $ pdf-to-md ~/Downloads/receipt.pdf
> # Stripe Receipt
>
> **Date:** 2026-04-15
> **Amount:** $20.00
> ...
> ```
>
> Looks correct. Headings preserved, amounts intact, no weird artifacts.
>
> Trying a multi-page academic paper next... output is good. Tables come through as Markdown tables (a bit ugly on cell padding but readable).
>
> Trying the malformed PDF (truncated): `PDFParseError: PDF file truncated at byte 12345 (path=/tmp/bad.pdf)`. Good error message.
>
> Verification passes.

(If verification had failed: Claude routes back to step 5 with a bug report, or step 4 if the issue is a plan gap.)

---

## Step 8 — Claude commits (Claude)

> **Claude:**
> Validating Codex's commit intent...
> - All 9 listed files exist in diff ✓
> - No extra files in diff ✓
> - Commit message subject ≤ 72 chars ✓
> - No `--force` or `--no-verify` flags ✓
>
> Executing commit...
>
> ```
> [main 7d3c8a2] Initial implementation: PDF to Markdown CLI
>  9 files changed, 287 insertions(+)
> ```
>
> Done. Try it: `pdf-to-md sample.pdf`

---

## Total time

- Your involvement: ~45 seconds (step 1 architecture pick).
- Pipeline wall time: ~17 minutes.
- Tokens: roughly 2× a solo Claude run, due to Codex's review + implementation + commit-intent steps.
- Bugs caught by Codex during step 3: 3 (custom error types, ABC for Block, missing image discussion).
- Bugs caught by Claude in step 7: 0 — table rendering is "a bit ugly" but functional, not a bug.

You read 2 messages and typed once. The pipeline did the rest.

---

## What this walk-through doesn't show

- A **deadlock** (3-round consensus exceeded) — uncommon. When it happens, you see the tiebreak card and pick A or B.
- A **failed verification** (step 7) — also uncommon. When it happens, the pipeline routes back to step 5 (code bug) or step 4 (plan gap), tries again, and only escalates to you after 3 same-failure rollbacks.
- A **destructive operation** during the pipeline — for example, if Codex's plan involved dropping a database column. The orchestrator's safety check (e.g. `risk-check`) fires regardless of pipeline mode and asks you to confirm.

For all of these, see [`SKILL.md`](../skills/codex-pair/SKILL.md) — section "Hard rules" and "User interaction timeline".
