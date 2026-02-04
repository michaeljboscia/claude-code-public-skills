---
name: pptx-toolkit
description: "PowerPoint presentation creation, editing, analysis, and conversion. Use when Claude needs to: create presentations from scratch (HTML→PPTX), create from templates (rearrange + replace), edit existing PPTX via OOXML, extract text/thumbnails, or convert formats."
shareable: true
---

# PPTX Toolkit — General-Purpose Skill

**Repo:** `/Users/mikeboscia/claude-office-skills`
**Purpose:** Wraps all four PPTX workflows (template, from-scratch, OOXML edit, analysis) into one decision-driven reference.
**Last Updated:** 2026-02-04

---

## Overview

This skill provides four workflows for working with PowerPoint files using the `claude-office-skills` repository. All Python commands use `venv/bin/python` (never system Python). All outputs go to `outputs/<name>/` inside the repo.

**CRITICAL**: Before starting any PPTX work, `cd /Users/mikeboscia/claude-office-skills` first.

---

## Related Documentation (DRY — Read These for Deep Dives)

| Document | Path | When to Read |
|----------|------|-------------|
| **SKILL.md** | `public/pptx/SKILL.md` | Template workflow details, design principles, thumbnail creation |
| **html2pptx.md** | `public/pptx/html2pptx.md` | From-scratch HTML→PPTX constraints, PptxGenJS API, chart/table reference |
| **ooxml.md** | `public/pptx/ooxml.md` | Raw XML editing reference (shapes, text formatting, images, tables) |
| **CLAUDE.md** | `CLAUDE.md` | Repo conventions, common commands, output directory rules |

**Rule:** Read the relevant doc completely (no range limits) before starting a workflow.

---

## Decision Matrix — Which Workflow?

| Situation | Workflow | Why |
|-----------|----------|-----|
| Create branded proposal from existing deck | **A: Template-Based** | Preserves design, masters, grouped shapes |
| Create new presentation from scratch | **B: HTML→PPTX** | Full design control, charts, custom layouts |
| Edit specific slides in existing PPTX | **C: OOXML Direct** | Surgical XML changes, add images/notes |
| Extract text, thumbnails, or analyze | **D: Analysis** | Read-only inspection and conversion |
| Modify tables in existing PPTX | **C: OOXML Direct** | Tables are graphicFrame elements — replace.py cannot modify them |

---

## Workflow A: Template-Based Creation

### Quick Start

```bash
cd /Users/mikeboscia/claude-office-skills
mkdir -p outputs/<name>/

# 1. Extract text + create thumbnails
venv/bin/python -m markitdown "template.pptx"
venv/bin/python public/pptx/scripts/thumbnail.py "template.pptx" outputs/<name>/template-grid

# 2. Analyze → save template-inventory.md (list ALL slides with 0-based indices)

# 3. Rearrange slides (0-indexed, comma-separated)
venv/bin/python public/pptx/scripts/rearrange.py "template.pptx" outputs/<name>/working.pptx 0,3,3,5,8

# 4. Extract text inventory
venv/bin/python public/pptx/scripts/inventory.py outputs/<name>/working.pptx outputs/<name>/text-inventory.json

# 5. Create replacements.json → apply
venv/bin/python public/pptx/scripts/replace.py outputs/<name>/working.pptx outputs/<name>/replacements.json outputs/<name>/final.pptx
```

### Detailed Steps

1. **Extract & thumbnail** — Run `markitdown` for text and `thumbnail.py` for visual grid. Read both completely.
2. **Inventory analysis** — Create `template-inventory.md` listing every slide (0-indexed) with purpose and layout type.
3. **Outline mapping** — Create `outline.md` mapping your content to template slide indices. Verify indices are within range.
4. **Rearrange** — `rearrange.py` duplicates, reorders, and deletes in one pass. Same index can repeat to duplicate.
5. **Text inventory** — `inventory.py` extracts all shapes with positions, fonts, overflow detection. Read entire JSON.
6. **Replacement JSON** — Generate `replacements.json` matching the inventory structure.
7. **Apply** — `replace.py` clears all text shapes, then applies only those with `"paragraphs"` defined.

### Replacement JSON Format

