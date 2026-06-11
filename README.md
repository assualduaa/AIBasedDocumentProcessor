# PDF File Processor

An intelligent PDF classification, renaming, and automatic filing system built with Flask. Designed for document management workflows where PDFs need to be classified by entity, document type, department, and language — then automatically renamed and moved into the correct folder structure.

---

## Project Goal

Manual document filing is slow, error-prone, and inconsistent — especially at scale or across bilingual organizations. The goal of this project is to fully automate the PDF intake process for a CEO Office Document Management System (DMS).

The system should:

- **Eliminate manual effort** — no human should need to read, rename, or sort incoming PDFs
- **Enforce consistent naming** — every filed document follows the same structured naming convention
- **Handle uncertainty gracefully** — documents that cannot be classified with confidence are flagged for review rather than silently misfiled
- **Support Arabic and English** — both language documents are processed equally, including scanned (image-only) PDFs via OCR
- **Provide full traceability** — every action (move, rename, delete, skip) is logged and auditable

---

## Main Steps

The system follows these steps for every PDF it receives:

**1. Receive the file** — via web UI upload, API call, folder scan, or the hot-folder watcher daemon.

**2. Validate** — confirm the file is a PDF. Non-PDF files are rejected (single mode) or skipped with a warning (batch mode).

**3. Extract text** — pdfplumber reads selectable text from the PDF. If the document is a scanned image, Tesseract OCR is used as a fallback.

**4. Detect language** — the document is identified as English or Arabic. This affects keyword matching and folder routing.

**5. Classify** — the classifier scores the document across three scenarios:
   - **Scenario 1** — explicit folder structure found on the first page (highest confidence)
   - **Scenario 2** — entity, document type, and department identified from keyword matching
   - **Scenario 3** — ambiguous content; triggers Intelligent Prediction Mode

**6. Predict folder (if needed)** — when standard classification is uncertain, TF-IDF cosine similarity and RapidFuzz fuzzy matching select the best-matching folder from the known DMS structure. If the prediction score meets the confidence threshold, the file is auto-filed; otherwise it is routed to *Uncategorized / Review Required*.

**7. Build the new filename** — a structured name is assembled from the extracted metadata: `YYYY_Entity_DocType_Subject_Client.pdf`.

**8. Calculate destination path** — the full output folder path is constructed dynamically based on entity, year, department, and document type.

**9. Create folder if missing** — the destination directory is created automatically if it does not exist.

**10. Move the file** — the PDF is copied to the destination, then the byte size is verified. Only after verification passes is the original source file deleted (when deletion is enabled).

**11. Write audit record** — a structured JSON entry is appended to `logs/audit_log.json` capturing the original path, destination, rename, confidence score, and outcome.

---

## Features

- **Intelligent 3-Scenario Classification Pipeline** — automatically detects folder structure from document content, keyword matching, and ML-based prediction
- **Bilingual Support** — processes documents in both English and Arabic (with OCR fallback for scanned PDFs)
- **Batch Processing** — process a single file or an entire folder recursively
- **AI-Assisted Folder Prediction** — TF-IDF cosine similarity + RapidFuzz fuzzy matching for ambiguous documents
- **Safe File Operations** — byte-size verification before source deletion, MD5 duplicate detection
- **Audit Log** — full structured JSON audit trail for every file processed
- **Folder Watcher** — optional `watcher.py` daemon for automated hot-folder monitoring
- **Web UI** — browser-based interface for uploading files and reviewing results

---

## Processing Pipeline

```
Upload / Folder Scan
        ↓
Text Extraction  (pdfplumber + Tesseract OCR fallback)
        ↓
Language Detection  (English / Arabic)
        ↓
Keyword Matching + Confidence Scoring
        ↓
   ┌────────────────────────────────────────────────────┐
   │  Scenario 1: Explicit folder structure on page 1   │ → Auto-process
   │  Scenario 2: Keyword / language-based match        │ → Auto-process
   │  Scenario 3: Ambiguous / low confidence            │
   │      ├── score ≥ threshold  → Intelligent Prediction → Auto-process
   │      ├── score < threshold  → Uncategorized / Review Required
   │      └── no candidates     → Under Review
   └────────────────────────────────────────────────────┘
        ↓
Rename  →  Create destination folder  →  Move file  →  Delete source (optional)
```

