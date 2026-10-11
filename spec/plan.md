# Technical Plan: Offline AI Study App

**Status:** Proposed architecture for approval. No application implementation is authorized until this plan and `spec/spec.md` are approved.
**Source:** `Architecture/Complete Arch.docx`; decisions approved 2026-10-11; upstream checks noted below as of 2026-10-11.
**Constraint:** Android MVP; keep core contracts portable for a later iOS adapter.

## 1. Architecture Overview

```text
React Native TypeScript UI and state
        | typed application ports / New Architecture native modules
        v
Android adapters (Kotlin): PDF, WorkManager, files/downloads, device, billing
        | JNI/JSI/TurboModule calls on bounded native worker executors
        v
Portable C++ core: llama.cpp inference adapter, embeddings interface,
chunking/token helpers, retrieval fusion, prompt and citation validation
        |
        +---- local SQLite database, FTS5, serialized vector BLOBs
        +---- app-private PDF and model files

Allowed network boundary: signed model catalog / model CDN / Play Billing /
approved minimal purchase-verification endpoint only. Never study content.
```

The native boundary owns CPU-heavy parsing, inference, embedding, indexing, and search. JavaScript coordinates user workflows and renders bounded progress/token events; it does not synchronously run those computations. Keep contracts independent from Android classes so a future iOS adapter can implement them without changing domain behavior.

## 2. Proposed Repository Structure

```text
app/
  src/app/                 navigation, app composition, localization
  src/features/library/    import, document list/detail, progress/error states
  src/features/chat/       conversation UI, streaming, source chips
  src/features/models/     catalog, recommendations, download/removal UI
  src/features/quiz/       quiz flow and validated result display
  src/features/settings/   privacy, diagnostics opt-in, delete-all, entitlements
  src/domain/              use cases, entities, ports, entitlement rules
  src/data/sqlite/         migrations, repositories, FTS/vector persistence
  src/native/              typed native-module adapters and event mapping
core/
  include/study_core/      stable C ABI or narrowly scoped C++ headers
  src/                     chunking, RRF, prompts, citations, inference adapter
  tests/                   native unit and contract tests
platform/android/
  app/                     manifest, backup rules, Android application shell
  src/                     Kotlin modules, WorkManager workers, PDF/file APIs
  billing/                 isolated Play Billing adapter (gated)
platform/server/            purchase-only verifier, only if separately approved
catalog/
  schema/                  signed catalog schema and compatibility fixtures
  tools/                   offline catalog validation/signing tools
  public-key/              public verification key only; no signing secret
licenses/
  model-license-register.md model, quantization, runtime and dependency review
eval/
  golden/en/               questions, expected page(s), answerability labels
  retrieval/               deterministic retrieval metrics
  answer/                  offline answer scoring/adjudication tooling
tests/
  integration/             persistence, ingestion and native contract tests
  e2e/                     Jest/Detox or approved UI automation
  device/                  repeatable reference-device procedures
```

Do not create iOS application files in the MVP. Keep cross-platform contracts under `app/domain` and `core`; Android-only implementations stay under `platform/android`.

## 3. Technology Decisions And Verification Gates

