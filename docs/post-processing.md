# Optional LLM post-processing

Handy's recognition engines run locally. Post-processing is a separate, optional
step that refines recognized **text**. A hosted provider sends that text and
prompt over the network; local recognition does not make remote refinement
offline or private by default. Choose the provider deliberately.

Sources: [actions.rs](../src-tauri/src/actions.rs),
[llm_client.rs](../src-tauri/src/llm_client.rs),
[settings.rs](../src-tauri/src/settings.rs), and
[post-processing UI](../src/components/settings).

## Configuration and invocation

Enable post-processing, choose a provider, supply credentials if required, select
a model and select a prompt. The initial settings do **not** supply a remote
model or selected prompt. Use the separate `transcribe_with_post_process`
shortcut or `handy --toggle-post-process`; ordinary `transcribe` uses the action
without LLM refinement. Defaults and persisted field shapes are listed in the
[settings reference](settings.md#post-processing).

Built-in provider definitions in `default_post_process_providers()`:

| Provider ID          | Default base URL                              | Structured output flag |
| -------------------- | --------------------------------------------- | ---------------------- |
| `openai`             | `https://api.openai.com/v1`                   | Yes                    |
| `zai`                | `https://api.z.ai/api/paas/v4`                | Yes                    |
| `openrouter`         | `https://openrouter.ai/api/v1`                | Yes                    |
| `anthropic`          | `https://api.anthropic.com/v1`                | No                     |
| `groq`               | `https://api.groq.com/openai/v1`              | No                     |
| `cerebras`           | `https://api.cerebras.ai/v1`                  | Yes                    |
| `bedrock_mantle`     | `https://bedrock-mantle.us-east-1.api.aws/v1` | Yes                    |
| `custom`             | `http://localhost:11434/v1`                   | No                     |
| `apple_intelligence` | `apple-intelligence://local`                  | Yes                    |

Apple Intelligence is only added on macOS ARM64. Availability is checked when
used, not assumed from its presence in the list. It uses
[apple_intelligence.rs](../src-tauri/src/apple_intelligence.rs), not an HTTP
request to that pseudo-URL. Its default model is `Apple Intelligence`.

Only Custom permits editing the base URL by default. It can point to an
OpenAI-compatible local service; the localhost default does not start or install
such a server. HTTP provider definitions use `/models` for discovery; Apple has
no model endpoint. Credentials and selected models are stored per provider.
The client includes provider-specific request handling, including Anthropic;
do not assume changing a URL adapts every API protocol.

## Prompts and output handling

The built-in `default_improve_transcriptions` template asks for spelling,
capitalization and punctuation cleanup, number formatting, spoken-punctuation
conversion and filler removal while preserving meaning, word order and language.
It instructs the model not to follow transcript instructions, answer questions,
or invent text for empty input. These are model instructions, not a guarantee
that an LLM cannot change meaning; review important output.

`post_process_transcription()` skips blank input and returns no replacement when
the provider, model, selected prompt or usable prompt text is missing. In the
structured route, `${output}` is removed from the instruction template and the
transcript is sent separately as user text with a response schema. The legacy
route substitutes `${output}` into the prompt. See the branch logic in
[actions.rs](../src-tauri/src/actions.rs) and request/response implementation in
[llm_client.rs](../src-tauri/src/llm_client.rs) before designing custom templates.

Structured request errors fall back to the legacy request. A response with
malformed structured JSON or a missing transcription field is returned as raw
content rather than discarded; review unexpected output rather than assuming
schema validation always protects delivery.

The pipeline uses a returned replacement as final text; when refinement returns
no replacement or is skipped it retains the recognized text. History can retain the original,
post-processed text and prompt used
([history manager](../src-tauri/src/managers/history.rs)). Refinement is not the
same as local custom-word correction or filler filtering, which are implemented
in the [transcription manager](../src-tauri/src/managers/transcription.rs).

## Credentials and troubleshooting

- API keys are serialized in settings. `SecretMap` redacts Rust debug formatting,
  **not** the stored file. Do not share settings backups without removing keys.
- If text is unrefined, check the action invoked, provider, model and selected
  prompt before assuming recognition failed. Inspect logs for request failures.
- A model list request succeeding does not prove that the selected model accepts
  the chat/structured-output request format. Provider flags describe the client
  route, not a guarantee for every model a provider lists.
- Keep sensitive transcripts local by leaving refinement off or deliberately
  configuring a suitable local provider. A Custom URL can also be remote.
