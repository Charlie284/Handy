# Models and engines

Recognition runs on your computer after model files are available. Downloading
models needs network access; local inference does not require an LLM provider.
Optional text refinement has its own [privacy considerations](post-processing.md).

## Choose by capability, not just family

The registry exposes language support, explicit language selection, language
detection, translation and streaming separately. Not every model in a family
supports the same features. The UI uses `ModelInfo` from
[managers/model.rs](../src-tauri/src/managers/model.rs), with capability evidence
from [model_capabilities.rs](../src-tauri/src/managers/model_capabilities.rs)
and the [bundled catalog](../src-tauri/src/catalog/catalog.json).

Streaming-capable models can produce live text during recording. Overlay style
only controls whether/how you see it. Language intent is reconciled with model
capabilities; Chinese script selections are normalized for recognition and
handled by output normalization. Hardware choices and idle unloading are
described in [settings](settings.md).

## Engine implementations

`EngineType` in [model.rs](../src-tauri/src/managers/model.rs) and `LoadedEngine`
in [transcription.rs](../src-tauri/src/managers/transcription.rs) are the source
of truth:

| Engine route                      | Model formats/families                                                                                                                                    |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TranscribeCpp`                   | GGML/GGUF; architecture detected by transcribe-cpp, including Whisper and non-Whisper families. All bundled catalog descriptors currently use this route. |
| `Parakeet`                        | ONNX Parakeet via transcribe-rs.                                                                                                                          |
| `Moonshine`, `MoonshineStreaming` | ONNX batch and streaming Moonshine implementations.                                                                                                       |
| `SenseVoice`                      | ONNX SenseVoice.                                                                                                                                          |
| `GigaAM`                          | ONNX GigaAM.                                                                                                                                              |
| `Canary`                          | ONNX Canary.                                                                                                                                              |
| `Cohere`                          | ONNX Cohere.                                                                                                                                              |

The checked-in catalog includes Canary, Cohere, Fun-ASR, GigaAM, Granite,
MedASR, Moonshine, Nemotron, Parakeet, Qwen3, SenseVoice, Voxtral and Whisper families.
The catalog is a list of supported packaged models, **not** a guarantee that an
arbitrary model with a similar name will load. Runtime compatibility depends on
the pinned libraries in [Cargo.toml](../src-tauri/Cargo.toml).

## Catalog and discovery

[catalog/mod.rs](../src-tauri/src/catalog/mod.rs) compiles `catalog.json` into
the binary, so browsing the bundled list does not require a network request.
Entries carry pinned HF revisions, quantization files, sizes, hashes, capability
flags, language lists, scores and recommendation metadata. The default quant
determines the initially offered file; on-disk alternate catalog quants retain
catalog metadata.

`ModelManager` combines the legacy table, catalog, local model directory and
shared Hugging Face cache scans. A registry ID for catalog/HF models is
`repo_id/filename`, allowing discovered cached files to deduplicate with catalog
entries. Header probing supplies metadata for custom files. HF cache discovery
filters compatibility; manual drop-ins are less restrictive and can appear in
the list without being loadable.

## Storage and downloads

Implementation: [model.rs](../src-tauri/src/managers/model.rs),
[download.rs](../src-tauri/src/managers/model/download.rs), and
[portable.rs](../src-tauri/src/portable.rs).

- The app-managed model directory is the portable-aware app data directory plus
  `models/`. In portable mode that is `Data/models/` beside the executable.
- HF-backed models use the shared `hf-hub` cache (`Cache::from_env()`, respecting
  `HF_HOME`; normally `~/.cache/huggingface/hub`), not necessarily the app's
  model directory. The manager also discovers files already present there.
- `ModelSource::Url` downloads directly; it can carry a SHA-256 expectation.
  Directory packages are extracted by the manager.
- `ModelSource::HuggingFace` downloads the chosen repository file/revision using
  hf-hub. Catalog pins make downloads immutable; lookup can also find older
  `main` cache entries.
- Catalog mirror fallback URLs are derived from mirror base, repository,
  revision and filename. A catalog SHA-256 is required before offering a mirror;
  mirrors are not trusted to supply the expected hash.
- `ModelSource::Local` needs no download. Progress, completion, cancellation,
  extraction and deletion are surfaced to the frontend as events.

Do not rename ONNX package contents arbitrarily: loaders expect their engine's
file layout. Download cancellation and partial files are managed by the backend;
the presence of a partial file is not proof of a loadable model.

## Adding a model

### Personal local GGML/GGUF model

Place a compatible `.gguf` or `.bin` model in the app's `models/` directory and
refresh the model list (or restart). Discovery is implemented by
`discover_custom_transcribe_models()` in [model.rs](../src-tauri/src/managers/model.rs).
The manager probes GGUF headers for metadata; legacy `.bin` capabilities remain
unknown until load. Unsupported or unreadable files are not made compatible
merely by their extension. Existing shared HF cache models are also
scanned. This does not add a new inference engine.

### Bundled catalog contribution

1. Verify the pinned transcribe-cpp supports the architecture and that the model
   loads with the intended quantization, languages and streaming/translation
   behavior. Do not advertise capabilities on family name alone.
2. Inspect [scripts/gen_catalog.py](../scripts/gen_catalog.py). It reads the
   `handy-computer` HF organization's model-card `transcribe_cpp` metadata and
   benchmarks, probes GGUF display metadata, and applies local `CURATION` /
   `OVERRIDES`. Recommendation badges and ordering are editorial inputs.
3. Regenerate the checked-in catalog using the script's documented environment
   (Python 3.10+, `huggingface_hub`, `fsspec`; `uv run` supports its inline
   dependencies). From the repository root, an explicit output command is
   `uv run scripts/gen_catalog.py src-tauri/src/catalog/catalog.json`.
   Follow the script's authentication guidance if HF access requires it; never
   commit credentials.
4. Review names, revisions, hashes, sizes, languages, capabilities and default
   quant, then exercise download, load and transcription. Validate custom/cache
   discovery and unsupported-file handling if those paths changed.

A new ONNX engine or an unsupported architecture is a code change, not just a
catalog edit: inspect `EngineType`, descriptor/discovery logic, capability
probing and all load/inference/unload paths in `managers/transcription.rs`.
Follow [CONTRIBUTING.md](../CONTRIBUTING.md) and the feature-freeze policy before
proposing engine additions.