| Concern | Plan | Decision / evidence / gate |
| --- | --- | --- |
| UI | React Native New Architecture, TypeScript, Zustand | Preserve source decision. Current RN documentation describes JSI/TurboModules and New Architecture as enabled by default in recent versions. Pin versions only after native package compatibility is proven. |
| LLM | `llama.cpp` through `llama.rn`, GGUF quantization selected per license/device evidence | `llama.rn` v0.10+ requires New Architecture; current repository documents streaming and embedding APIs and Android arm64-v8a/x86_64 support. M0 must measure callback behavior, load/unload, cancellation, peak RSS, ABI/build packaging, and actual tokens/s. Do not infer that a promise or JSI call is off the UI thread; profile and use a native executor. |
| Embedding | Replaceable `EmbeddingProvider`; production backend unselected until M0 | Compare (A) GGUF embedding model via `llama.rn` and (B) ONNX Runtime Mobile with multilingual-e5-small. Same test documents, queries, dimensions/normalization, device state, and runtime. Record binary/model sizes, compatibility, pages/minute, retrieval hit rate, peak memory, integration complexity, and thermal impact. Approve exactly one backend before M1; no dual production backends absent measured need. |
| E5 behavior if ONNX wins | 384-dimensional multilingual-e5-small; query prefix `query: ` and chunk prefix `passage: `; normalize consistently; respect 512-token model input limit | Model card confirms 384 dimensions, 100-language family support (low-resource quality may degrade), required prefixes, and truncation beyond 512 tokens. Validate the selected ONNX export and tokenizer exactly; do not assume exported graph parity. |
| Persistence | One local SQLite database, app-private; `op-sqlite` candidate, FTS5, brute-force cosine on vector BLOBs for MVP | Matches architecture's stated MVP approach; defer `sqlite-vec`, which is pre-v1 and breaking-change-prone. Verify the exact SQLite build has FTS5, transactions, required BLOB behavior, New Architecture compatibility, and acceptable scan speed at expected document limits. |
| PDF | Kotlin Android adapter using PdfBox-Android candidate for per-page text; Android `PdfRenderer` only for display/navigation | `PdfRenderer` does not extract text per source. PdfBox-Android is Apache-2.0 but its latest listed release is old; run malformed/encrypted/large/scanned corpus, memory and API-level tests before retaining it. Do not replace without an ADR if it fails. |
| Durable indexing/download | WorkManager for restart-resilient import/index/download steps with progress notification and storage/network constraints; interactive chat inference uses a foreground native worker, not a durable WorkManager job | Android guidance supports persistent/long-running work with foreground notification and constraints. Confirm long-running quotas, cancellation, process death, Doze, Android 10+, and user-initiated download policy against current OS behavior. |
| Model catalog | Signed data-only JSON, versioned schema, model IDs, URLs, byte sizes, SHA-256, minimum RAM/ABI, license URL/identifier, attribution, language list, quantization, embedding compatibility, signature metadata | Catalog is data, never executable code. Signature algorithm, key rotation/revocation, host allowlist, and signing operations remain approval gates. App contains verification key only. |
| Billing | Kotlin Play Billing adapter isolated behind entitlement port; small backend is the recommended verification path pending approval | Current Google docs (checked 2026-10-11) require Billing Library 8+ for new updates after 2026-08-31; integration guide currently shows 9.1.0 and recommends secure backend purchase verification. Do not hardcode version until implementation begins. Current docs say pending purchases do not confer entitlement and completed purchases must be acknowledged. Backend receives only purchase token/necessary opaque install identifier and returns minimal entitlement; no account or study content. |
| Billing offline behavior | Cache verified entitlement locally; locally enforce Free caps; recheck on foreground/connectivity | Cache lifetime, signed cache format, revocation latency, reinstall/restore, and offline grace are not specified. Do not implement before decision record and approved server retention/security design. |
| Security | Android sandbox; backup exclusion; SHA-256 plus signed catalog; content-free logs; opt-in diagnostics; SQLCipher deferred | Document residual unencrypted-at-rest risk. Any later database key is generated/stored through Android Keystore; model files remain separately removable. Test legacy and current Android backup/data-extraction rules. |

### Current License Findings (Not Legal Approval)

- Qwen2.5-3B-Instruct's current model card points to a Qwen Research License Agreement. It explicitly limits the grant to non-commercial purposes and requests a separate license for commercial use. Block this candidate from the Pro catalog unless written commercial permission is obtained.
- Llama 3.2-3B-Instruct uses Meta's custom community license, includes use/redistribution terms and attribution requirements, and has a gated download flow. Legal review and artifact/source verification are required before commercial distribution.
- Gemma 3-4B requires acceptance of Google terms; review commercial app distribution, attribution, and any restrictions before catalog approval.
- Phi-4-mini-instruct's model card states MIT, but review the exact GGUF quantizer, weight source, conversion/derivative rights, and notice files before catalog inclusion.
- multilingual-e5-small's card states MIT. Verify the exact ONNX export, tokenizer files, and redistribution packaging.
- `llama.rn` is MIT; `llama.cpp` is MIT; `op-sqlite` is MIT; `sqlite-vec` provides MIT/Apache-2.0 licenses; PdfBox-Android is Apache-2.0. Record transitive dependencies and native binaries in the register; verify notices and versions in the actual lockfiles/build artifacts.