---

## Project Structure

```
pdf_processor/
├── app.py                    # Flask application — main entry point
├── watcher.py                # Hot-folder watcher daemon (watchdog)
├── start_watcher.bat         # Windows launcher for the watcher
├── requirements.txt
├── data/
│   └── keywords.json         # Keyword database (entities, doc types, departments, folder structure)
├── static/
│   └── css/
│       └── style.css
├── templates/
│   └── index.html            # Web UI
├── utils/
│   ├── classifier.py         # Document classification logic
│   ├── file_handler.py       # Safe copy + delete helpers
│   ├── filename_parser.py    # Filename construction rules
│   ├── folder_manager.py     # Destination path builder
│   ├── keyword_db.py         # Keyword database loader / cache
│   ├── language_detector.py  # English / Arabic detection
│   └── pdf_reader.py         # Text extraction (pdfplumber + OCR)
├── logs/
│   ├── operations.log        # Rolling operation log
│   └── audit_log.json        # Structured audit records (CR3)
└── uploads/                  # Temporary directory for uploaded files
```

---

## Requirements

**Python 3.9+** is required.

| Package | Purpose | Required? |
|---|---|---|
| `flask >= 3.0.0` | Web framework | ✅ Core |
| `pdfplumber >= 0.11.0` | PDF text extraction | ✅ Core |
| `watchdog >= 3.0.0` | Hot-folder watcher | ✅ Core |
| `scikit-learn >= 1.3.0` | TF-IDF prediction (CR2) | ⭐ Recommended |
| `rapidfuzz >= 3.0.0` | Fuzzy folder matching (CR2) | ⭐ Recommended |
| `pytesseract >= 0.3.10` | OCR for scanned PDFs | ⭐ Recommended |
| `Pillow >= 10.0.0` | Image processing for OCR | ⭐ Recommended |

