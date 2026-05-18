# From Corpus to Gazetteer: A Process Document

## How Mnemotron Wiki for Research and Claude Code Produced the Hamilton Gazetteer

**Author:** Patrick R. Wallace, Hamilton College LITS  
**Date:** 2026-05-18  
**Project:** `research-wiki-test` (Hamilton Gazetteer)  
**Tooling:** `mnemotron-wiki-r` + Claude Code (Claude Sonnet 4.6)

Copyright (C) 2026 Patrick R. Wallace. Permission is granted to copy, distribute and/or modify this document under the terms of the GNU Free Documentation License, Version 1.3 or any later version published by the Free Software Foundation; with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts. Full license text: <https://www.gnu.org/licenses/fdl-1.3.html>

*AI assistance notice: This document was drafted with assistance from Claude Code (Anthropic, claude-sonnet-4-6). The author reviewed, edited, and takes responsibility for the content.*

---

## Contents

1. [What Was Built — A Summary for Any Reader](#1-what-was-built)
2. [The Tool: What Mnemotron Wiki for Research Is](#2-the-tool)
3. [The Pipeline in Plain Language](#3-the-pipeline-in-plain-language)
4. [The Pipeline in Technical Detail](#4-the-pipeline-in-technical-detail)
5. [What Actually Happened: The Hamilton Gazetteer Project Step by Step](#5-what-actually-happened)
6. [Claude Code's Role Beyond the Scripts](#6-claude-codes-role-beyond-the-scripts)
7. [The Output: What the Wiki Contains](#7-the-output)
8. [Alternative Approaches and Their Tradeoffs](#8-alternative-approaches)
9. [How the Tool and Workflow Could Be Improved](#9-improvements)
10. [Reproducing This for a Similar Project: A Practical Guide](#10-reproducing-this)

---

## 1. What Was Built

The Hamilton Gazetteer is a structured knowledge base — a searchable, cross-referenced encyclopedia of Hamilton College's institutional history — produced by processing approximately 3,500 primary source documents into a collection of 3,510 source transcriptions, 72 synthesized topic essays, and 410 entity profiles. The primary sources span two centuries (1793–2025) and include:

- 1,113 issues of *Hamilton Life* (1899–1942), the college's original student newspaper
- 78 issues of *Hamiltonews* (1942–1947)
- Hundreds of issues of *The Spectator* (1947–2025)
- 205 course catalogs (1814–2025)
- The 1922 *Documentary History of Hamilton College*
- Wikipedia exports for notable alumni and affiliates
- YHM (Your Hamilton Museum) archival catalog entries and newspaper issues

The Gazetteer documents institutional milestones, student life, named individuals, buildings, controversies, and events in a form that is navigable without specialized archival knowledge. It was produced in approximately two weeks of intermittent work using a combination of automated scripts and AI-mediated synthesis, without requiring manual reading of every source.

This document explains how.

---

## 2. The Tool

### For any reader

**Mnemotron Wiki for Research** (`mnemotron-wiki-r`) is a set of Python scripts and structured instructions designed to turn a folder of documents into an organized research wiki. Think of it as a librarian's assistant that can:

- Read a document and write a clean summary of its contents
- File that summary in a sensible location
- Connect it to other documents on the same subject
- Keep a log of everything it has processed
- Never accidentally process the same document twice

The "librarian" doing the actual reading and writing is Claude Code — Anthropic's AI system operating through the command line. The scripts handle the mechanical parts (finding files, extracting text, keeping records); Claude handles the interpretive parts (understanding what a document is about, what topics it touches, and how to integrate its findings into the broader knowledge base).

The end result is a wiki: a collection of interlinked markdown files that can be browsed on disk or exported to a self-contained website (via `wiki_export.py`).

### For librarians and archivists familiar with AI workflows

Mnemotron Wiki for Research is a **human-in-the-loop corpus processing system** built around three distinct layers:

1. **Mechanical extraction layer** (`extract_text.py`, `ocr.py`): deterministic Python code that converts heterogeneous document formats to plain text. This layer uses pdfminer for native PDFs, BeautifulSoup for HTML, and a tiered OCR pipeline (Tesseract-first with Claude Vision fallback) for scanned images.

2. **Ingestion tracking layer** (`manifest.py`, `check_ingest.py`): a content-hash-based idempotency system. Files are identified by MD5 digest, not filename, so renames and moves do not cause reprocessing. A separate log (`processed.json`) tracks Internet Archive items by identifier. Both logs are committed to git for reproducibility across machines.

3. **AI synthesis layer** (`RESEARCH_WIKI_TASK.md` + Claude Code): structured natural language instructions that tell Claude Code how to interpret extracted text, create structured wiki pages, update thematic topic essays, and maintain referential integrity between source pages, topic pages, and entity pages.

The architectural insight is that layers 1 and 2 are fast, cheap, and deterministic; layer 3 (the expensive AI layer) is invoked selectively and purposively. The system is designed to minimize unnecessary Claude API calls: for Internet Archive items, IA's own pre-built Tesseract OCR (`*_djvu.txt`) is used when it passes quality checks, avoiding a local OCR pass entirely; for printed documents, Tesseract runs locally (free) and Claude Vision is a fallback only for pages Tesseract cannot read.

---

## 3. The Pipeline in Plain Language

When you add a document to the `ingest/` folder and run the ingest task, the following happens:

1. **The system checks what is new.** It looks at all files in `ingest/`, calculates a fingerprint (hash) of each one, and compares against its log. Files already processed are silently skipped.

2. **Text is extracted.** For a PDF with embedded text, the text is pulled out directly. For a scanned image or a PDF that is just a photograph of pages, an OCR engine reads the image and produces text. For a webpage, the visible text is extracted and navigation/advertising content is stripped.

3. **A source page is written.** The extracted text is saved as a structured markdown file in `wiki/sources/`. This page records what the document is, where it came from, how its text was obtained, and its full content. It is a faithful transcription — not a summary, not an interpretation.

4. **The topic network is updated.** Claude reads the source page and identifies which existing topics it relates to. It adds the source to those topic pages and may update the topic's synthesis — its "Key Points" section — with new findings. If the source introduces a topic that does not yet have a page, Claude creates one.

5. **Entity pages are maintained.** Significant people, organizations, and places are given their own pages. When a source mentions Elihu Root or the Kennedy Arts Center, Claude checks whether those entities already have pages, updates them with new information, and links them back to the source.

6. **The index is regenerated.** A master index listing all sources, topics, and entities is rewritten to reflect the new content.

7. **Changes are committed to git.** Everything is version-controlled so that the state of the wiki at any point in time can be recovered.

---

## 4. The Pipeline in Technical Detail

This section documents the code that implements the pipeline. It is oriented toward people who may want to extend, adapt, or troubleshoot the tooling.

### 4.1 File discovery and idempotency: `check_ingest.py` and `manifest.py`

`check_ingest.py` scans `ingest/` recursively for files with supported extensions (`.pdf`, `.txt`, `.md`, `.html`, `.htm`, `.csv`, `.docx`, `.odt`, `.tif`, `.tiff`, `.jpg`, `.jpeg`, `.png`). It excludes:

- Files in `ingest/failed/` (quarantined files that have previously failed processing)
- Hidden files (dotfiles, `.DS_Store`)
- Files whose MD5 content hash already appears in `.manifest.json`

The manifest (`.manifest.json`) records each processed file as a JSON object keyed by MD5 hex digest:

```json
{
  "<md5-hex>": {
    "filename":  "original-file.pdf",
    "path":      "/absolute/path/at/time/of/processing",
    "processed": "2026-05-14T18:23:11+00:00",
    "wiki_page": "wiki/sources/slug.md"
  }
}
```

Content-hash identification has two important properties: a file can be renamed or moved within `ingest/` without triggering reprocessing; but modifying a file's contents — even trivially — causes it to be treated as a new file. The manifest is committed to git, so ingest history persists across machines and clones.

IA items (fetched by `ia_ingest.py`) are tracked separately in `ingest/ia-sources/processed.json`, keyed by IA identifier, because IA items have no local file to hash.

### 4.2 Text extraction: `extract_text.py`

`extract_text.py` dispatches to format-specific backends:

| Format | Backend | Notes |
|--------|---------|-------|
| `.pdf` | pdfminer.six | Scanned PDFs (< 100 non-whitespace chars extracted) are flagged `is_scan=True` for re-routing to `ocr.py` |
| `.html`, `.htm` | BeautifulSoup + lxml | `<script>`, `<style>`, `<head>` stripped; visible text extracted |
| `.txt`, `.md` | Direct read | UTF-8 with error replacement |
| `.csv` | stdlib csv | Rendered as pipe-separated text; row/column counts recorded as metadata |
| `.docx`, `.odt` | python-docx | Paragraph text; title and author captured from core properties |

The function never raises. Errors are returned in `result["error"]`; callers decide what to do with failures (log and continue, quarantine to `ingest/failed/`, etc.).

### 4.3 OCR pipeline: `ocr.py`

The OCR pipeline is designed to minimize API calls to Claude Vision while still producing high-quality transcriptions of difficult material.

**Decision logic:**

```
Input image or scanned PDF
    │
    ├─ hint == "handwritten" ──► Claude Vision for all pages
    │
    └─ hint == "print" or "auto"
           │
           ▼
    [Pre-flight] Tesseract on 25%-scale thumbnail of page 1
           │
    Passes quality?
           ├── NO ──► Skip Tesseract; Claude Vision for all pages
           └── YES
                  │
                  ▼
           For each page at full resolution:
           │
           ├─ Tesseract → passes quality check? ──YES──► keep result
           │
           └─ quality poor ─────────────────────────► Claude Vision (this page only)
```

**Quality checks for Tesseract output** (all three must pass):

| Check | Default threshold | What failure indicates |
|-------|------------------|----------------------|
| Word count | ≥ 15 words | Blank output; Tesseract found no text |
| Alpha ratio | ≥ 45% non-whitespace chars are alphabetic | Symbol noise, garbling, non-text image |
| Mean word length | ≥ 2.0 chars/word | Single-character noise stream (common failure mode on degraded scans) |

The pre-flight thumbnail check (run in ~200 ms on a 25%-scale image) avoids wasting time running a full-resolution Tesseract pass on documents it clearly cannot read. This matters especially for large PDFs: without the pre-flight, a 200-page document where Tesseract consistently fails would generate 200 failed Tesseract calls before any Claude Vision calls are made.

Per-page fallback avoids the opposite waste: if only 3 pages of a 200-page document are difficult for Tesseract, only those 3 pages are sent to Claude. Without per-page fallback, the whole document would be sent to Claude Vision.

The reported `ocr_method` in source page frontmatter records which engine(s) were used: `"tesseract"`, `"claude"`, or `"tesseract+claude"`.

### 4.4 Batch ingest: `batch_ingest.py`

`batch_ingest.py` automates source page creation for large batches of pre-processed text files. It:

1. Calls `check_ingest.py` to get unprocessed files
2. Calls `extract_text.py` (or `ocr.py` for images) on each
3. Generates a slug (URL-safe, collision-avoiding) from the filename
4. Dispatches to a page generator based on file type and naming convention
5. Writes `wiki/sources/<slug>.md`
6. Updates `.manifest.json` after every file (crash-safe incremental progress)

**Corpus-specific page generators** can be registered in `_build_page()`. The script ships with a generic `_document_page` fallback (derives a title from the filename, writes full text as content) and a `_csv_page` handler (records column names and row counts, includes a preview of the first 25 rows). The scaffolding and instructions for adding custom generators are in the file, but no corpus-specific generators have been added for the Hamilton Gazetteer — this is a known improvement area. See Section 9.1.

### 4.5 Internet Archive ingest: `ia_ingest.py`

`ia_ingest.py` fetches and ingests items directly from Internet Archive by identifier. For each identifier:

1. Fetch IA metadata via `ia metadata <identifier>` and verify `mediatype == "texts"`
2. Attempt the djvu.txt path: download `*_djvu.txt` (IA's pre-built Tesseract OCR, typically 100–400 KB); evaluate quality (word count ≥ 100, alpha ratio ≥ 0.40); write source page if it passes
3. If djvu.txt is absent or fails quality: download the original image PDF (50–100 MB); try pdfminer text-layer extraction first; if that fails, run the local Tesseract/Claude OCR pipeline
4. Write source page to `wiki/sources/`; record identifier in `ingest/ia-sources/processed.json`

The djvu.txt preference is a major performance optimization for IA collections: IA's Tesseract OCR is already computed and stored, downloading it takes seconds rather than minutes, and quality is generally acceptable for well-scanned text. The fallback to the local pipeline handles items where IA's OCR is absent or degraded.

### 4.6 The synthesis task: `RESEARCH_WIKI_TASK.md`

`RESEARCH_WIKI_TASK.md` is Claude Code's operating instructions for the ingest task. It is not a script — it is a natural language specification that Claude Code reads and follows, the same way a researcher might follow a standard operating procedure. It defines:

- **Four pipeline stages** (corpus assessment, document ingest, index update, git commit) with a fifth optional stage for deep synthesis passes
- **Detailed templates** for source pages, topic pages, and entity pages
- **OCR repair rules** specifying exactly which character substitutions to correct, which ambiguities to flag, and what not to touch
- **Synthesis depth guidelines** calibrated to batch size (1–5 files: full per-file synthesis; 50+ files: pre-establish taxonomy, sample-based synthesis)
- **Concurrency safeguards** for parallel agent runs (always use `Edit` not `Write` on existing pages; re-read before editing; never proceed with stale content)
- **The additive editing rule**: topic and entity pages are accumulated over time; `Write` destroys prior content and is forbidden on existing pages except to rewrite empty stubs

The task document is licensed CC0 (public domain) to enable free reuse and adaptation.

### 4.7 Keyword linking: `synthesize_links.py`

`synthesize_links.py` is a mechanical first-pass synthesis tool. It reads each source page's `## Content` section, matches it against a topic map, and appends a `## Related Topics` section to pages that match. This is the cheapest way to establish cross-references for a large batch: it runs in seconds, requires no API calls, and handles the bookkeeping so that the manual synthesis pass can focus on interpretation.

**The topic map is auto-generated at runtime** by reading all `wiki/topics/*.md` files and extracting keywords from each topic's H1 title, slug, and Overview section. This keeps the map in sync with the wiki without manual maintenance — adding a new topic page is sufficient; no edits to the script are needed. Compound phrases extracted from topic Overview text are preferred over single words to reduce false positives.

**Key flags:**

| Flag | Effect |
|------|--------|
| *(none)* | Adds Related Topics to source pages that lack the section entirely |
| `--rebuild` | Strips and regenerates Related Topics on every source page against the current map |
| `--dry-run` | Shows what would change without writing |
| `--show-map` | Prints the auto-generated topic map and exits |

Run `--rebuild` after adding new topic pages to backfill cross-references across all existing source pages. For the Hamilton Gazetteer, a rebuild pass touches ~3,500 pages in about 30 seconds.

### 4.8 Open questions aggregation: `scripts/open_questions.py`

`open_questions.py` scans the `## Open Questions` section of every topic page, deduplicates, and writes a consolidated `wiki/OPEN-QUESTIONS.md` grouped by topic with links back to each topic page. It is run as part of Stage 3 (index update) after any synthesis pass. The output — currently 444 questions across 69 topics — is the primary research roadmap for identifying what the corpus does not yet answer.

### 4.9 OCR quality reporting: `scripts/quality_report.py`

`quality_report.py` reads all source page frontmatter and `## Notes` sections to produce aggregate OCR statistics: source counts by `ocr_method`, by document type, by publication title, and a list of pages flagged "Manual review recommended: yes." This is the primary tool for identifying which portions of the corpus need a closer look before a publication-quality export is produced.

### 4.10 Static export: `wiki_export.py`

`wiki_export.py` converts the markdown wiki to a self-contained static HTML site (the Hamilton Gazetteer). It uses the Python `markdown` library with extensions for tables, fenced code, table of contents, and typographic improvements. Each page gets a standard header (with navigation back to the index), a disclaimer attributing AI-assisted generation, and a footer. CSS is embedded in a single file. The export excludes source pages by default (there are ~3,500 of them); pass `--all` to include them.

The export was renamed from "Spectator Research Wiki" to "Hamilton Gazetteer" during development as the scope expanded beyond the Spectator archive to cover the full institutional record.

---

## 5. What Actually Happened: The Hamilton Gazetteer Project Step by Step

This section documents the actual sequence of work on the Hamilton Gazetteer project. It is more specific and more honest about the iterative, non-linear nature of the process than a description of the tool alone would be.

### 5.1 Starting point

The project began as a test of `mnemotron-wiki-r` on the Hamilton College Spectator digital archive. The initial corpus was a set of Spectator issues from the 1947–2025 period, obtained as text files from the Internet Archive. The `research-wiki-test/` directory was created as an instance of the mnemotron-wiki-r template.

### 5.2 Initial Spectator ingest (1947–2025)

The first large ingest processed the Spectator corpus — hundreds of issues covering 1947 to 2025. Claude Code ran the four-stage ingest task (corpus assessment → document ingest → index update → git commit) on batches of issues. For batches of 50+ files, the Stage 0 taxonomy pass was critical: Claude established topic pages for major institutional themes before processing any source files, preventing source pages from becoming disconnected islands.

**Per-file synthesis was applied to every issue** — not just a sample. For the 1947–1980 period, issues were processed sequentially with a full per-file synthesis pass (each issue read completely, all relevant topics updated). For 1981–2025, five parallel synthesis agents ran simultaneously, each covering a distinct date range and writing to non-overlapping topic pages. This is a pattern the task document explicitly supports: agents writing to different topics can safely run in parallel because they are unlikely to edit the same file at the same time.

**The parallel agent pattern** requires careful setup. Each agent was given:
- A precise list of source files derived from targeted grep searches across the corpus
- The names of the specific topic pages it should write to
- Explicit instructions to use `Edit` (not `Write`) and to re-read the current file state before editing

Under-specified agents — those given only a date range and no file list or target topic map — tend to duplicate prior work or conflict with each other. Specificity is the key to safe parallelism. After all agents complete, `synthesize_links.py --rebuild` is run to backfill Related Topics cross-references across all source pages against the updated topic map.

### 5.3 Broadening the corpus

After the Spectator was ingested, the corpus was expanded in successive rounds:

**Documentary History of Hamilton College (1922):** A single large text — the college's own centenary history — was ingested as a close reading. This established the pre-1922 baseline: four topic pages (Founding, Kirkland/Oneida Mission, Early Campus, Early Governance) and four entity pages (Samuel Kirkland, Hamilton-Oneida Academy, Alexander Hamilton, Oneida Nation).

**Wikipedia exports:** Wikipedia pages for notable alumni and affiliates were downloaded as HTML and ingested. These added 12 new entity pages and expanded three existing ones. The HTML extractor in `extract_text.py` stripped navigation and advertising, leaving clean biographical text.

**Hamilton Life archive (1899–1942):** 1,113 issues of the college's original student newspaper were downloaded as djvu.txt files from Internet Archive and placed in `ingest/`. This was the largest single ingest: `batch_ingest.py` processed all files in a single run, creating 1,113 source pages. Claude then ran a synthesis pass on a representative sample (earliest, latest, and mid-range issues plus issues flagged as thematically rich in the Stage 0 sample read), producing the comprehensive Hamilton Life Archive topic essay.

**Course catalogs (1814–2025):** 205 catalogs were ingested, creating the Course Catalogs Collection and Curriculum and Academic Departments topic pages. Catalogs from the early 19th century are particularly useful for tracking which subjects were taught when.

**YHM archive materials:** 78 archival newspaper issues and 205 catalog entries from the Your Hamilton Museum were added to the corpus.

### 5.4 Deep synthesis passes

After the breadth passes were complete, targeted depth passes addressed specific open questions. These were typically run as parallel agents, each with a clearly bounded research question:

- BLSU/Black student history (1965–1985)
- Apartheid divestment arc (1986–1992)
- Chapel abolition history (1951–1965)
- Women's athletics and NESCAC founding (1971–1993)

Each agent was given: the question it was answering, the grep commands needed to find relevant source files, the names of the topic pages it should write to, and explicit instructions to use `Edit` (not `Write`) and to re-read the current file state before editing. This specificity is essential for parallel work — under-specified agents tend to overlap with each other or with prior work.

### 5.5 Consolidation and cleanup

Several topic pages were created as era-specific stubs during the breadth pass and later consolidated:

- **Five overlapping environmental stub pages** (`environmental-activism-1988-1995.md`, `environmental-activism-heag.md`, `environmental-activism-and-sustainability.md`, `heag-and-campus-sustainability.md`, and the original canonical) were merged into a single `environmental-sustainability-and-climate-action.md` covering the full 1972–2022 arc, with new content added from HEAG sources.
- **Three computing pages** (two era stubs plus a canonical page) were consolidated into `computing-and-technology-at-hamilton.md`; the stubs were expanded with new content from source files before becoming redirects.

Consolidation was done by a dedicated agent that read all source stubs, synthesized their content into the canonical unified page, and converted the stubs to **redirect pages** (not deleted). Redirect pages contain a brief notice and link pointing to the canonical page, which keeps any existing incoming links functional and makes the merge history legible.

### 5.6 Expansion: new topic pages via targeted corpus sweeps

After the initial breadth pass was complete, a second round of synthesis targeted specific topic gaps identified during depth passes. For each gap, the approach was:

1. Run targeted grep commands across all 3,510 source pages to find every source containing relevant terms
2. Group matches into era-based file lists (e.g., 1947–1969, 1970–1989, 1990–2009, 2010–2025)
3. Launch one parallel synthesis agent per new topic, giving it the complete file list and instructions to write a new topic page with Key Points, Sources table, Open Questions, and Related Topics sections
4. After all agents complete: run `synthesize_links.py --rebuild` to backfill Related Topics on all source pages, then run `scripts/open_questions.py` to regenerate `wiki/OPEN-QUESTIONS.md`

Topics added in this expansion round:

- **Residential Life and Campus Housing** — full sweep 1947–2020; covers the fraternity-dominated housing era, Kirkland opening, coed housing experiments, special-interest houses, Beinecke Village, and the REAL Program
- **International Students and ISA** — full sweep; note: the organization is the International Students *Association* (ISA), not Council — "ISC" in these sources refers to the Inter-Society Council (fraternity/sorority governing body); covers ISA programming, advocacy moments, post-9/11 visa rule changes, and the "From Where I Sit" column
- **Free Speech and Academic Freedom** — full sweep; covers the Ward Churchill (2005) and Susan Rosenberg (2004) controversies, the Kirkland Project founding and restructuring, and the founding of the Alexander Hamilton Center
- **Palestine Solidarity and Campus Activism** — full sweep 1982–2025; covers the full arc from PLO/Israeli speaker debates through the 2023–2025 SJP organizing and divestment campaigns

The same pattern — grep, group, parallel agents, rebuild — can be applied to any newly identified topic gap. The key constraint for parallelism is that agents must write to different files. If two topics share a target page, they must run sequentially.

### 5.7 Export

`wiki_export.py` was run to produce the Hamilton Gazetteer static site, exported to `hamilton-gazetteer/` (475 HTML files). During this step, the site was renamed from "Spectator Research Wiki" to "Hamilton Gazetteer" and a disclaimer was added crediting AI-assisted generation.

---

## 6. Claude Code's Role Beyond the Scripts

The scripts handle mechanics; Claude Code handles judgment. This distinction is worth making explicit because it clarifies both what can be automated further and what genuinely requires AI mediation.

### What the scripts do (deterministic, no AI required)

- Scanning `ingest/` for unprocessed files
- Hashing files and checking the manifest
- Extracting text from PDFs, HTML, and structured text files
- Running Tesseract OCR
- Generating URL-safe slugs
- Writing source pages from templates
- Keyword-matching source content against a topic map
- Updating `.manifest.json`
- Running `git add` and `git commit`

### What Claude Code does (requires judgment)

**Corpus assessment:** Before processing files, Claude reads a sample and identifies the document type, institutional context, time period, and recurring themes. It creates a taxonomy suited to the corpus — not a generic one copied from the tool's examples, but one calibrated to what is actually in the documents.

**OCR repair:** After Tesseract produces raw output, Claude applies a careful repair pass: correcting unambiguous character substitutions (0/O, 1/l/I, rn→m), joining hyphenated line breaks that are formatting artifacts, removing noise lines, flagging uncertain readings with `[?word?]`, and marking genuinely illegible passages `[illegible]`. This is a form of careful reading, not string manipulation — knowing whether "1" is a digit or the letter "l" requires understanding the sentence it appears in.

**Source interpretation:** Claude writes the `## Source Information` section of each source page: what the document is, who produced it, approximately when, and what context is relevant to understanding it. This section is not extracted from the document — it is inferred from it. A course catalog does not say "I am a course catalog from 1903"; Claude identifies it as such, notes its institutional context, and records that judgment.

**Topic synthesis:** Claude reads source pages and updates topic essays. This requires deciding what is significant, how a new finding relates to prior findings, and how to phrase a claim that draws on multiple sources. The `## Key Points` section of a topic page is genuine synthesis — not a concatenation of quotes, but a coherent analytical narrative with citations.

**Entity extraction:** Claude identifies which named individuals, organizations, and places are substantively present in a source (not merely mentioned in passing) and creates or updates entity pages. It determines the appropriate level of detail for a given entity based on how central it is to the research questions.

**Question formulation:** After each ingest pass, Claude populates the `## Open Questions` sections of topic pages with unresolved gaps it has noticed. These become the agenda for subsequent depth passes. A well-maintained Open Questions section is a research roadmap.

### The interaction model

In practice, the work is driven by a series of conversational exchanges in Claude Code, not by running scripts in isolation. A typical session might look like:

1. User: "Run the ingest task on these 50 new files"
2. Claude: Runs Stage 0, drafts taxonomy, checks for existing pages, creates 8 new topic stubs
3. User: "The Hamiltonews archive is 1942–1947 specifically, not just 'wartime' — please adjust the overview page accordingly"
4. Claude: Edits the overview page; continues with Stage 1 ingest
5. User: "Great. Now do a depth pass on the WWII topic — I want to know about the military training programs specifically"
6. Claude: Greps the source corpus, reads relevant files, updates the WWII topic page with specific findings

This is the "human-in-the-loop" part. The human is not doing the reading — Claude is — but the human is directing what to read, correcting misframings, and making judgments about what matters. The quality of the resulting wiki depends heavily on the quality of these interactions.

---

## 7. The Output

### Scale and coverage

At the time of documentation, the Hamilton Gazetteer contains:

| Content type | Count | Coverage |
|-------------|-------|---------|
| Source transcriptions | 3,510 | ~3,500 primary documents, 1793–2025 |
| Topic essays | 72 (66 active, 6 redirect stubs) | Institutional history, campus life, notable events, people and culture |
| Entity profiles | 410 | Notable alumni, faculty, administrators, organizations, buildings |
| Synthesis coverage | Comprehensive | Full per-file synthesis 1947–2025 via parallel agents; deep passes on BLSU/divestment/athletics; targeted topic sweeps for residential life, free speech, Palestine, international students |

### What makes a good wiki page

Source pages are transcriptions. A well-made source page accurately records the document's content, correctly identifies its origin and date, repairs OCR artifacts without introducing interpretive changes, and connects to relevant topic pages. Its value is as a stable, findable text record.

Topic pages are arguments. A well-made topic page synthesizes findings across multiple sources into a coherent narrative with specific citations. It records what is known, what is uncertain, and what is unknown. Its value grows as more sources are ingested — each new source either confirms existing Key Points, qualifies them, or adds new ones.

Entity pages are reference records. A well-made entity page gives a concise biography or description, explains the entity's relevance to Hamilton's history, and provides a trail of links into the source corpus. Its value is as a disambiguation and navigation aid.

### Notable discoveries documented in the wiki

Several historically significant findings emerged during synthesis that are worth noting as examples of what this kind of corpus-wide reading can surface:

- **B.F. Skinner '26** appears in 15 Hamilton Life issues. His full birth name "Burrhus F. Skinner" appears in an appendicitis notice from February 1924 — a detail that confirms his identity in the corpus and had not been previously documented in the college's own historical materials.
- **Peter Falk '49** appears in a Hamiltonews item from August 30, 1945, describing his service in the Merchant Marine — a biographical detail about the actor's wartime service documented in the student paper before his fame.
- **The Stryker Lusitania speech (May 12, 1915):** President M. Woolsey Stryker addressed the Hamilton community days after the Lusitania sinking and predicted U.S. entry into WWI within 12 months. The Life editors described it as "one of the greatest speeches ever made upon this Hill." This speech is not documented in existing Hamilton histories.
- **The 1932 FDR straw vote:** A fall 1932 issue of Hamilton Life records a campus presidential straw poll of 265 for Hoover vs. 89 for Roosevelt — a striking data point about Hamilton's political character during the Depression era.
- **The Ethics department suspension (1933):** The department was suspended as an "economic measure" during the Depression, documented in the February 1933 Life — an institutional decision that would be difficult to find without comprehensive newspaper coverage.

These discoveries did not require reading every issue in sequence. They emerged from the synthesis passes, which were guided by Open Questions about presidential eras, notable alumni, and Depression-era history.

---

## 8. Alternative Approaches and Their Tradeoffs

### 8.1 Full AI extraction vs. the tiered OCR approach

**Alternative:** Send every document to Claude Vision directly, skipping Tesseract entirely.

**Advantage:** Simpler pipeline; consistently high quality for difficult scans; no dependency on Tesseract or poppler.

**Disadvantage:** Much higher API cost. At roughly 1,500 tokens per page for a typical newspaper issue, processing 1,113 Hamilton Life issues (4–8 pages each) would cost thousands of dollars in API fees. Tesseract handles approximately 90% of clean printed text adequately at essentially zero cost; Claude Vision is most valuable for the 10% that Tesseract cannot read.

**When to use it:** When the corpus is small (fewer than 50 documents), when handwriting is prevalent, when scan quality is uniformly poor, or when cost is not a constraint.

### 8.2 Automated synthesis vs. human-directed synthesis

**Alternative:** Run fully automated synthesis — have Claude generate all topic pages from extracted text without human review or direction.

**Advantage:** Faster; no back-and-forth required; can process very large corpora overnight.

**Disadvantage:** Quality degrades in proportion to corpus size. Without human direction about what matters, Claude tends toward generic observations. The Hamilton Gazetteer's depth on specific topics (the Stryker Lusitania speech, the FDR straw poll, B.F. Skinner's footprint in the corpus) required a human to say "I want to know more about this." Fully automated synthesis produces a competent overview; human-directed synthesis produces a genuinely useful research instrument.

**When to use it:** For a first-pass breadth pass across a large corpus where the goal is coverage rather than depth; as a starting point to be refined by human-directed depth passes.

### 8.3 One large wiki vs. multiple focused wikis

**Alternative:** Split the corpus into separate wiki instances by time period, document type, or subject.

**Advantage:** Smaller wikis are faster to ingest, easier to navigate, and simpler for Claude to hold in context during synthesis. A wiki covering only 1903–1942 Hamilton Life issues would allow Claude to develop very deep expertise in that period.

**Disadvantage:** Cross-period connections are lost. One of the most valuable things the Gazetteer does is link entity pages across periods — Elihu Root '64 appears in sources spanning 1899–1949, and his entity page connects all of them. A split wiki would sever these connections.

**When to use it:** When a corpus is so large (10,000+ sources) that even the breadth-first approach becomes unwieldy; or when different corpus segments have different audiences and different research questions.

### 8.4 Markdown wiki vs. a database or graph store

**Alternative:** Store extracted data in a relational database or knowledge graph (RDF/SPARQL) rather than markdown files.

**Advantage:** More powerful querying; structured data extraction (dates, names, relationships) could be made precise; integration with existing library systems (ArchivesSpace, AtoM, CollectiveAccess) is more natural.

**Disadvantage:** Much higher setup cost; requires specialized expertise; harder for Claude Code to write to directly; loses the simplicity of "it is just files." The markdown wiki can be read, edited, and committed by any tool that understands text files, which makes it robust to tooling changes.

**When to use it:** When downstream integration with archival systems is a requirement; when the research questions demand structured queries (e.g., "all documents mentioning Elihu Root between 1905 and 1915"); or when the wiki has grown large enough that navigating it as files is impractical.

### 8.5 Git versioning vs. a database with change history

Git as the versioning mechanism for a wiki of 4,000+ files works but has friction: `git log` on a single page is useful; understanding "what changed across the whole wiki in a given synthesis session" requires reading a commit with 400 files changed. A dedicated version tracking layer would improve this. However, git provides one thing that is hard to replicate: it is already present, costs nothing, requires no setup, and Claude Code knows how to use it natively.

### 8.6 Internet Archive as primary source vs. local file management

This project used Internet Archive as the primary repository for digitized newspaper content. This has significant advantages for a library context:

- IA's Tesseract OCR (djvu.txt) is already computed and available; downloading it takes seconds
- Items remain accessible regardless of local storage constraints
- IA identifiers are stable, citable, and can be shared

The downside is dependency on IA's availability, their OCR quality decisions, and the fact that some institutional collections are not on IA or are access-restricted. For collections that live on local servers or institutional repositories, `batch_ingest.py` (processing local files) is the appropriate path.

---

## 9. How the Tool and Workflow Could Be Improved

These are areas where the current implementation has known limitations and where targeted improvements would have the highest impact.

### 9.1 Corpus-specific source page generators

**Current state:** `batch_ingest.py` ships with a generic page generator (`_document_page`) that derives a title from the filename and writes full text as content. For a corpus with a consistent naming convention (like `spec-YYYY-MM-DD-djvu.txt`), this produces pages titled "Spec 2006 10 27 Djvu" instead of "The Spectator, October 27, 2006." The script includes scaffolding and instructions for adding custom generators, but none have been written for the Hamilton Gazetteer corpus.

**Improvement:** Add corpus-specific generators to `_build_page()` in `batch_ingest.py` that parse the date from the filename, construct human-readable titles, record the date in frontmatter, and assign appropriate tags (publication title, decade, era) automatically. This would improve source page titles and make date-range searches possible without reading every page.

**General pattern for adaptation:** Any corpus with a meaningful naming convention should register a custom generator. The pattern is:
```python
# In _build_page():
if re.match(r"spec-\d{4}-\d{2}-\d{2}", filepath.stem):
    return _spectator_page(filepath, text, slug)
```

### 9.2 Structured metadata extraction

**Current state:** Source page frontmatter records basic provenance (title, type, OCR method, ingest date, original file). It does not record structured metadata like publication date, author, or volume/issue numbers.

**Improvement:** For newspaper corpora, add YAML frontmatter fields for `publication_date`, `publication`, `volume`, `issue`. For course catalogs, add `academic_year`. Structured dates would enable timeline navigation in the static export; structured publication fields would enable filtering by source type. Claude can infer most of these from document content or filename during ingest. This pairs naturally with corpus-specific generators (9.1) — a well-written generator would populate these fields automatically.

### 9.3 Embedding-based topic linking

**Current state:** `synthesize_links.py` uses keyword matching to link source pages to topic pages. This catches explicit mentions but misses implicit connections — a source about "the hill" might be related to "campus buildings" even if neither keyword appears literally.

**Improvement:** Replace or supplement keyword matching with embedding-based semantic search. Compute embeddings for source page content and topic page content; link source pages to the most semantically similar topic pages above a similarity threshold. This would improve recall significantly for sources that use period-specific vocabulary or indirect language.

### 9.4 Finding aid and catalog export formats

**Current state:** `wiki_export.py` produces a clean static HTML site suitable for browsing. It does not produce any of the standard archival finding aid formats (EAD, Dublin Core, MODS, MARC) that would integrate with institutional discovery systems.

**Improvement:** Add export modes for:
- **EAD XML:** Use source page frontmatter and content to generate an Encoded Archival Description finding aid. Title, date, extent, scope/content, and subject terms are all available in the wiki's structured data.
- **Dublin Core CSV:** A simple spreadsheet format for each source page: title, date, creator (publication/author), description (Source Information section), subject terms (tags), identifier (original_file/IA identifier).

For a library context, the most immediate practical value would be Dublin Core export: a CSV that can be imported into ArchivesSpace, AtoM, or ContentDM to make the Gazetteer's contents discoverable through the library's standard catalog.

### 9.5 Parallel agent coordination

**Current state:** Parallel synthesis agents are launched by hand with carefully specified prompts. There is no formal mechanism to track which agents are running, which topic pages they are writing to, or whether they conflict.

**Improvement:** A lightweight coordination manifest (a JSON file listing active agents and their target pages) that each agent reads before starting and writes to on completion. This would allow Claude Code to detect conflicts before they happen and alert the user if two agents claim the same target page.

---

## 10. Reproducing This for a Similar Project

This section is a condensed practical guide for someone wanting to apply this workflow to a comparable archival corpus — for example, a college's institutional newspaper archive, a collection of annual reports, or a set of finding aids.

### Step 1: Assemble your corpus

Identify your primary sources. For an IA-hosted collection, find the item identifiers (the IA search interface can export CSV lists). For local files, organize them in a folder you can point `ingest/` at. Mixed corpora (some local, some IA) work fine — use `batch_ingest.py` for local files and `ia_ingest.py` for IA items.

**What makes a good corpus for this approach:**
- Sufficient volume to justify the setup cost (≥ 50 documents; the approach becomes most powerful at ≥ 500)
- A coherent subject focus (a single institution, a single project, a defined research question)
- Text-bearing content (printed text, typed text, HTML) — handwritten material works but is more expensive
- Documents on Internet Archive or available as files in standard formats (PDF, TXT, HTML)

### Step 2: Set up the wiki instance

```bash
# Copy mnemotron-wiki-r to your project directory
cp -r mnemotron-wiki-r/ my-project-wiki/
cd my-project-wiki/
bash setup.sh
export ANTHROPIC_API_KEY="sk-ant-..."
```

Edit `scripts/config.py` if you need to change any default paths. The defaults work for most cases.

### Step 3: Load your corpus into `ingest/`

For IA collections: create `ingest/ia-sources/search.csv` with one identifier per row, then `python ia_ingest.py --dry-run` to preview the run before downloading anything.

For local files: copy or symlink them into `ingest/`.

**Do not delete originals.** The tool does not remove files from `ingest/` after processing; the markdown source pages are the canonical retained form. Retain your local copies elsewhere until you have verified the wiki pages are correct.

### Step 4: Run Stage 0 before processing any files

Open a Claude Code session in the wiki directory and say: "Run the research wiki ingest task." Claude will read `RESEARCH_WIKI_TASK.md` and begin with Stage 0.

Before Claude processes any files, intervene if needed:
- If the default taxonomy Claude proposes does not match your research questions, say so now
- If there are topic categories that are specific to your corpus and not in the generic checklist, add them
- If you have prior knowledge about the corpus (a finding aid, a collection description, a history) that Claude should read first, provide it

The Stage 0 conversation is the most important part of the whole process. Time spent here prevents source pages from being islands.

### Step 5: Process in batches

For large corpora (1,000+ files), process in batches:
1. `python batch_ingest.py --limit 100` to create the first 100 source pages
2. Run a Claude synthesis pass on the sample
3. Continue `batch_ingest.py` in increments; run synthesis passes at regular intervals

Do not wait until all source pages are created before synthesizing. Synthesis guides subsequent ingestion: if the first 100 issues of a newspaper establish that the 1920s are particularly rich in a given topic, you know to look more carefully at 1920s issues as they are ingested.

### Step 6: Use parallel agents for large synthesis passes

Once the topic taxonomy is established and source pages exist, parallel agents dramatically accelerate synthesis of large corpora. The pattern:

1. Grep the source corpus for terms related to each topic you want to develop
2. Group results by era (e.g., pre-1970, 1970–1990, 1990–2010, 2010–present)
3. Launch one agent per topic (not per era), giving each a complete file list and the name of its target topic page
4. After all agents complete, run `python synthesize_links.py --rebuild` and `python scripts/open_questions.py`
5. Commit everything in a single commit with a descriptive message

The essential constraint: each agent must write to a different set of files. Two agents writing to the same topic page will produce conflicts. If a topic requires contributions from multiple agents, run them sequentially.

### Step 7: Iterate between breadth and depth

The synthesis workflow has two modes:
- **Breadth (Stage 1):** Process files, link each to existing topics, update topic pages with new findings
- **Depth (Stage 4):** Pick a specific open question, grep for relevant sources, update the topic page with a focused analysis

Do not stay in either mode exclusively. Breadth without depth produces many thinly populated topic pages. Depth without breadth risks a wiki that is deep on a few questions and silent on others. A reasonable rhythm for a large newspaper corpus: one full breadth pass, then two or three depth passes, then another breadth pass for any newly ingested material.

### Step 8: Maintain as you ingest

The wiki is most useful as a living document, not a one-time product. When new material becomes available:
1. Add it to `ingest/`
2. Run `batch_ingest.py` (or `ia_ingest.py`)
3. Run a Claude synthesis pass
4. Run `python synthesize_links.py --rebuild`
5. Run `python scripts/open_questions.py`
6. Commit

The manifest system means there is no risk of reprocessing files you have already handled. New material is seamlessly integrated into existing topic and entity pages.

### Step 9: Export when needed

```bash
python wiki_export.py              # export topics and entities only
python wiki_export.py --all        # include source pages
python wiki_export.py --clean -o my-export/  # fresh export to named directory
```

The export is a static HTML site with no server-side dependencies. It can be hosted on any web server, on GitHub Pages, or simply distributed as a zip file.

---

## Appendix A: Project Statistics

| Metric | Value |
|--------|-------|
| Wiki source pages | 3,510 |
| Topic essays | 72 (66 active, 6 redirect stubs) |
| Entity profiles | 410 |
| Primary sources by type | ~1,113 Hamilton Life; ~78 Hamiltonews; ~hundreds of Spectator; 205 catalogs; 1 documentary history; ~19 Wikipedia exports; 78+ YHM archive items |
| Date range covered | 1793–2025 |
| Synthesis coverage | Complete (1947–2025 full per-file synthesis via parallel agents; 1899–1942 comprehensive breadth + sample-based depth; 1793–1922 documentary sources; targeted topic sweeps for residential life, free speech, Palestine solidarity, international students) |
| Git commits | Multiple; history in `research-wiki-test/.git/` |
| Static export | `hamilton-gazetteer/` (475 HTML files) |

---

## Appendix B: File Reference

| File | Purpose |
|------|---------|
| `mnemotron-wiki-r/batch_ingest.py` | Bulk source page creation from local files |
| `mnemotron-wiki-r/ia_ingest.py` | Source page creation from Internet Archive items |
| `mnemotron-wiki-r/synthesize_links.py` | Auto-generated topic cross-referencing (runtime TOPIC_MAP; `--rebuild` flag) |
| `mnemotron-wiki-r/scripts/config.py` | Central configuration (paths, OCR settings) |
| `mnemotron-wiki-r/scripts/extract_text.py` | Text extraction from native document formats |
| `mnemotron-wiki-r/scripts/ocr.py` | Tiered OCR: Tesseract + Claude Vision fallback |
| `mnemotron-wiki-r/scripts/manifest.py` | Content-hash tracking of processed files |
| `mnemotron-wiki-r/scripts/check_ingest.py` | Lists unprocessed files in `ingest/` |
| `research-wiki-test/RESEARCH_WIKI_TASK.md` | Claude Code operating instructions for the ingest task |
| `research-wiki-test/wiki_export.py` | Markdown to static HTML export |
| `research-wiki-test/scripts/open_questions.py` | Regenerates `wiki/OPEN-QUESTIONS.md` from all topic pages |
| `research-wiki-test/scripts/quality_report.py` | OCR quality summary; flags pages for manual review |
| `research-wiki-test/.manifest.json` | Ingest history (committed to git) |
| `research-wiki-test/ingest/ia-sources/processed.json` | IA identifier log |
| `research-wiki-test/wiki/` | The wiki itself: sources/, topics/, entities/, INDEX.md |
| `research-wiki-test/hamilton-gazetteer/` | Static HTML export |

---

## Appendix C: Glossary

**Breadth pass:** An ingest run in which each source file is linked to existing topics and topics are updated with findings, aiming for wide coverage of the corpus. Contrast with depth pass.

**Corpus assessment (Stage 0):** The pre-ingest step in which Claude reads a sample of the incoming batch, characterizes its content, and establishes the topic and entity taxonomy needed to receive it.

**Depth pass (Stage 4):** A targeted synthesis run driven by a specific open question, using grep to find relevant source files rather than reading files in sequence. Contrast with breadth pass.

**djvu.txt:** Internet Archive's pre-built Tesseract OCR output for scanned text items. Available for most IA text items; downloading it is much faster than downloading the original PDF and running local OCR.

**Entity page:** A wiki page profiling a named person, organization, or place that appears substantively in the source corpus.

**Idempotency:** The property of an operation that can be run multiple times without producing different results. The manifest system makes the ingest pipeline idempotent: running it on files you have already processed has no effect.

**Manifest:** The `.manifest.json` file tracking which source files have been processed, identified by content hash.

**OCR (Optical Character Recognition):** The process of converting a scanned image of text into machine-readable text. Tesseract is the offline OCR engine used in this system; Claude Vision is the fallback for pages Tesseract cannot read.

**Parallel agent:** A Claude Code agent running simultaneously with one or more others, each assigned to a distinct set of target files or topic pages. Safe when agents write to non-overlapping files; requires care when topics share target pages.

**Redirect page:** A topic page that was superseded by a canonical page. Contains a brief notice and link to the canonical page; preserved (not deleted) to keep incoming links functional.

**Source page:** A wiki page containing the extracted or transcribed text of a single source document, along with provenance metadata.

**Topic page:** A wiki page synthesizing findings across multiple source pages on a common theme. Contains Overview, Key Points, Open Questions, Sources table, and Related Topics.

**Tiered OCR:** The system's approach to OCR: try the cheapest option first (IA's pre-built djvu.txt, or local Tesseract), fall back to the more expensive option (local OCR pipeline, or Claude Vision) only when the cheaper option fails quality checks.

---