The license register is a release gate, not a substitute for counsel. Each row must record exact artifact/revision, source URL, declared license, commercial-use basis, redistribution/quantization basis, attribution/notice actions, reviewer, review date, and Approved/Blocked status.

## 4. Module Boundaries And Contracts

Contracts below are conceptual TypeScript domain interfaces, not claims about third-party API signatures. Native module APIs must be confirmed against pinned package docs at implementation time.

```ts
export type DocumentId = string;
export type ConversationId = string;
export type ModelId = string;

export interface PageText {
  page: number;
  text: string;
}

export interface IngestionPort {
  importPdf(uri: string): Promise<DocumentId>;
  enqueueIndex(documentId: DocumentId): Promise<void>;
  observeProgress(documentId: DocumentId): AsyncIterable<IndexProgress>;
  cancel(documentId: DocumentId): Promise<void>;
}

export interface EmbeddingProvider {
  readonly modelId: string;
  readonly dimensions: number;
  embedPassages(texts: string[]): Promise<Float32Array[]>;
  embedQuery(text: string): Promise<Float32Array>;
}

export interface Retriever {
  search(query: string, scope: DocumentId[] | 'all', limit: number): Promise<RetrievedChunk[]>;
}

export interface AnswerGenerator {
  streamAnswer(request: AnswerRequest): AsyncIterable<GenerationEvent>;
  cancel(requestId: string): Promise<void>;
}

export interface QuizGenerator {
  generate(request: QuizRequest): Promise<ValidatedQuiz>;
}

export interface ModelManager {
  refreshCatalog(): Promise<CatalogResult>;
  recommend(device: DeviceProfile): Promise<ModelRecommendation>;
  download(modelId: ModelId): Promise<DownloadProgress>;
  remove(modelId: ModelId): Promise<void>;
}

export interface ChatStore {
  createConversation(modelId: ModelId, scope: DocumentId[] | 'all'): Promise<ConversationId>;
  appendMessage(conversationId: ConversationId, message: PersistedMessage): Promise<void>;
  loadRecent(conversationId: ConversationId, turnLimit: number): Promise<PersistedMessage[]>;
}

export interface EntitlementProvider {
  current(): Promise<EntitlementSnapshot>;
  refresh(): Promise<EntitlementSnapshot>;
  purchase(productId: string): Promise<PurchaseOutcome>;
  restore(): Promise<EntitlementSnapshot>;
}
```

`IngestionPort` orchestrates Android file copying, PDF page extraction, chunking, embedding, and atomic database commits. `Retriever` combines SQLite FTS5 and cosine candidates with reciprocal rank fusion (RRF). `AnswerGenerator` performs prompt preparation/inference on native workers and emits bounded token events. Citation validation maps only returned citation IDs to `RetrievedChunk` page metadata. `ChatStore` is the sole conversation persistence boundary. `QuizGenerator` returns only schema-valid locally persisted items. Entitlement provider is isolated and must not be called with content-bearing objects.

### Threading And Lifecycle

- Keep one serialized database writer or enforce explicit transaction ownership; do not let background indexing race delete-all or document removal.
- Interactive generation, model load/unload, tokenization, embeddings, and vector scoring run on native worker executors. Emit progress/tokens using bounded batches and preserve UI cancellation/backpressure semantics.
- WorkManager jobs persist only identifiers and restart-safe state. Each indexing batch writes chunks, vectors, FTS rows, and checkpoint progress atomically; retry is idempotent.
- A native process crash cannot be caught by JavaScript. Persist an unclean-exit marker outside the inference context; next launch offers safe recovery. Do not call this process isolation.
- Model unload after two idle minutes is a target; confirm memory reclamation and user-visible latency in M0 before making it a hard behavior.
- Thermal behavior and performance-core identification vary across vendors. Begin with Android-supported thermal status and conservative adaptation; do not rely on undocumented core topology. Validate sensor availability and thresholds on reference devices.

## 5. SQLite Design And Migration Rules

