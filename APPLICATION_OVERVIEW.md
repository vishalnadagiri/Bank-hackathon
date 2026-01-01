# Banking KYC Document Upload & Verification Application — Overview

## 1) What this application does
This project is a Streamlit-based KYC (Know Your Customer) portal that supports:

- Customer **registration/login** and session-based navigation
- Secure **document upload** (ID proof, address proof, photo)
- **OCR + field extraction** (with image preprocessing and field-level confidence)
- **Validation and verification** of extracted fields
- **KYC status tracking** with progress/verification score
- Admin utilities (where enabled) such as reports, training utilities, and audit views

The goal is to provide a bank-style KYC workflow where customers upload documents and the system automatically extracts and verifies key details with traceable confidence.

---

## 2) High-level workflow (end-to-end)

### 2.1 Authentication & session
1. User lands on the app (`main.py`).
2. User registers / logs in via `pages/authentication.py`.
3. App stores session state in `st.session_state` (authenticated user, customer_id, customer_data, admin mode).

### 2.2 Document upload
1. User navigates to upload screens (e.g., customer upload pages).
2. Uploaded files are stored on disk under `uploads/` (runtime folder; should not be committed).
3. A record is written to the SQLite database `data/kyc_bank.db` under the `documents` table (paths + type + timestamps).

### 2.3 Extraction + verification
1. The extraction pipeline loads the image.
2. Image preprocessing improves OCR readability (denoise, thresholding, deskew, ROI operations).
3. OCR runs and produces text + confidence.
4. Post-processing normalizes text and extracts fields using patterns/rules.
5. Field validation assigns pass/fail + confidence thresholds.
6. Verification results are persisted (document row updates + verification logs + `kyc_status` updates).

### 2.4 Dashboard / status tracking
1. User navigates to `pages/dashboard.py`.
2. Dashboard pulls KYC status and document states.
3. UI displays an overall KYC status (completion % and verification score) plus per-document status cards.

---

## 3) Extraction strategy (how we extract fields)
The extraction approach is intentionally **hybrid**:

- **Classical OCR** (Tesseract via `pytesseract`) for broad text extraction
- **Document-type heuristics** and regex-based parsing for key fields
- **Image preprocessing** tuned per field type to improve OCR confidence
- **Validation rules** to reduce false positives and quantify quality
- **Fallback mode** to recover data when strict thresholds fail

### 3.1 Core modules

#### Image preprocessing
- Module: `services/image_preprocessing_service.py`
- Responsibilities:
  - Normalize input images
  - Apply OCR-friendly transforms (thresholding, contrast, denoise)
  - Support “field type” specific preprocessing (e.g., name vs. id_number vs. DOB)

#### OCR + parsing
You’ll find multiple extractors (to support different document types and maturity levels):

- `services/document_extractor.py`
  - General OCR + parsing utilities
  - Pattern-based field extraction

- `services/ml_document_extractor.py`
  - ML-assisted extraction orchestration (where used)

- `services/production_ml_extractor.py`
  - A production-oriented wrapper: structured output + metadata

- `services/enhanced_ml_extractor.py`
  - Adds confidence-aware decisions and a **fallback mode** (lower thresholds temporarily to salvage partial extraction)

- `services/trocr_aadhaar_extractor.py`
  - Aadhaar-focused extraction logic using regex + positional heuristics
  - Designed to reduce header/footer noise and prioritize likely Aadhaar zones

> Note: The system is built so you can route by `document_type` and gradually upgrade extraction per document.

### 3.2 Field validation and confidence
- Module: `services/field_validation_service.py`
- Responsibilities:
  - Define confidence thresholds per field
  - Validate formats (e.g., ID number, date-of-birth patterns)
  - Produce structured results such as:
    - `extraction_success`
    - `confidence`
    - quality metrics / issues

### 3.3 Verification and business rules
- Module: `services/verification_service.py`
- Responsibilities:
  - Combine extraction output + validation into business verification states
  - Generate overall KYC scoring and completion logic

### 3.4 Training / improvements (optional path)
There is a training workflow (annotation + sample storage) intended for iterative improvement:

- `pages/document_training.py`
- `services/document_trainer.py`
- `components/*annotator.py`

This supports collecting labeled samples and improving extraction rules/models over time.

---

## 4) Data model (SQLite)
Database module: `config/database.py`

SQLite file: `data/kyc_bank.db`

Core tables (representative):
- `customers` — customer identity and auth
- `documents` — uploaded document records
- `otp_records` — OTP log/records (if enabled)
- `audit_logs` — audit trail
- `kyc_status` — overall per-customer KYC status
- `aadhaar_kyc` — Aadhaar verification attempts and results

Design intent:
- Keep document uploads as filesystem objects; DB stores metadata + pointer path.
- Persist extraction output (JSON) and verification scores for traceability.

---

## 5) UI / Pages
Entry point: `main.py` (Streamlit navigation)

Key pages:
- `pages/authentication.py` — login/register
- `pages/dashboard.py` — KYC status overview + document status + customer profile
- `pages/document_upload.py` / `pages/user_document_upload.py` — document uploads
- `pages/ml_extractor_page.py` — extraction UI (ML-assisted)
- `pages/ocr_debugger_page.py` — OCR debugging UI
- `pages/reports.py` — reporting view
- `pages/admin_panel.py` — admin tools

---

## 6) Tech stack

### Frontend / App framework
- **Streamlit** (Python) for UI, page routing, session state, and components

### Backend / services
- **Python** services under `services/`
- **SQLite** for persistence (`sqlite3`), with schema management in `config/database.py`

### Document processing & extraction
- **OpenCV** (`cv2`) for image preprocessing and transformations
- **Tesseract OCR** via **pytesseract** (OCR text extraction + confidence)
- **Regex / rule-based parsing** for field extraction and normalization

### Model-based extraction (where enabled)
- Aadhaar-specific extraction utilities in `trocr_aadhaar_extractor.py`
- ML extractor wrappers (`ml_document_extractor.py`, `production_ml_extractor.py`, `enhanced_ml_extractor.py`)

### Observability / debugging
- Python `logging`
- OCR debugger page for inspecting intermediate outputs

---

## 7) Deployment strategy (practical)

### Runtime folders
These are generated at runtime and should not be committed:
- `uploads/`
- `data/training/`
- `data/ocr_debug/`

### Configuration
- Use `.env` for secrets (do not commit); see `.env.example`.

### Start command
Typical:
```bash
streamlit run main.py --server.address 0.0.0.0 --server.port 8501
```
(Or platform-provided `$PORT`.)

---

## 8) Operational strategy (how to keep it production-grade)

- **PII hygiene**: never commit `uploads/` or real customer documents
- **DB strategy**: keep schema in code; DB file is runtime
- **Quality gates**: use confidence thresholds and field validation to avoid false positives
- **Fallbacks**: allow partial extraction with clear metadata (`fallback_mode`) for recoverability
- **Auditability**: store verification decisions and timestamps

---

## 9) Where to extend next
- Add document-type routing (Aadhaar / PAN / passport etc.) to choose the best extractor
- Introduce stronger layout-aware OCR and structured extraction for complex documents
- Add automated tests for parsing/validation rules
- Add background jobs for extraction to avoid blocking UI on large images

