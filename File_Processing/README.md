# File Processing Skills

Skills for reading, extracting, and processing uploaded files. Acts as a router — selecting the correct reading strategy by file type so Claude never blindly `cat`s a binary.

## Skills in This Category

| Skill | File Types | Trigger |
|-------|------------|---------|
| [General File Reading](#general) | All types | Any uploaded file path present in context |
| [PDF Reading](#pdf-reading) | .pdf | Uploaded .pdf requiring extraction or analysis |

---

## General

Routes to the correct reading tool and strategy by file extension.

**Routing table:**

| Extension | Strategy |
|-----------|----------|
| .pdf | PDF reading skill |
| .docx | python-docx outline mode (offset=0) or raw XML (offset>0) |
| .xlsx / .xlsm | openpyxl `read_only=True, data_only=True` |
| .xlsb | pandas `engine='pyxlsb'` |
| .csv / .tsv | pandas `read_csv` |
| .json | `json.load` |
| .png / .jpg / .jpeg / .gif / .webp | View tool — Claude sees natively, no extraction needed |
| .txt / .md / .html | `read_text_file` |
| .zip | `zipfile` + iterate contents |
| .pptx | python-pptx shape text extraction |

**Key behaviors:**
- Checks whether file content is already in the context window before reaching for a tool
- Never runs `cat` on binary files
- For very large files, reads in chunks using offset/length pagination

---

## PDF Reading

Selects the right reading strategy based on the PDF content type.

**Strategy selection:**

| Content Type | Indicators | Approach |
|-------------|------------|----------|
| Text-heavy document | Selectable text in viewer | pdfminer.six or pymupdf text extraction |
| Scanned / image-based | No selectable text | pdf2image + pytesseract OCR |
| Slide deck | Large images, minimal text per page | Page-by-page rasterization + vision |
| Data tables | Grid layouts, numeric content | camelot or tabula-py |
| Fillable form | Form fields, interactive elements | pypdf form field extraction |

**Key behaviors:**
- Attempts text extraction first; falls back to OCR only if text is empty or garbled
- Paginates large documents using offset/length to avoid loading everything at once
- Extracts embedded images when requested separately from text
- Reports page count and structure before diving into content