One database under app-private storage; numbered forward migrations; foreign keys enabled and tested. Values below are logical fields, not finalized SQL. Enable WAL only after checking backup, transaction, and recovery behavior. Keep model binaries and original PDFs in private file storage; persist relative/opaque paths, never public file URIs.

| Entity | Proposed fields and constraints |
| --- | --- |
| `document` | `id`, `title`, `path`, `mime`, `size_bytes`, `pages`, `status` (`queued/indexing/ready/failed`), `embed_model`, `created_at`, `updated_at`, `failure_code`; unique active source identity if approved. |
| `chunk` | `id`, `document_id` FK cascade, `ordinal`, `page`, `text`, `embedding` BLOB, `embedding_model`, `embedding_dimensions`, normalized-vector indicator, `token_count`; unique `(document_id, ordinal)`. MVP uses 384 dimensions only if the selected embedder confirms that shape. |
| `chunk_fts` | FTS5 external-content index over `chunk.text`, rowid=`chunk.id`; triggers or transaction-managed synchronization for insert/update/delete, verified with consistency tests. |
| `conversation` | `id`, `title`, `model_id`, `scope` as versioned local ID list or explicit all-documents marker, `updated_at`. Define deletion behavior for removed documents. |
| `message` | `id`, `conversation_id` FK cascade, checked role, `content`, structured `sources` JSON, `tokens`, `created_at`; validate source IDs before persistence. |
| `quiz` / `quiz_item` | `quiz` references conversation/document scope, model/schema version, timestamps; items contain validated question/options/correct answer/explanation/source chunk IDs. Store no remote state. |
| `usage_day` | local entitlement/question count keyed by a documented day boundary; anti-clock-manipulation behavior is an open decision. |
| `entitlement_cache` | minimal plan/status/valid-through/signature or integrity field and last-verified time; purchase tokens retained only as required by the approved billing design. No content. |
| `model_install` | `model_id`, private path, expected/verified size and SHA-256, state, catalog version, license acceptance/notice state, timestamps. Download partials tracked separately or by verified atomic temp-file convention. |
| `index_checkpoint` | document/job ID, committed page/batch/ordinal, progress and retry metadata; same transaction as corresponding chunks and FTS updates. |

**Migration tests:** create/upgrade from every supported schema version; preserve history; verify foreign-key cascades, FTS synchronization, vector dimensions, crash rollback, delete-all, and transaction idempotence. Keep a fixture database for every released schema.

## 6. Model Catalog And License Gate

Catalog fields: `schema_version`, `catalog_version`, `issued_at`, `expires_at`, list of model records, and detached signature metadata. Each model record includes `id`, display name, exact upstream and quantized artifact identifiers, URL, byte size, SHA-256, minimum RAM, supported ABIs, quantization, runtime compatibility, license identifier/URL, attribution/notice text reference, intended commercial-use status, embedding compatibility, supported languages, and approval state. App refuses an invalid signature, unsupported schema, blocked license state, digest mismatch, or ABI/RAM incompatibility.

Never download executable logic. Verify signature before using URLs/metadata; permit only approved HTTPS hostnames. Catalog signer uses a protected offline/restricted private key; app stores only public verification material. Key rotation and revocation require a signed, backward-compatible procedure and a test for compromised/expired catalogs. Do not publish a model entry until its exact model and quantized file provenance is license-reviewed and its per-language quality gate passes.

## 7. Data And Control Flows

### Ingestion

1. UI asks the platform file picker for a PDF URI; adapter checks MIME, byte size, page limit, and available storage before copying.
2. Copy to an app-private temporary path; validate PDF readability/encryption and extract page-indexed text on a native worker.
3. Clean headers/footers/hyphenation conservatively; record extraction diagnostics without content. Empty/scanned documents fail clearly (OCR deferred).
4. Chunk by headings/paragraphs/sentences at 350 tokens with 60 overlap. Tokenizer and prefix overhead are tied to the selected embedding model; prevent silent truncation at the embedding model's input cap.
5. Batch 16 texts for embedding. In one SQLite transaction per committed batch, insert chunks/vectors, update FTS5 and checkpoint. Respect stop/cancel and storage pressure.
6. On successful final validation, mark ready and atomically rename the original file into its final private path. On failure, expose a stable error code and clean only this operation's uncommitted temporary data.
7. On restart, reconcile queued/indexing states, checkpoints, partial files, and committed records before scheduling resume. Removing a document cancels work and deletes file, chunks, FTS rows, and dependent quiz/source links consistently.

