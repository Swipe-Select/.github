# Qwen2.5-3B Tuning & In-Place PDF Template Editor

This document captures the changes implemented across **Swipe-Select-ML**, **Swipe-Select-Backend**, and **Swipe-Select-Frontend-Web** to fix local `Qwen2.5-3B-Instruct.Q4_K_M.gguf` resume output quality and preserve the user's uploaded PDF design during resume tailoring.

It is intended as quick context for both developers pulling these updates and AI agents working on top of them later.

---

## 1. Branches & Teammate Setup Notes

| Repo | Feature Branch | Summary of Changes |
|---|---|---|
| `Swipe-Select-ML` | `qwen-tuning` | Inference & prompt tuning, anti-hallucination rules, Markdown cleanup, and PyMuPDF in-place PDF inspection/editing endpoints |
| `Swipe-Select-Backend` | `pdf-editor` | Multi-part ML client helper + in-place PDF tailoring orchestration (`local` Qwen & `ai` Gemini) |
| `Swipe-Select-Frontend-Web` | `UI-Polishing` | Fixed ATS paper heading parser, corrected engine labels, defaulted to Local Qwen2.5-3B, and added edited PDF preview + diff viewer |

### Do Model Files Need Re-Uploading?
**No.** No model weights (`Qwen2.5-3B-Instruct.Q4_K_M.gguf` or `models/fake_job_detection/model.safetensors`) were retrained, re-quantized, or altered on disk. All behavior changes are in Python/TypeScript code.

### Steps Required After Pulling
1. Pull the updated branches in `Swipe-Select-ML`, `Swipe-Select-Backend`, and `Swipe-Select-Frontend-Web`.
2. Inside `Swipe-Select-ML`'s Python virtual environment, install the two new Python dependencies added to `requirements.txt`:
   ```bash
   pip install -r requirements.txt
   ```
   Added packages:
   - `pymupdf>=1.24.0`
   - `python-multipart>=0.0.9`
3. Ensure the local Qwen GGUF file is named cleanly as:
   `Swipe-Select-ML/models/resume_generation/Qwen2.5-3B-Instruct.Q4_K_M.gguf`
4. Restart the ML server (`uvicorn main:app --host 0.0.0.0 --port 8000`), Backend (`npm run dev`), and Web Frontend (`npm run dev`).

---

## 2. Part 1 — Local Qwen2.5-3B Output & Grounding Fixes

### Root Causes Found
1. **Wall-of-text & missing punctuation:** `Swipe-Select-ML/main.py` previously used `repeat_penalty=1.22`, `presence_penalty=0.5`, `frequency_penalty=0.5`, and `max_tokens=600`. On Qwen2.5-3B Q4_K_M, those high penalties suppressed repeated formatting characters (`\n`, `-`, `#`, hyphens) and truncated generation mid-output.
2. **Fabricated experience (`SolutionInn` / fake metrics):** The Markdown template shown in `_build_resume_prompt` always included a `## PROFESSIONAL EXPERIENCE` slot, nudging the 3B model to invent the target company/title as past work when a candidate's resume only listed Projects and Education.
3. **Frontend Markdown splitting bug:** `ResumePaperTemplate.tsx` previously split sections on `\n#{2,3}\s+`, which broke apart inner `###` subheadings and dumped unmatched sections into the single summary block.
4. **Mislabeled UI engine toggle:** `TailorResumeModal.tsx` previously labeled the cloud Gemini option (`engine === "ai"`) as `"Fine-Tuned Qwen2.5-3B Model"`, making users think Qwen produced the Gemini layout while local Qwen was hidden under `"Local Edge Model (GGUF)"`.

### Implemented Fixes
- **`Swipe-Select-ML/main.py`**:
  - Updated `/generate/resume` sampling parameters to `max_tokens=1500`, `temperature=0.35`, `top_p=0.9`, `repeat_penalty=1.06` (removed `frequency_penalty` and `presence_penalty`).
  - Strengthened `RESUME_SYSTEM_PROMPT` to forbid fabricating employers/dates/metrics or copying job-description sentences verbatim as fake past work.
  - Updated `_build_resume_prompt()` to detect whether the input contains prior work history (`has_work_history`) and omit the `## PROFESSIONAL EXPERIENCE` template block when none exists.
  - Added `_normalize_markdown_output()` to normalize collapsed section headers and deduplicate stray repeated tokens.
- **`Swipe-Select-Frontend-Web/src/components/ResumePaperTemplate.tsx`**:
  - Updated section splitting to split on top-level `\n##\s+` when `##` headers exist (falling back to `\n#{2,3}\s+` only if no `##` exists).
  - Updated Experience, Projects, and Education parsers to match both `### ...` subheadings and `**...**` bold headers.

---

## 3. Part 2 — Layout-Preserving In-Place PDF Editor

Instead of discarding multi-column Canva/Figma resume designs and rebuilding everything into a plain ATS text box, the system now extracts editable text blocks from the uploaded PDF and replaces text directly inside the original PDF bounding boxes while keeping photos, colored banners, sidebars, icons, contact info, and education intact.

### Architecture Overview

