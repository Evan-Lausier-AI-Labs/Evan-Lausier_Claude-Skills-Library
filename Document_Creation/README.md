# Document Creation Skills

Skills for generating professional documents in Word, PDF, PowerPoint, and Excel formats. These skills ensure Claude follows best-practice library patterns for each format rather than guessing.

## Skills in This Category

| Skill | Trigger Phrases | Output |
|-------|-----------------|--------|
| [Word](#word) | "Word doc", "docx", "report", "memo", "letter", "write a document" | .docx |
| [PDF](#pdf) | "PDF", "create a PDF", "fill a form", "merge PDFs", "extract from PDF" | .pdf |
| [PowerPoint](#powerpoint) | "deck", "slides", "presentation", ".pptx", "pitch deck" | .pptx |
| [Excel](#excel) | "spreadsheet", ".xlsx", "Excel", "CSV to spreadsheet" | .xlsx |

---

## Word

Generates high-quality `.docx` files with proper styles, headings, tables of contents, page numbers, and professional layout using python-docx.

**Key behaviors:**
- Reads the `docx` SKILL.md before writing any code
- Uses built-in Word styles (Heading 1/2/3, Normal, Table) rather than manual formatting
- Delivers to `/mnt/user-data/outputs/` for download
- Offers table of contents and page numbers for documents over 5 pages

**Do NOT use for:** Google Docs, plain markdown, or content the user will read inline.

---

## PDF

Handles PDF creation, text extraction, merging, splitting, watermarking, form-filling, and OCR depending on the task.

**Library routing:**

| Task | Library |
|------|---------|
| Create from scratch | reportlab or fpdf2 |
| Extract text | pdfminer.six or pymupdf |
| Merge / split | pypdf |
| Fill form fields | pypdf or pdfrw |
| OCR scanned doc | pytesseract + pdf2image |

**Key behaviors:**
- Reads the `pdf` SKILL.md to select the right approach
- Distinguishes text-based PDFs from scanned image PDFs before choosing a strategy
- Never uses `pdftotext` directly without checking for scanned content first

---

## PowerPoint

Generates professional slide decks with consistent layouts, speaker notes, and correct use of slide masters and placeholder types using python-pptx.

**Key behaviors:**
- Always reads the `pptx` SKILL.md before generating code
- Uses `slide.shapes.placeholders` to place content in the right zones
- Applies consistent fonts and color scheme across all slides
- Adds speaker notes when content complexity warrants it

**Do NOT use for:** Simple bullet lists the user will read inline; offer a .pptx only when the user clearly wants a file.

---

## Excel

Creates and manipulates spreadsheets including formulas, charts, conditional formatting, named ranges, and multi-sheet workbooks using openpyxl.

**Key behaviors:**
- Reads the `xlsx` SKILL.md before generating code
- Uses `read_only=True, data_only=True` when reading existing files
- Uses `engine='pyxlsb'` for `.xlsb` files (requires `pyxlsb` package)
- Applies `number_format` and `alignment` for clean presentation
- Creates charts within the workbook rather than as separate image files