### Query And Answer

1. Verify local model availability, entitlement/question quota, and nonempty ready-document scope.
2. Rewrite only follow-up questions using at most the last two turns; embed the rewritten query locally.
3. Query top 20 semantic matches by cosine and top 20 FTS5 matches; fuse rankings with an explicitly configured RRF constant and deterministic tie-break, then choose up to four chunks within a 1,500-token evidence budget.
4. Compare evidence confidence against a threshold calibrated on held-out relevant and absent-answer sets. The document's phrase “best similarity” does not identify a comparable score after RRF; calibrate raw semantic score and/or a separate confidence feature, version the threshold, and approve exact semantics before implementation.
5. Prompt contains fixed system grounding rules, untrusted numbered context and page labels, recent chat, and question. Model output is streamed to UI; answer only from retrieved context, cite numbered sources, answer in question language, and keep concise by default.
6. Validate citations against the source map; save user/assistant messages and structured citations transactionally. Tap source opens the private PDF viewer at the preserved page index.
7. If retrieval threshold fails, return the specified not-found response without model guessing. Record quality telemetry locally without content.

### Quiz

1. Verify Pro entitlement and selected ready-document scope.
2. Retrieve the chunk/evidence set and invoke the same local model with a quiz-specific constrained output schema/grammar.
3. Parse and validate item count/type/answer/source references. Invalid output follows the approved bounded retry/refusal rule; never expose invalid JSON as a quiz.
4. Persist the quiz and sources locally. Quiz citations reuse source-page navigation. Flashcards and SM-2 remain deferred.

### Purchase And Entitlement (Design Only; Implementation Gate)

1. Native Play Billing presents current localized product details; do not hardcode regional prices.
2. Client submits purchase token and minimum opaque installation identifier to the approved verifier only; verifier checks purchase state with Google Play and returns a signed/minimal entitlement snapshot. No account is introduced by default.
3. Client grants Pro only for verified `PURCHASED` state, acknowledges eligible purchases within current Play deadlines, handles restore/out-of-app purchase and revocation, and caches the snapshot locally.
4. Offline use follows the approved cache grace/expiry. When offline eligibility expires, keep private data intact and explain that entitlement refresh needs connectivity; do not upload study data.
5. Lifetime is modeled as a one-time non-consumable product; monthly/yearly are subscription base plans. UPI AutoPay and regional product configuration require current Play Console and India-specific verification before launch.

## 8. Prompt, Grounding, And Injection Handling

System rules are immutable application prompt content: answer only from numbered evidence; cite source numbers; state “I could not find this in your notes” if evidence is insufficient; answer in the question language; be concise unless asked for detail. Place retrieved PDF text in a clearly delimited untrusted-data section. It cannot add instructions, change policy, call tools, or alter citation mapping. Do not put private message content into local logs. Test prompt injection using malicious instructions embedded in PDFs and retrieval distractors across every approved model/template version.

## 9. Error And Recovery Matrix

| Condition | Required behavior |
| --- | --- |
| Unsupported Android/ABI/RAM | Explain unsupported state, preserve local data, no cloud fallback. |
| Catalog offline | Use last verified cached catalog and installed models; show setup-required state if no model is installed. |
| Bad signature/expired schema/unknown host | Reject catalog and retain last valid copy. |
| Partial model download/interruption | Resume validated ranges by expected byte/hash; never load partial. Mismatch deletes partial and offers retry. |
| Low storage | Preflight bytes plus approved working reserve before each operation; show required/free amounts, preserve data, allow cancel/free-space. |
| Oversize/over-page PDF | Reject before copy/extraction. |
| Malformed/encrypted/scanned/no-text PDF | Stable actionable error; cleanup this import's incomplete artifacts; no OCR/password guessing. |
| Interrupted indexing | Resume from atomically committed checkpoint; reconcile stale state and guarantee idempotence. |
| Corrupt/missing model or native failure | Report actionable error; persist unclean exit marker; next launch offer smaller approved context/model; do not promise process isolation. |
| No/weak retrieval | Return not-found; do not invoke unconstrained answer fallback. |
| Invalid citations/quiz structure | Validate and boundedly retry only if approved; otherwise refuse and do not persist invalid result. |
| Delete while work active | Cancel work, serialize with writer, remove all related local artifacts, and verify completion. |
| Billing offline/pending/revoked | Never grant a new pending/unverified purchase; use only still-valid cached entitlement under approved grace; preserve data if Pro expires. |

