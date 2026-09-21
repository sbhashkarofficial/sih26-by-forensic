# Technical System Architecture & Operational Documentation

## 1. Executive Summary & Pipeline Overview

The **AI-Powered Document Screening Platform** is an enterprise-grade hybrid computer vision and cryptographic ledger system engineered for border control checkpoints. It processes multi-modal inputs—scanning physical travel documents alongside live passenger camera captures—to automate layout-agnostic segmentation, optical character recognition (OCR), multi-layered forgery detection, and biometric 1:1 face verification. The platform ensures zero-trust data integrity by coupling high-throughput neural inference with an immutable, permissioned consortium blockchain ledger.

---

## 2. Stage-by-Stage Module Specifications

### Stage 0: Image Ingestion

The frontend interface ingests two discrete high-resolution streams simultaneously:

* **Document Image:** Physical travel document captured by a webcam/high-resolution camera.
* **Person’s Photo:** Live camera capture of the traveler at the inspection booth.

#### Dual-Path Duplicate Routing

* **Original Image:** Forwarded directly to the Hash Engine.
* **Copy of Original Image:** Routed to the pre-processing pipeline.

---

### Stage 1: Image Preprocessing

* **Background Segmentation:** BireRefNet (a deep learning framework) is used to segregate the document substrate from background artifacts.
* **Flattening & Perspective Warping:** Detects the contour of the document substrate (simplifying it to 4 or 6 polygonal edges) and applies an affine/perspective transformation matrix to unwarp and deskew the document canvas.

---

### Stage 2: YOLO Instance Segmentation Engine

The pre-processed image passes through a fine-tuned YOLO Segmentation model (`yolo11n-seg`) to extract **7 Universal Visual Classes**:

| Class ID | Macro-Class Name | Target                                                        | Target Module                  |
|:--------:|:---------------- |:------------------------------------------------------------- |:------------------------------ |
| `0`      | `doc_header`     | Issuing state header & document title text                    | Hierarchical Router            |
| `1`      | `main_photo`     | Primary facial portrait                                       | Face Verification Engine       |
| `2`      | `ghost_photo`    | Secondary laser-engraved / UV / shadow portrait               | Photo-Substitution Engine      |
| `3`      | `mrz_zone`       | Machine Readable Zone (2 or 3 lines of monospaced text)       | Checksum MRZ                   |
| `4`      | `signature`      | Holder's handwritten or engraved signature                    | Stamp, Seal & Signature Module |
| `5`      | `stamp_seal`     | Official ink stamps, seals, or visa watermarks                | Stamp, Seal & Signature Module |
| `6`      | `fields_block`   | Unified bounding block encompassing key-value personal fields | PaddleOCR Engine               |

---

### Stage 3: Hierarchical Classification

* **Level 1 Classification (Document Type):** Categorizes the document layout into one of four macro-categories:
  * `passport`
  * `visa`
  * `national_id`
  * `dl` (Driving License)
* **Level 2 Classification (Country & Schema Resolution):** Parses country identifiers (e.g., characters 3–5 of MRZ Line 1 or header OCR) to execute *Identify country* and load *what fields to be extracted* from the local Schema Registry.

---

### Stage 4: Verification and Checks Engine

The system executes three parallel analysis tracks on the extracted crops and pre-processed image canvas:

#### Track A: Hardcoded Logic Checks (Deterministic Engine)

* **MRZ Checksum Algorithm:** Executes ICAO Doc 9303 Modulus 10 calculation with a repeating $7\text{-}3\text{-}1$ weighting sequence on:
  * Passport Number Check Digit
  * Date of Birth Check Digit
  * Date of Expiry Check Digit
  * Personal Number Check Digit
  * Composite Check Digit
* **Temporal & Date Logic Checks:**
  * Expiry Logic: $\text{Current Date} < \text{Date of Expiry}$
  * Issue Logic: $\text{Date of Birth} < \text{Date of Issue} < \text{Date of Expiry}$
  * Passport Lifespan: $(\text{Date of Expiry} - \text{Date of Issue}) \le 10\text{ years}$
* **String & Standard Format Checks:** Validates 3-letter issuing state codes against ISO 3166-1 alpha-3 standards and verifies character lengths (e.g., exactly 44 characters per line for TD3 passports).

#### Track B: ANN / DNN-Based Verifications

