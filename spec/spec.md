# Product Specification: Offline AI Study App

**Status:** Draft for approval; no application implementation is authorized yet.
**Source of truth:** `Architecture/Complete Arch.docx` and the decisions approved 2026-10-11.
**MVP platform:** Android only, English-first. This document describes user-visible behavior and measurable outcomes; technical choices belong in `spec/plan.md`.

## 1. Problem And Target User

Students have their own PDFs and notes but need a private, reliable way to ask questions, find evidence, and study without sending their learning material to a cloud AI service. The MVP lets a learner use downloaded on-device models to ask questions against their own PDFs and receive concise answers with page citations. English is the first supported answer language; other languages are enabled only after their model-specific evaluation gates pass.

## 2. User Stories And Acceptance Criteria

### US-01: First launch and model setup

As a learner, I want to know whether my phone is supported and install an approved local model so I can study offline.

- **AC-01.1** Given a phone running Android 10+ with arm64 and at least 4 GB RAM, when the app starts, then it offers the supported setup flow; otherwise it clearly explains that the device is unsupported and does not offer a cloud fallback.
- **AC-01.2** Given network connectivity, when the app refreshes its model catalog, then it accepts only a valid signed catalog, caches the last verified copy, and presents only models that have passed technical and license approval.
- **AC-01.3** Given a catalog model, when the learner reviews it, then the app displays the model name, download size, supported languages, minimum RAM, and license/attribution information before download.
- **AC-01.4** Given the learner requests a model download, when storage is checked, then the app shows required and available space before starting; if insufficient, it preserves existing data and lets the learner free space or cancel.
- **AC-01.5** Given the learner starts a model download, when Wi-Fi is unavailable or the app/process is interrupted, then the download does not start over unnecessarily, can resume when permitted, and is verified against the catalog SHA-256 before it becomes selectable; a checksum mismatch deletes the invalid partial file and reports failure.
- **AC-01.6** Given a verified model is already downloaded, when the phone has no network connection, then the learner can open the app and use the local study flow.
- **AC-01.7** Given the learner removes a model, when removal completes, then its private model file is deleted and remaining local study data is unchanged.

### US-02: Add and index a PDF

As a learner, I want to add my own PDF and see when it is ready so I can ask questions about its contents.

- **AC-02.1** Given a PDF larger than 100 MB or longer than 1,000 pages, when the learner attempts import, then the app rejects it with the applicable limit and preserves existing data.
- **AC-02.2** Given a PDF within the configured limits, when import is requested, then the app checks required and available storage before copying or starting indexing; if storage is insufficient, it reports both values and lets the learner free space or cancel without deleting existing data.
- **AC-02.3** Given a valid text PDF, when indexing runs, then page numbers are preserved, content is cleaned, and text is chunked at a target of 350 tokens with 60-token overlap, preferring headings and paragraphs before sentences.
- **AC-02.4** Given an indexing job is running, when the learner leaves the screen or the process is interrupted, then progress is visible where appropriate and committed batches remain consistent; work resumes from the last committed batch without duplicate chunks.
- **AC-02.5** Given an indexing job, when all content, embeddings, and keyword-index updates have committed, then the document becomes ready; until then it is not searchable as a ready document.
- **AC-02.6** Given malformed, encrypted/password-protected, scanned, or text-free PDFs, when the learner imports them, then the app reports a specific actionable error or explains that no searchable text was found; OCR is not attempted in the MVP.
- **AC-02.7** Given any PDF, when it is imported, then the original file and all extracted/indexed content remain in app-private storage and are never transmitted.

### US-03: Ask a question and receive a cited answer

As a learner, I want answers grounded in my selected notes so I can trust and verify them.

- **AC-03.1** Given one or more ready documents in the selected scope, when the learner asks a question, then retrieval searches semantic and keyword indexes, combines ranked results, and supplies at most the best four chunks within a 1,500-token evidence budget to the answer process.
- **AC-03.2** Given a follow-up question, when the answer process prepares retrieval, then it may use the last two conversation turns to resolve references such as “explain that again,” without sending those turns off-device.
- **AC-03.3** Given relevant evidence is available, when an answer is generated, then it is streamed to the UI, stays grounded in retrieved evidence, uses the language of the question, and includes numbered citations with document/page information.
- **AC-03.4** Given the strongest evidence is below the approved threshold or does not support an answer, when the learner asks the question, then the app responds “I could not find this in your notes” (or the approved localized equivalent) instead of guessing.
- **AC-03.5** Given an answer contains citations, when the learner taps a citation, then the app opens the corresponding PDF at the cited page.
- **AC-03.6** Given generated citations refer to absent or invalid source numbers, when the answer is checked, then invalid citations are not presented as tappable sources and the answer is repaired or safely refused according to the approved recovery rule.