#### A. Python PDF Inspector & Editor (`Swipe-Select-ML/pdf_editor.py`)
- **`inspect_pdf(pdf_bytes)`**:
  - Uses `pymupdf` (`import pymupdf as fitz`) to check for a selectable text layer (`has_text_layer`).
  - Classifies each block (`_classify_section_hint`) by matching its vertical & horizontal column position against the nearest section header above (`PROTECTED_SECTION_KEYWORDS` vs `EDITABLE_SECTION_KEYWORDS`) and checking `_is_protected_content()`.
  - **Protected (never modified):** Name banner, large title text, contact info (emails, phone numbers, URLs), dates, education/coursework/certifications, links, and section headers.
  - **Editable (`B0`, `B1`, ...):** Paragraph/bullet content blocks under Summary/Profile, Skills, Experience/Work History, and Projects — annotated with `max_chars` (capacity estimate from box area/font size), original font size, text color (`_int_color_to_rgb`), background fill (`_detect_bg_color`), and alignment.
- **`apply_pdf_replacements(pdf_bytes, replacements, inspected)`**:
  - Skips whitespace-only or unchanged blocks.
  - Uses PyMuPDF redaction with transparent fill (`fill=None`) and zero media/vector removal (`images=fitz.PDF_REDACT_IMAGE_NONE`, `graphics=fitz.PDF_REDACT_LINE_ART_NONE`) so dark sidebars, colored headers, vector icons, and profile photos remain untouched.
  - Re-inserts replacement text via `page.insert_textbox(...)` using `_fit_font_size()` (auto-scaling font size slightly downward if needed so text never clips/overflows the box) and returns `(modified_bytes, changes)`.

#### B. ML FastAPI Endpoints (`Swipe-Select-ML/main.py`)
1. `POST /pdf/inspect` (`multipart/form-data` `file`): returns `has_text_layer`, `full_text`, `page_count`, and `editable_blocks`.
2. `POST /pdf/apply` (`multipart/form-data` `file` + `replacements_json`): applies `{block_id: text}` edits and returns `tailored_pdf_base64`, `blocks_modified`, `changes`, and `tailored_text`.
3. `POST /tailor/pdf` (`multipart/form-data` `file`, `job_title`, `job_company`, `job_description`):
   - Runs `inspect_pdf`.
   - Prompts local `Qwen2.5-3B` with `PDF_TAILOR_SYSTEM_PROMPT` (`temperature=0.45`, `response_format={"type": "json_object"}`) to actively rewrite editable blocks within each block's `max_chars` budget without hallucinating facts.
   - Calls `apply_pdf_replacements` and returns `has_text_layer`, `tailored_pdf_base64`, `blocks_found`, `blocks_modified`, `changes`, and `tailored_text`.

#### C. Backend Orchestration (`Swipe-Select-Backend`)
- **`src/services/resume.service.ts`**: added `PdfEditableBlock` interface and `tailorPdfBlocksWithAI()` (asks Gemini to return JSON `{"replacements": {"B0": "...", ...}}` respecting each block's `max_chars` and truthfulness rules).
- **`src/controllers/ml.controller.ts`**:
  - Added `mlFormFetch()` helper for forwarding `multipart/form-data` from Node `fetch` cleanly to the FastAPI ML service.
  - Updated `tailorResumeFromUpload`: when an uploaded PDF exists (`section === 'tailored_resume'`):
    - If `engine === 'local'`: calls ML `POST /tailor/pdf`.
    - If `engine === 'ai'` (or any non-`local` engine): calls ML `POST /pdf/inspect` -> `tailorPdfBlocksWithAI()` -> ML `POST /pdf/apply` (falling back to regular Markdown tailoring if inspection returns `has_text_layer: false` or an error occurs).
  - Returns existing Markdown response fields plus `tailoredPdfBase64`, `pdfEdited`, `blocksModified`, and `pdfChanges`.

#### D. Web Frontend (`Swipe-Select-Frontend-Web`)
- **`src/api/ml.ts`**: added `PdfBlockChange` and added optional `tailoredPdfBase64`, `pdfEdited`, `blocksModified`, `pdfChanges` fields to `ResumeGenResult`.
- **`src/components/TailoredPdfPreview.tsx`**:
  - `TailoredPdfPreview`: renders `tailoredPdfBase64` via `URL.createObjectURL` inside an `<iframe>`, with **Download PDF**, **Save as Base**, a collapsible **What Changed** block diff panel, and a toggle to **ATS Paper View**.
  - `TailoredResultPane`: shared wrapper used by both `TailorResumeModal.tsx` and `AiToolsPage.tsx` that shows `TailoredPdfPreview` by default when `pdfEdited && tailoredPdfBase64` is returned, or falls back to `ResumePaperTemplate` for scanned PDFs / plain-text resumes.
- **`src/components/TailorResumeModal.tsx` & `src/pages/AiToolsPage.tsx`**:
  - Default tailoring engine set to `"local"` (**Local Qwen2.5-3B (GGUF)**).
  - Fixed engine card copy so `"Local Qwen2.5-3B Model (GGUF)"` and `"Cloud Gemini AI"` are accurately labeled.
  - Split (`BEFORE` / `AFTER`) and standalone output views now render the real edited PDF via `TailoredResultPane`.

---

## 4. Modified & Added Files Inventory

- `Swipe-Select-ML/requirements.txt`
- `Swipe-Select-ML/pdf_editor.py` *(new)*
- `Swipe-Select-ML/main.py`
- `Swipe-Select-Backend/src/services/resume.service.ts`
- `Swipe-Select-Backend/src/controllers/ml.controller.ts`
- `Swipe-Select-Frontend-Web/src/api/ml.ts`
- `Swipe-Select-Frontend-Web/src/components/ResumePaperTemplate.tsx`
- `Swipe-Select-Frontend-Web/src/components/TailoredPdfPreview.tsx` *(new)*
- `Swipe-Select-Frontend-Web/src/components/TailorResumeModal.tsx`
- `Swipe-Select-Frontend-Web/src/pages/AiToolsPage.tsx`