## 10. Testing And Evaluation Strategy

- **M0 device spike:** run LLM loading/streaming and the embedding comparison on a real Android phone, then the 4/6/8 GB reference set. Record exact device/build/model/hash, environment, warm/cold state, size, indexing pages/min, retrieval top-4 hit rate, TTFT, tokens/s, peak RSS, battery/thermal behavior, crashes, and integration effort. Store results under `eval/reports/`; an ADR names the single embedding backend. No winner that meets minimum retrieval and resource needs means stop and revise plan before M1.
- **Unit tests:** chunk boundaries/overlap/page provenance; text cleanup; vector normalization/cosine; RRF ordering and ties; threshold; prompt rules; citation/source mapping; schema validation; entitlement/quota transitions; catalog signature/manifest validation.
- **Native tests:** PDF fixtures for text, page boundaries, malformed, encrypted, scanned/text-free, 100 MB/page-limit boundaries; worker cancellation/retry/process death; memory/thermal behavior; file cleanup.
- **Database tests:** migration from every schema; FTS triggers; transaction rollback; batch checkpoint idempotence; deletion cascades; vector round-trip/dimension; backup exclusions.
- **Retrieval CI:** fixed 100-query English golden corpus with expected page labels, top-4 hit rate >=85%, per-change run on chunker/embedding/retrieval changes. Keep evaluation corpus licensed/owned and free of real user documents.
- **Answer evaluation:** fixed 100 answerable and 30 absent cases, source correctness and grounding rubric; >=90% grounded and >=90% correct absent-answer refusal. Run offline with fixed evaluator/version and human adjudication sample; no evaluation content sent to app services.
- **UI and flow tests:** Jest plus Detox or an approved equivalent for import, progress/resume, chat streaming/cancel, citations to page, history restart, limits, quiz validation, delete-all, and offline mode. Do not add a test framework until native-package compatibility is confirmed.
- **Network/privacy tests:** instrument proxy/DNS and Android network security tests for allowlisted hosts; airplane mode; assert no content in logs, crash payloads, analytics, requests, backups, or diagnostics export. Verify opt-in starts off.
- **Performance/release device tests:** TTFT over 50 fixed questions on 6 GB; 3B speed >=5 tok/s; 100-page indexing <3 min; cold start <2 s; battery <6%/10-minute chat; run on 4/6/8 GB devices and report measurement method. Crash-free >=99.5% is a release/closed-test gate with approved sample source/count.
- **Task discipline:** each T-ID creates or updates its acceptance test before implementation, runs the narrow test after implementation, and records measured device evidence and plan deviations. No commit is required by this approved workflow unless separately requested.

## 11. Requirements Traceability