### US-04: Keep chat history and delete private data

As a learner, I want local history and a dependable way to remove it.

- **AC-04.1** Given a conversation, when messages are sent or received, then its user/assistant content, model identifier, timestamps, and cited sources are persisted locally and remain available after process restart.
- **AC-04.2** Given the learner chooses delete-all, when deletion completes, then source files, document rows, chunks, embeddings, keyword-index rows, conversations, messages, and quizzes are removed; the app reports completion only after verifying deletion.
- **AC-04.3** Given app backup or device-to-device transfer is configured, when backup rules are applied, then documents, chat history, embeddings, and model files are excluded from cloud backup.

### US-05: Use Free or Pro features

As a learner, I want clear plan limits and reliable access to purchased features, including while offline.

- **AC-05.1** Given a Free entitlement, when the learner uses the app, then only one approved small model is available, no more than three documents can be ready/actively indexed, and no more than 20 questions are accepted per day.
- **AC-05.2** Given an active Pro entitlement, when the learner uses the app, then approved models, unlimited documents, and basic quiz generation are available.
- **AC-05.3** Given the learner views an offer, when monthly, yearly, or lifetime pricing is shown, then the app uses localized store-provided purchase details where applicable; ₹49 monthly, approximately ₹299 yearly, and approximately ₹499 lifetime remain proposed values until validated and configured in the store.
- **AC-05.4** Given the learner completes a verified purchase, when the app receives the entitlement, then the approved Pro features become available; pending, canceled, expired, revoked, restored, and offline states are handled according to the approved entitlement policy.
- **AC-05.5** Given the app performs purchase verification or entitlement refresh, when network data is sent, then it contains only the minimum purchase/entitlement data and never documents, extracted text, embeddings, prompts, messages, answers, or quiz content.

### US-06: Generate a basic quiz