```json
{
  "slide-0": {
    "shape-0": {
      "paragraphs": [
        {
          "text": "Title text here",
          "alignment": "CENTER",
          "bold": true
        },
        {
          "text": "Subtitle or body text",
          "font_size": 18.0,
          "color": "595959"
        },
        {
          "text": "A bullet point without bullet symbol in text",
          "bullet": true,
          "level": 0
        },
        {
          "text": "Theme-colored text",
          "theme_color": "DARK_1"
        }
      ]
    }
  }
}
```

### Paragraph Properties Reference

| Property | Type | Notes |
|----------|------|-------|
| `text` | string | Required. Do NOT include bullet symbols (•, -, *) — added automatically |
| `bold` | boolean | Header/title text |
| `italic` | boolean | Emphasis |
| `underline` | boolean | Underline |
| `bullet` | boolean | When true, `level` is required |
| `level` | integer | Indent level (0-based). Required when `bullet: true` |
| `alignment` | string | `"CENTER"`, `"RIGHT"`. Omit for default left. **Do NOT set on bullets** — auto left-aligned |
| `font_name` | string | e.g., `"Arial"`, `"Century Gothic"` |
| `font_size` | float | Points, e.g., `14.0` |
| `color` | string | RGB hex without #, e.g., `"FF0000"` |
| `theme_color` | string | e.g., `"DARK_1"`, `"TEXT_1"`, `"BACKGROUND_1"` |
| `space_before` | float | Points before paragraph |
| `space_after` | float | Points after paragraph |
| `line_spacing` | float | Line spacing in points |

### Critical Rules (Template Workflow)

- **Auto-clear**: ALL text shapes are cleared. Only shapes with `"paragraphs"` in replacement JSON get content.
- **No bullet symbols**: Never include •, -, * in text — `"bullet": true` adds them automatically.
- **Bullet alignment**: Paragraphs with `"bullet": true` are automatically left-aligned. Do NOT set `"alignment"` on bullets.
- **Shape validation**: replace.py validates that referenced shapes exist. Errors list available shapes.
- **Overflow detection**: replace.py reports if replacement text worsens overflow.
- **Read inventory completely**: Never set range limits when reading text-inventory.json.

---

## Workflow B: From-Scratch (HTML→PPTX)

### Quick Start

```bash
cd /Users/mikeboscia/claude-office-skills
mkdir -p outputs/<name>/images/

# 1. Read html2pptx.md completely (MANDATORY)
# 2. Create HTML files for each slide (720pt x 405pt for 16:9)
# 3. Write and run a Node.js script using html2pptx.js
node outputs/<name>/build.js

# 4. Visual validation
venv/bin/python public/pptx/scripts/thumbnail.py outputs/<name>/presentation.pptx outputs/<name>/thumbnails
```

### HTML Constraints (Memorize These)

| Constraint | Details |
|------------|---------|
| **Body dimensions** | 16:9 = `720pt × 405pt`; 4:3 = `720pt × 540pt` |
| **Text must be in** | `<p>`, `<h1>`-`<h6>`, `<ul>`, `<ol>` — text in bare `<div>` or `<span>` is silently lost |
| **No `<br>` tags** | Use separate `<p>` elements instead |
| **No CSS gradients** | Rasterize to PNG with Sharp first, then use as `<img>` or background-image |
| **No manual bullets** | Use `<ul>`/`<ol>` — never •, -, * symbols |
| **Fonts** | Web-safe only: Arial, Helvetica, Times New Roman, Georgia, Courier New, Verdana, Tahoma, Trebuchet MS |
| **Shape styling** | Backgrounds, borders, shadows work ONLY on `<div>` elements, not text elements |
| **Colors in PptxGenJS** | NEVER use `#` prefix — causes file corruption. Use `"FF0000"` not `"#FF0000"` |
| **`display: flex`** | Use on body to prevent margin collapse breaking overflow validation |

### Design Principles

Before writing code, analyze the content and choose a color palette. See SKILL.md for 18 example palettes and visual detail options (geometric patterns, border treatments, typography, chart styling, layout innovations).

**For detailed API reference** (charts, tables, images, placeholders, PptxGenJS): Read `public/pptx/html2pptx.md` completely.

---

## Workflow C: OOXML Direct Editing

### Quick Start