| Spec criterion group | Owning modules | Required evidence |
| --- | --- | --- |
| AC-01.1 | DeviceProfile, ModelManager, Android adapter | API/ABI/RAM unit tests and unsupported-device manual check |
| AC-01.2–01.6 | CatalogVerifier, ModelManager, DownloadWorker, private file store | Signature/hash tests, resume/kill/retry device check, offline check, license register |
| AC-02.1–02.7 | PDF adapter, StoragePreflight, IngestionPort, IndexWorker, repositories | Boundary fixtures, storage-full test, page/chunk assertions, interrupted-worker test, privacy audit |
| AC-03.1–03.6 | EmbeddingProvider, Retriever, PromptBuilder, AnswerGenerator, CitationValidator, PDF viewer | Golden retrieval/answer eval, injection tests, source-map/citation tests, page navigation test |
| AC-04.1–04.3 | ChatStore, SQLite migrations, DeleteAllUseCase, Android backup rules | Restart persistence, deletion reconciliation, manifest/backup inspection |
| AC-05.1–05.5 | EntitlementProvider, QuotaPolicy, Billing adapter/backend, ModelManager | Quota boundary tests, Play license-test scenarios, offline grace/revocation tests, network payload audit |
| AC-06.1–06.4 | QuizGenerator, Retriever, schema/grammar validator, QuizStore | Schema/property tests, source validity, unsupported evidence, Free/Pro boundary tests |
| AC-07.1–07.3 | NetworkPolicy, Catalog/Download/Billing adapters, diagnostics/privacy settings | Allowlist and airplane-mode tests, log/export/backup scans, opt-in default-off assertion |
| NFR targets | Core, workers, UI, evaluation/device suite | Published benchmark report with exact device/build/model versions and sample method |

## 12. Alternatives And Decisions

- **Embedding:** native GGUF through the selected LLM runtime may reuse native infrastructure and avoid another runtime; ONNX Runtime Mobile provides a dedicated graph and a model-card-supported embedding path. Neither is selected without the approved M0 measurement because RAM, speed, package size, quality, and tokenizer parity on real phones decide the trade.
- **Vector retrieval:** `sqlite-vec` may improve index query performance but is pre-v1 and adds extension packaging/version risk. Brute-force cosine over BLOBs is the architecture's MVP approach; benchmark it at maximum supported corpus size and revisit only with measured evidence.
- **SQLite:** `op-sqlite` is the listed candidate with FTS5 and extension support; a hand-written Kotlin SQLite layer would reduce JS/native database crossing but duplicate data API ownership. Retain op-sqlite provisionally and validate New Architecture, FTS5, native background access, and transaction behavior.
- **PDF extraction:** PdfBox-Android follows the source architecture and is Apache-2.0; Android PdfRenderer is for display only. If PdfBox fails modern build or corpus tests, compare a maintained commercial-compatible local parser and obtain approval before changing the specified technology.
- **Purchase verification:** client-only checks reduce backend footprint but have replay/tampering and delayed-revocation limitations. A purchase-only verifier is the smallest recommended compliant design given current Play guidance; it receives no study content. Approval is required for backend/network scope, identifiers, retention, signing, and grace policy before coding.
- **Database encryption:** SQLCipher now would add key lifecycle/migration and build risk. Source explicitly defers it; retain sandbox+backup exclusion for MVP, document residual at-rest risk, and plan SQLCipher/Keystore as later hardening.

## 13. Remaining Risks And Approval Gates

1. Embedding comparison requires an actual Android handset and repeatable fixed corpus; no backend can be selected from desktop/docs alone.
2. No model is approved for paid release yet. Qwen2.5-3B is explicitly non-commercial under the current license unless separately licensed. Other candidate model/quantized artifacts remain review-blocked.
3. `sqlite-vec` is pre-v1; plan intentionally does not require it for MVP.
4. PdfBox-Android maintenance age, malformed-PDF resource consumption, and behavior on supported APIs require a device/parser spike.
5. `llama.rn` New Architecture compatibility is documented but exact React Native, NDK, Gradle, CMake, ABI, artifact download, and worker-thread behavior need a pinned build spike.
6. ONNX embedding export/tokenizer parity, quantization, native packaging, model size, and licensing need exact artifact verification.
7. Purchase verification can require a backend and permitted network transfer; its privacy and security design is pending approval. Play requirements, India/UPI support, price configuration, regional pricing, and lifetime product configuration need launch review.
8. Retrieval confidence thresholds, score semantics after RRF, daily quota reset rules, storage reserve, and offline entitlement policy are acceptance-critical unresolved requirements.
9. SQLCipher is deferred: sandboxing and backup exclusion do not protect files from a compromised/unlocked device or all local forensic access.
10. Thermal and battery behavior varies by manufacturer; numerical release claims require representative physical devices, not emulators.
11. Architecture roadmap's 17–18 full-time weeks conflicts with an embedded 6–8 weeks of evenings. Scope should be scheduled by measured M0/M1 exit gates, not promise a launch date.
