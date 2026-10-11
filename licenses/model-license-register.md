# Model And Dependency License Register

**Status:** Initial desk review; not legal approval. No model artifact is approved for production release yet.
**Last reviewed:** 2026-10-11.
**Policy:** Verify the exact upstream revision, exact quantized/exported artifact, transitive native binaries, and current license text before catalog inclusion. A repository/model-card license label alone does not clear redistributed derivatives. Block unknown, incompatible, or commercially unreviewed assets.

## Model Candidates

| Asset | Exact artifact / source | Declared terms observed | Commercial / redistribution review | Attribution / notices | Status |
| --- | --- | --- | --- | --- | --- |
| Qwen2.5-3B-Instruct | [Model card](https://huggingface.co/Qwen/Qwen2.5-3B-Instruct); [license file](https://huggingface.co/Qwen/Qwen2.5-3B-Instruct/blob/main/LICENSE) | Qwen Research License Agreement; non-commercial purposes only under reviewed text | Pro use requires separate commercial license. Exact GGUF quantization/depot also needs review. | Agreement requires license copy and attribution notice for redistributed Materials. | **Blocked** unless separate written commercial permission is obtained and reviewed. |
| Llama 3.2-3B-Instruct | [Model card](https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct) | Llama 3.2 Community License and Acceptable Use Policy; gated access | Custom terms; commercial use conditions, redistribution, notices and geographic/user conditions require counsel review. Quantized artifact provenance must be checked. | Agreement requires license copy and “Built with Llama” display/attribution conditions when applicable; retain required notices. | **Review required; blocked from catalog.** |
| Gemma 3-4B-it | [Model card](https://huggingface.co/google/gemma-3-4b-it); [Terms](https://ai.google.dev/gemma/terms) | Gemma terms; access files require acceptance | Review commercial paid-app use, restrictions, redistribution, notices, and quantized artifact provenance. | Record exact required attribution/notices from accepted terms. | **Review required; blocked from catalog.** |
| Phi-4-mini-instruct | [Model card](https://huggingface.co/microsoft/Phi-4-mini-instruct) | Card states MIT | Verify the exact GGUF quantized artifact, who created/distributes it, whether source weights and conversion are redistributable, and all included notices. | Include MIT license and copyright notice if the selected artifact is covered by MIT. | **Candidate; artifact-level review required.** |
| Other LLM candidate | Exact model/revision not selected | Not assessed | Must meet commercial-use, redistribution, device, quality, and language gates. | Determine after exact artifact selection. | **Not assessed; do not catalog.** |
| GGUF embedding candidate | Exact model/revision not selected | Not assessed | Verify model license, conversion/quantization rights, tokenizer/metadata redistribution, and commercial paid-app use. | Determine after artifact selection. | **Not assessed; M0 candidate only.** |
| multilingual-e5-small | [Model card](https://huggingface.co/intfloat/multilingual-e5-small) | Card states MIT | Verify exact ONNX export/conversion, tokenizer/config files, quantized variant, and redistribution source. | Preserve applicable MIT notice for all redistributed model files. | **Candidate; exact ONNX bundle review required.** |

## Runtime, Platform, And Direct Dependencies

| Component | Source | Declared license/status observed | Required action | Status |
| --- | --- | --- | --- | --- |
| React Native | [Project](https://github.com/facebook/react-native) | MIT | Pin selected version and archive license/NOTICE from actual package. | Candidate; verify resolved version. |
| Zustand | [Project](https://github.com/pmndrs/zustand) | MIT | Pin selected version and include notice in SBOM/notices. | Candidate; verify resolved version. |
| llama.rn | [Project](https://github.com/mybigday/llama.rn) | MIT | Record exact npm version, downloaded native artifacts, subcomponents, and upstream commit. Audit postinstall/native artifact provenance and notices. | Candidate; exact build unresolved. |
| llama.cpp | [Project](https://github.com/ggml-org/llama.cpp) | MIT | Pin commit as architecture requires; archive exact license and third-party notices for vendored native code. | Candidate; commit unresolved. |
| ONNX Runtime Mobile / React Native package | [Docs](https://onnxruntime.ai/docs/get-started/with-javascript/react-native.html), [project](https://github.com/microsoft/onnxruntime) | MIT for ONNX Runtime project | Verify the exact package version, native binary build, execution-provider additions, and bundled third-party notices. | Candidate; M0 alternative. |
| op-sqlite | [Project](https://github.com/OP-Engineering/op-sqlite) | MIT | Record exact release and selected plugins/backends; inspect native packaged components and notices. | Candidate; exact version unresolved. |
| sqlite-vec | [Project](https://github.com/asg017/sqlite-vec) | Dual MIT/Apache-2.0; pre-v1 software | MVP plan defers it. If added later, pin stable reviewed revision, select license basis, audit C sources and notices. | **Not in MVP dependency set.** |
| SQLite / FTS5 | [SQLite](https://www.sqlite.org/copyright.html) | SQLite source is public domain; confirm any bundled build/extensions separately | Identify which SQLite binary op-sqlite ships and verify FTS5 source/compile options and third-party additions. | Candidate; bundled artifact review required. |
| PdfBox-Android | [Project](https://github.com/TomRoush/PdfBox-Android) | Apache-2.0 | Preserve Apache license/NOTICE and review transitive Android dependencies; verify exact Maven artifact version and maintenance suitability. | Candidate; parser spike required. |
| Android WorkManager | [AndroidX docs](https://developer.android.com/topic/libraries/architecture/workmanager) | AndroidX libraries generally Apache-2.0; verify exact resolved artifact | Archive license and dependency notices from Gradle resolution. | Candidate; exact version unresolved. |
| Android PdfRenderer / Android SDK | [API docs](https://developer.android.com/reference/android/graphics/pdf/PdfRenderer) | Platform API; distribution governed by Android SDK/platform terms | No separate app-bundled model/parser license assumed; verify SDK/build requirements and notices. | Platform API candidate. |
| Google Play Billing Library | [Current integration docs](https://developer.android.com/google/play/billing/integrate) | Exact artifact terms not confirmed in this review | Verify exact Billing Library artifact, license/notice, current minimum version, and distribution terms before dependency pin. | **Billing design gate; not yet selected.** |
| TypeScript / Jest / Detox / build tools | Exact versions not selected | Not assessed in this workspace | Generate inventory from lockfiles and Gradle/SBOM once scaffold is approved; verify licenses of transitive packages/native binaries. | Not assessed. |

## Review Checklist Per Release Artifact

- [ ] Record package/model name, exact version/revision, hash, source URL, acquisition date, and maintainer.
- [ ] Record license identifier and link to the license text as shipped/currently accepted.
- [ ] Confirm commercial use, model redistribution, quantization/export/derivative rights, and gated access conditions.
- [ ] Confirm attribution, visible branding, NOTICE/license-file placement, and user-facing license disclosure.
- [ ] Inspect native libraries, vendored code, tokenizers, vocabularies, generated ONNX graphs, GPU/NPU providers, and transitive dependencies.
- [ ] Add required notices to the app/catalog distribution and Play listing where required.
- [ ] Record reviewer, review date, decision rationale, follow-up/renewal date, and Approved/Blocked status.
- [ ] Recheck before launch and when model terms, artifact source, dependency version, quantizer, or distribution method changes.

## Current Release Gate

**No LLM artifact is approved for the commercial Pro catalog.** Qwen2.5-3B is blocked under the reviewed non-commercial license absent a separate commercial grant. Llama and Gemma remain blocked pending legal review. Phi-4-mini is a potentially compatible candidate based on its card's MIT label, but its exact GGUF artifact and redistribution provenance still need approval. The selected embedding artifact and all resolved native/transitive dependencies also require artifact-level clearance.