As a learner, I want a quiz based on my notes so I can check my understanding. **[ASSUMPTION: quiz generation is Pro-only, consistent with the architecture's tier table; confirm in the open decisions.]**

- **AC-06.1** Given a Pro learner selects a ready document or approved document scope, when they request a quiz, then each generated question and answer is based only on retrieved document content and is stored locally.
- **AC-06.2** Given the model produces quiz output, when the app validates it, then output must satisfy the versioned structured schema and required fields before it is shown or saved; malformed output is not silently accepted.
- **AC-06.3** Given a quiz item, when the learner reviews its answer, then the supporting source/page can be inspected and any citation opens the correct PDF page.
- **AC-06.4** Given the learner is Free or supporting evidence is inadequate, when a quiz is requested, then the app explains the plan limit or refuses to generate unsupported material.

### US-07: Stay private and usable offline

As a learner, I want the study flow to remain private and usable without a connection.

- **AC-07.1** Given the app is used for importing, indexing, retrieval, chat, or quizzes, when network traffic is inspected, then no user document or derived/user-authored study content is sent. Only catalog refresh, model delivery, Play Billing, and the explicitly approved purchase-entitlement path are permitted.
- **AC-07.2** Given logging, analytics, or diagnostics are active, when an event is recorded or exported, then no document content, question, answer, prompt, embedding, or quiz content is present. Analytics/crash reports are off by default and require explicit opt-in.
- **AC-07.3** Given a verified catalog and downloaded model are cached, when the learner is offline, then already-supported core study actions continue without a network dependency, subject to the approved entitlement policy.

## 3. MVP Scope

- Android-only onboarding and support checks for Android 10+, arm64, and 4 GB RAM minimum.
- Signed model catalog refresh/cache, model recommendation, model license disclosure, Wi-Fi-default resumable downloads, checksum verification, local model removal, and offline use of downloaded models.
- PDF import with initial configurable limits of 100 MB and 1,000 pages, storage preflight, private copy, text extraction per page, chunking, embeddings, keyword index, progress, and resumable committed batches.
- Local document library, document selection/scope, cited question-answer chat, refusal for unsupported answers, citation navigation to the source page, and persistent chat history.
- Free limits of one approved small model, three documents, and 20 questions per day; Pro monthly/yearly/lifetime offers with proposed pricing; Pro access to approved models, unlimited documents, and basic quizzes.
- Basic quiz generation from retrieved content with schema/grammar-constrained structured output, local persistence, and source inspection.
- Delete-all, private storage, backup exclusion, content-free logs, and explicit opt-in anonymous diagnostics.
- English-first quality gate. Additional languages are not advertised until tested per model and passed.

## 4. Explicitly Out Of Scope

- iOS implementation, OCR, study packs, user accounts, cloud inference, document/chat synchronization, flashcards, spaced repetition, and summaries.
- SQLCipher encryption in the MVP. Data remains in app-private storage and excluded from backup, but database/model files are not encrypted at rest by this release. This is a documented residual risk; future encryption keys must be protected by Android Keystore.
- Any model or dependency that has not passed current license, attribution, redistribution, and commercial-use review.
- Billing implementation before design approval; the MVP requirements include tiers and offers, but coding is gated on current policy review and purchase-verification design approval.

## 5. Non-Functional Requirements

- TTFT: under 4 seconds on a 6 GB reference phone, measured over 50 fixed questions. **[ASSUMPTION: report median as the gate and also publish p95; approval needed.]**
- 3B-model generation: at least 5 tokens/second on the approved 6 GB reference phone; the model must also pass license review.
- Indexing: 100 pages in under 3 minutes on the agreed fixed PDF set and reference device.
- Retrieval hit rate: at least 85% for the expected page in the top four across the English golden set.
- Grounded answers: at least 90% on the fixed 100-question evaluation; the scoring rubric and judge/human adjudication need approval.
- Absent-answer refusal: at least 90% over 30 fixed out-of-scope questions.
- Cold start to chat screen: under 2 seconds on reference hardware.
- Battery per 10-minute chat: under 6% on reference hardware.
- Crash-free sessions: at least 99.5% before production launch; opt-in diagnostics alone may not provide a sufficient sample, so the measurement source/sample size needs approval.
- Installed application: under 100 MB without bundled language models.
- Every quality/performance target is measured on documented 4 GB, 6 GB, and 8 GB Android reference devices where applicable. The 6 GB device is the TTFT/generation gate device.

## 6. Edge Cases And Recovery

- Unsupported Android version, ABI, or memory: block unsupported operation with a clear message; no cloud fallback.
- No model or catalog connection: use the last verified catalog and downloaded models; if none is available, explain that setup requires connectivity.
- Interrupted model download: retain valid partial data for resume; discard a checksum-invalid artifact.
- Insufficient storage before import/download/indexing: show required and available storage; do not delete or partially replace existing user data; offer free-space/cancel actions.
- PDF exceeds size/page limits: reject before copying/parsing and state the limit.
- Malformed, encrypted/password-protected, scanned, or text-free PDF: show a specific error; OCR is not attempted.
- Process death during indexing: resume after the last fully committed batch; reconcile stale document status and orphaned chunks/files on next start.
- Low-quality retrieval or no evidence: use the exact not-found response rather than fabricate an answer.
- Model/runtime failure or unclean inference exit: recover on next launch with an actionable message and offer a lower-memory approved configuration; never claim in-process crash isolation unless separately implemented and measured.
- Invalid model signature, SHA-256 mismatch, unsupported model/license, invalid citation, or malformed quiz structure: block activation/display and preserve unrelated local data.
- Billing unavailable/offline, pending purchase, cancellation, expiration, refund, or revocation: follow an approved cached-entitlement/grace policy; do not grant unverified new purchases.

## 7. Open Decisions Requiring Approval

1. **Embedding backend:** M0 compares both approved candidates on the same real phone and corpus. Record results for compatibility, exact model size, pages/minute, retrieval metrics, peak memory, build size, and integration effort; approve one winner before M1. **No dual production backends.**
2. **LLM model catalog:** approve a specific model, quantization, source/download host, license and language list only after commercial and redistribution review. Qwen2.5-3B is blocked absent a separate commercial license; Llama/Gemma terms need review; other alternatives also require artifact-level checks.
3. **Model policy:** confirm the exact Free “small” model and which approved models are Pro. The RAM-to-model mapping and fallback recommendations need real-device evidence.
4. **Conversation scope:** choose default scope (all ready documents vs one selected document), multi-document behavior, and how scope changes affect follow-up retrieval.
5. **Daily question quota:** define timezone/reset instant, whether refusals/quiz generation/retries count, and what constitutes one question.
6. **Offline Pro entitlement:** approve cache duration/grace, what happens after that window without connectivity, and how refund/revocation freshness is balanced with offline access.
7. **Billing verification:** approve whether the minimal backend verifies Play purchase tokens and issues a signed locally cached entitlement. Define backend retention, security, purchase-token handling, and the network allowlist. No account or study content is needed by the proposed design.
8. **Quiz behavior:** confirm quiz access is Pro-only, number/type of questions, answer reveal flow, retry policy for invalid structured output, and whether quiz generation counts against the daily question cap.
9. **PDF encryption:** confirm password-protected PDFs are rejected with a clear message rather than prompting for a password in MVP.
10. **Storage preflight:** approve minimum working-space reserve and estimation method for PDF copy, extracted text, vectors, temporary files, and model downloads.
11. **Evaluation protocol:** approve TTFT statistic, answer-grounding rubric/adjudication, and the sample/measurement source for crash-free-session target.
12. **Model catalog signing:** approve signature algorithm, key custody/rotation/revocation procedure, and catalog endpoint before catalog implementation.