* **Face Verification Engine:**
  * Extracts deep facial embeddings using a facial recognition network (e.g., `InsightFace` / `DeepFace`).
  * Computes 1:1 cosine similarity across three vector pairs:
    $$\text{Embedding}(\text{main\_photo}) \leftrightarrow \text{Embedding}(\text{Person's Photo [Live]})$$
    $$\text{Embedding}(\text{ghost\_photo}) \leftrightarrow \text{Embedding}(\text{Person's Photo [Live]})$$
    $$\text{Embedding}(\text{main\_photo}) \leftrightarrow \text{Embedding}(\text{ghost\_photo}) \quad \text{(Catches photo-substitution attacks)}$$
* **PaddleOCR Field-Value Extraction:**
  * Processes the pre-processed document for key-value pairs as per the schema resolved in Stage 3.
  * Applies spatial bounding box clustering to pair keys and values dynamically without cross-field contamination.
* **Stamp, Seal, and Signature Verification:**
  * Analyzes cropped `stamp_seal` regions via edge-density and color-histogram classifiers to distinguish physical ink stamps from printed digital graphics.
  * Evaluates `signature` regions for structural background editing artifacts.
* **MRZ and VIZ Cross-Matching:** Compares extracted Visual Inspection Zone (VIZ) text fields against decoded MRZ fields (e.g., confirming $\text{VIZ\_Surname} == \text{MRZ\_Surname}$).

#### Track C: Error Level Analysis (ELA)

* Re-compresses the image canvas and passes pixel difference heatmaps through a CNN to isolate compression anomalies, spliced text, and localized physical/digital alterations.

---

### Stage 5: Verification Pipeline & Dynamic Schemas

The extracted data and validation statuses are validated against the appropriate Dynamic Document Schema matching the document type.

Each schema dictates mandatory feature requirements:

* `passport_v1`: Standard non-biometric layout (Requires: `main_photo`, `mrz_zone`, `fields_block`).
* `visa_v1`: Entry visa layout (Requires: `fields_block`, `stamp_seal`; Optional: `mrz_zone`).

---

## 3. Cryptographic Ledger & Cybersecurity Infrastructure

### 1. Ingestion Hash Generation

Before passing the image to computer vision models, the system computes a SHA-256 cryptographic hash of the raw image bytes:

$$\text{Document\_Hash} = \text{SHA256}(\text{Raw\_Image\_Bytes})$$

### 2. Pre-Execution Ledger Query

The system queries the shared ledger using $\text{Document\_Hash}$:

* **Duplicate Detection:** If the hash exists and was previously marked `PASSED`, the system skips compute-heavy OCR/YOLO modules and routes directly to 1:1 Live Face Matching.
* **Revocation Check:** If the hash matches a record on the Stolen / Revoked Document Smart Contract, the transaction is immediately terminated and flagged as **CRITICAL HIGH RISK**.

### 3. Blockchain Transaction Logging

Once verification completes, the system commits an immutable transaction record to the ledger via smart contract execution.

---

## 4. Data Schemas, Scores, and Payload Specifications

### A. Composite Risk Score Engine

The final output synthesizes scores into a normalized scale ($0 \text{ to } 100$):

$$\text{Final\_Risk\_Score} = (w_1 \times S_{\text{mrz}}) + (w_2 \times S_{\text{face}}) + (w_3 \times S_{\text{forensics}}) + (w_4 \times S_{\text{logic}})$$

| Score Category             | Sub-Metrics Evaluated                                                                      | Weight ($w_i$) | Pass Threshold    |
|:-------------------------- |:------------------------------------------------------------------------------------------ |:--------------:|:-----------------:|
| `mrz_verification_score`   | Modulus 10 check digits, character length, B900 IR drop-out ratio                          | $0.25$         | $\ge 95\%$        |
| `face_verification_score`  | Cosine similarity: (Main vs. Live), (Ghost vs. Live), (Main vs. Ghost)                     | $0.35$         | $\ge 85\%$        |
| `other_scores` (Forensics) | ELA discrepancy score, stamp/seal ink density, signature background continuity             | $0.20$         | $\ge 90\%$        |
| `other_scores` (Logic)     | Date validity ($\text{Expiry} > \text{Current}$), VIZ vs. MRZ text match, ISO country code | $0.20$         | $100\%$ (Boolean) |

#### Risk Action Matrix

* **Score $0 - 15$ (LOW RISK):** Automated clearance (e-Gate open; total processing time $< 3\text{ seconds}$).
* **Score $16 - 50$ (MEDIUM RISK):** Flagged for primary visual check by SSB officer (e.g., minor OCR read noise).
* **Score $51 - 100$ (HIGH RISK):** Document rejected; automatic alert sent to secondary security booth.

---

### B. JSON Payload Specifications

#### 1. Internal Verification State Object (Python Engine Output)

```json

```