**OCR also requires:**
- [Tesseract](https://github.com/tesseract-ocr/tesseract) binary on your `PATH`
- Arabic language pack (`tesseract-ocr-ara`) for Arabic documents

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-username/pdf-file-processor.git
cd pdf-file-processor/pdf_processor

# 2. Create and activate a virtual environment (recommended)
python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. (Optional) Install Tesseract for OCR support
#    https://github.com/tesseract-ocr/tesseract
#    Also install the Arabic language pack if needed

# 5. Run the application
python app.py
```

The server starts at `http://127.0.0.1:5000`.

---

## Configuration

### Destination Root Folder

Set where processed files are filed via the UI or the API:

```bash
curl -X POST http://localhost:5000/destination \
  -H "Content-Type: application/json" \
  -d '{"destination": "C:/CEO Office/DMS"}'
```

The setting is persisted in `destination_config.json`.

### Keyword Database (`data/keywords.json`)

The keyword database drives classification. It contains:

- **entities** — company entities (e.g. Corporate, Maritime, TRBA, TTC)
- **document_type_indicators** — keywords mapped to document types (Contract, Invoice, Minutes, etc.)
- **department_indicators** — keywords mapped to departments
- **folder_structure** — the target DMS folder hierarchy
- **keywords** — English keywords and Arabic translations

After editing `keywords.json`, reload without restarting:

```bash
curl -X POST http://localhost:5000/keywords/reload
```

### Prediction Confidence Threshold

Adjust how aggressively the intelligent prediction accepts uncertain matches by editing `PREDICTION_CONFIDENCE_THRESHOLD` in `app.py` (default: `0.35`, range: `0.0 – 1.0`).

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Web UI |
| `POST` | `/process` | Process a single PDF upload |
| `POST` | `/process-batch` | Batch: folder path (JSON) or multiple file uploads (multipart) |
| `POST` | `/delete-processed-files` | Delete source files after a browser-upload batch |
| `GET` | `/destination` | Get current destination root path |
| `POST` | `/destination` | Update destination root path |
| `GET` | `/logs?n=100` | Return last N lines of the operations log |
| `GET` | `/audit-log?n=100` | Return last N structured audit records |
| `GET` | `/keywords` | Return keyword database summary |
| `POST` | `/keywords/reload` | Reload keyword database from disk |

### Single File — `POST /process`

```bash
curl -X POST http://localhost:5000/process \
  -F "file=@/path/to/document.pdf"
```

**Response:**
```json
{
  "success": true,
  "under_review": false,
  "prediction_mode": "standard",
  "original_filename": "document.pdf",
  "renamed_filename": "2026_Corporate_Contract_ACME.pdf",
  "destination": "C:/DMS/CEO Office/2026/Corporate/Contracts/2026_Corporate_Contract_ACME.pdf",
  "folder_path": "CEO Office/2026/Corporate/Contracts",
  "scenario": 1,
  "language": "english",
  "confidence": 0.91,
  "matched_keywords": ["contract", "corporate"],
  "timestamp": "2026-06-11 10:00:00",
  "metadata": {
    "entity": "Corporate",
    "department": null,
    "client": "ACME",
    "doc_type": "Contract",
    "date": "April 2026",
    "year": "2026"
  }
}
```

### Batch — `POST /process-batch` (folder path)

```bash
curl -X POST http://localhost:5000/process-batch \
  -H "Content-Type: application/json" \
  -d '{"folder_path": "C:/Inbox/PDFs", "delete_source": true}'
```

### Batch — `POST /process-batch` (file uploads)

```bash
curl -X POST http://localhost:5000/process-batch \
  -F "files[]=@doc1.pdf" \
  -F "files[]=@doc2.pdf"
```

---

## Folder Watcher

To automatically process PDFs dropped into a source folder, run the watcher daemon:

```bash
python watcher.py
```

Or on Windows, double-click `start_watcher.bat`.

---

## Audit Log

Every processed file generates an audit record in `logs/audit_log.json`:

```json
{
  "id": "audit_20260611100000123456",
  "timestamp": "2026-06-11 10:00:00",
  "original_path": "C:/Inbox/contract.pdf",
  "destination_path": "C:/DMS/CEO Office/2026/Corporate/Contracts/...",
  "renamed_filename": "2026_Corporate_Contract_ACME.pdf",
  "move_status": "success",
  "deletion_status": "deleted",
  "confidence_score": 0.91,
  "prediction_mode": "standard",
  "scenario": 1,
  "entity": "Corporate",
  "doc_type": "Contract",
  "department": null,
  "language": "english"
}
```

`move_status` values: `success` | `skipped_duplicate` | `failed` | `verification_failed`  
`deletion_status` values: `deleted` | `retained` | `not_found` | `delete_failed`

---

## Duplicate Detection

Before copying, the system checks for duplicates in the destination folder using:

1. **Filename collision** — if the name exists, computes MD5 hash of both files to confirm whether content is identical
2. **Content hash scan** — scans all PDFs in the destination folder for matching MD5 hash (catches renamed duplicates)

Exact duplicates are skipped; filename conflicts result in a `_(1)`, `_(2)`, … suffix.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.9+, Flask 3 |
| PDF Extraction | pdfplumber |
| OCR | Tesseract + pytesseract + Pillow |
| ML Prediction | scikit-learn (TF-IDF), RapidFuzz |
| File Watching | watchdog |
| Frontend | HTML / CSS / Vanilla JS |

---

## Conclusion

The PDF File Processor transforms a manual, repetitive filing process into a fully automated pipeline. By combining rule-based keyword matching with ML-assisted folder prediction and OCR for scanned documents, it handles the full spectrum of real-world document quality — from clean digital PDFs to handwritten-signed scans in Arabic.

The three-scenario classification model ensures that high-confidence documents are filed immediately without human intervention, while uncertain documents are safely quarantined for review rather than misfiled. The audit log and duplicate detection provide the traceability and data integrity required in a professional document management environment.

The system is designed to be extended: new entities, document types, and departments are added simply by updating `keywords.json`, with no code changes required. The same architecture can be adapted to any organization with a structured DMS folder hierarchy.

---