```bash
cd /Users/mikeboscia/claude-office-skills

# 1. Read ooxml.md completely (MANDATORY)

# 2. Unpack
venv/bin/python public/pptx/ooxml/scripts/unpack.py input.pptx outputs/<name>/unpacked/

# 3. Edit XML files (ppt/slides/slideN.xml, etc.)

# 4. MANDATORY: Validate before packing
venv/bin/python public/pptx/ooxml/scripts/validate.py outputs/<name>/unpacked/ --original input.pptx

# 5. Pack
venv/bin/python public/pptx/ooxml/scripts/pack.py outputs/<name>/unpacked/ outputs/<name>/final.pptx
```

### Key File Paths (Inside Unpacked Structure)

| Path | Contains |
|------|----------|
| `ppt/presentation.xml` | Slide references, slide order (`<p:sldIdLst>`) |
| `ppt/slides/slideN.xml` | Individual slide content |
| `ppt/notesSlides/notesSlideN.xml` | Speaker notes |
| `ppt/slideMasters/` | Master slide templates |
| `ppt/slideLayouts/` | Layout templates |
| `ppt/theme/theme1.xml` | Colors (`<a:clrScheme>`), fonts (`<a:fontScheme>`) |
| `ppt/media/` | Images and media |
| `ppt/_rels/presentation.xml.rels` | Presentation relationships |
| `ppt/slides/_rels/slideN.xml.rels` | Per-slide relationships (images, layouts) |
| `[Content_Types].xml` | Content type declarations |
| `docProps/app.xml` | Slide count, statistics |

### Critical Rules (OOXML Workflow)

- **ALWAYS validate before packing** — never pack without validation.
- **Element order in `<p:txBody>`**: `<a:bodyPr>`, `<a:lstStyle>`, `<a:p>` (order matters).
- **Add `dirty="0"`** to `<a:rPr>` and `<a:endParaRPr>` elements.
- **Whitespace**: Add `xml:space='preserve'` to `<a:t>` with leading/trailing spaces.
- **Unicode**: Escape in ASCII content — `"` becomes `&#8220;`.
- **Slide operations**: When adding/deleting slides, update `[Content_Types].xml`, `presentation.xml.rels`, `presentation.xml`, and `docProps/app.xml`.
- **Don't renumber**: After deleting slides, keep original IDs and filenames.

---

## Workflow D: Analysis & Extraction

### Commands

| Task | Command |
|------|---------|
| **Extract text (markdown)** | `venv/bin/python -m markitdown "file.pptx"` |
| **Text inventory (JSON)** | `venv/bin/python public/pptx/scripts/inventory.py "file.pptx" outputs/<name>/inventory.json` |
| **Thumbnail grid** | `venv/bin/python public/pptx/scripts/thumbnail.py "file.pptx" outputs/<name>/grid [--cols 4]` |
| **Convert to PDF** | `soffice --headless --convert-to pdf --outdir outputs/<name>/ "file.pptx"` |
| **PDF → page images** | `pdftoppm -jpeg -r 150 file.pdf outputs/<name>/slide` |
| **Specific page range** | `pdftoppm -jpeg -r 150 -f 2 -l 5 file.pdf outputs/<name>/slide` |

### Thumbnail Grid Details

- Default: 5 columns, max 30 slides per grid (5x6)
- Custom columns: `--cols 4` (range 3-6)
- Grid limits: 3 cols = 12/grid, 4 cols = 20/grid, 5 cols = 30/grid, 6 cols = 42/grid
- For large decks, multiple numbered grid images are created
- Slides are 0-indexed in the grid labels

---

## All Commands Reference

| Script | Purpose | Example |
|--------|---------|---------|
| `venv/bin/python -m markitdown` | Text extraction to markdown | `venv/bin/python -m markitdown "deck.pptx"` |
| `public/pptx/scripts/inventory.py` | JSON text inventory with positions/fonts | `venv/bin/python public/pptx/scripts/inventory.py deck.pptx out/inv.json` |
| `public/pptx/scripts/replace.py` | Apply replacement JSON to PPTX | `venv/bin/python public/pptx/scripts/replace.py in.pptx rep.json out.pptx` |
| `public/pptx/scripts/rearrange.py` | Duplicate/reorder/delete slides | `venv/bin/python public/pptx/scripts/rearrange.py in.pptx out.pptx 0,3,3,5` |
| `public/pptx/scripts/thumbnail.py` | Visual thumbnail grid | `venv/bin/python public/pptx/scripts/thumbnail.py deck.pptx out/grid --cols 4` |
| `public/pptx/scripts/html2pptx.js` | HTML→PPTX library (Node.js) | `node build-script.js` (uses `require('./html2pptx')`) |
| `public/pptx/ooxml/scripts/unpack.py` | Extract PPTX to XML | `venv/bin/python public/pptx/ooxml/scripts/unpack.py in.pptx out/unpacked/` |
| `public/pptx/ooxml/scripts/validate.py` | Validate unpacked XML | `venv/bin/python public/pptx/ooxml/scripts/validate.py out/unpacked/ --original in.pptx` |
| `public/pptx/ooxml/scripts/pack.py` | Repack XML to PPTX | `venv/bin/python public/pptx/ooxml/scripts/pack.py out/unpacked/ out/final.pptx` |
| `soffice` | PPTX → PDF conversion | `soffice --headless --convert-to pdf --outdir out/ "deck.pptx"` |
| `pdftoppm` | PDF → page images | `pdftoppm -jpeg -r 150 file.pdf out/slide` |

---

## Troubleshooting

### Template Workflow Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `Shape 'shape-X' not found on 'slide-Y'` | Replacement JSON references non-existent shape | Read text-inventory.json; only reference shapes that exist |
| `Slide 'slide-X' not found in inventory` | Replacement JSON references non-existent slide | Check slide count after rearrange; slides are 0-indexed |
| `overflow worsened by X"` | Replacement text too long | Shorten text or reduce font_size |
| Text cleared but no replacement shown | Shape missing from replacement JSON | All shapes auto-clear; add `"paragraphs"` for shapes you want to keep |
| Bullets show duplicate symbols | Manual bullet chars in text + `"bullet": true` | Remove •, -, * from text — bullets auto-added |
| Centered text on bullets | `"alignment": "CENTER"` set with `"bullet": true` | Remove alignment on bullet paragraphs — auto left-aligned |

### HTML→PPTX Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Text missing from slide | Text in bare `<div>` or `<span>` | Wrap in `<p>`, `<h1>`-`<h6>`, `<ul>`, or `<ol>` |
| Corrupted PPTX file | `#` in PptxGenJS colors | Use `"FF0000"` not `"#FF0000"` |
| Gradient not showing | CSS `linear-gradient` used | Rasterize to PNG with Sharp first |
| Content overflow error | Content exceeds body dimensions | Adjust content size or increase body dimensions |
| Dimension mismatch error | HTML body size ≠ pptx.layout | Match body (720pt×405pt) to `LAYOUT_16x9` |
| Chart with 1 data point | Wrong time aggregation | Match granularity: <30d=daily, 30-365d=monthly, >365d=yearly |

### OOXML Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| PowerPoint won't open file | Packed without validation | Always validate before packing |
| Text not appearing | Wrong element order in `<p:txBody>` | Must be: `<a:bodyPr>`, `<a:lstStyle>`, `<a:p>` |
| Broken references after delete | `_rels` files still reference deleted slides | Clean up all relationship files |
| Multiple notes slides conflict | Duplicated slide shares notes reference | Remove or update notes references in `_rels` |
| Missing slideLayout errors | Content_Types.xml incomplete | Declare ALL slides, layouts, themes in `[Content_Types].xml` |

---

## Prerequisites

### Python (via venv)
- `python-pptx`, `openpyxl`, `pypdf` — Office format handling
- `defusedxml`, `lxml` — XML processing
- `Pillow`, `pdf2image` — Image handling
- `markitdown` — Text extraction
- **Always use:** `venv/bin/python` (never system Python)

### Node.js
- `pptxgenjs` (v4.0.1) — Presentation generation
- `playwright` + Chromium — HTML rendering
- `sharp` — Image processing, SVG rasterization
- `react`, `react-dom`, `react-icons` — Icon rasterization

### System Tools
- `soffice` (LibreOffice) — PPTX → PDF conversion
- `pdftoppm` (Poppler) — PDF → image conversion
- `pandoc` — Document text extraction

### Install / Verify

```bash
cd /Users/mikeboscia/claude-office-skills

# Python deps
venv/bin/pip install -r requirements.txt

# Node deps
npm install

# System tools (verify presence)
which soffice pdftoppm pandoc
```
