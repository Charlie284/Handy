# Settings reference

This reference covers every field of `AppSettings` in
[settings.rs](../src-tauri/src/settings.rs). Defaults below are **fresh-install**
values from `get_default_settings()`, not a claim about an upgraded store or the
frontend's display labels. Rust types are shown; `Option<T>` serializes as a value
or `null`, vectors as arrays, and maps as JSON objects. Enum values below use
their serialized spellings.

## Persistence and compatibility

Settings are the `settings` object in `settings_store.json` (tauri-plugin-store).
[portable.rs](../src-tauri/src/portable.rs) redirects the store in portable mode.
Use the UI/commands rather than editing a running application's file: commands
also apply runtime side effects such as device, shortcut and overlay updates.
Missing fields receive defaults; missing bindings and built-in providers are
merged. Invalid fields are individually salvaged, and migrations preserve old
overlay-hidden intent. CLI flags are runtime overrides, not preference writes.

`SecretMap` redacts values in Rust debug output but serializes string values into
the store; **do not treat the settings file as encrypted or share it unredacted**.
The frontend [settings store](../src/stores/settingsStore.ts) normalizes absent
device values to its `Default` label and obtains defaults from Rust.

## Application and interface

Usage: [lib.rs](../src-tauri/src/lib.rs), [App.tsx](../src/App.tsx),
[shortcut/mod.rs](../src-tauri/src/shortcut/mod.rs),
[overlay.rs](../src-tauri/src/overlay.rs),
[tray.rs](../src-tauri/src/tray.rs), and
[What's New](../src/components/whats-new).

| Field                         | Rust type                          | Default                                     | Meaning                                                                                            |
| ----------------------------- | ---------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `settings_schema_version`     | `u32`                              | `1`                                         | Internal one-time migration marker; not a user preference.                                         |
| `bindings`                    | `HashMap<String, ShortcutBinding>` | Built-in bindings below                     | Shortcut definitions and current assignments.                                                      |
| `push_to_talk`                | `bool`                             | `true`                                      | Hold to record; false uses press-to-toggle behavior.                                               |
| `start_hidden`                | `bool`                             | `false`                                     | Start without showing settings.                                                                    |
| `autostart_enabled`           | `bool`                             | `false`                                     | Launch at login via autostart integration.                                                         |
| `update_checks_enabled`       | `bool`                             | `true`                                      | Enable update checks.                                                                              |
| `show_whats_new_on_update`    | `bool`                             | `true`                                      | Permit release notes after upgrades.                                                               |
| `whats_new_last_seen_version` | `String`                           | Current package version                     | Release-notes acknowledgment; older stores can be migrated to an empty value.                      |
| `onboarding_completed`        | `bool`                             | `false`                                     | Tracks completion of initial onboarding.                                                           |
| `overlay_position`            | `OverlayPosition`                  | `bottom`                                    | Placement: `top` or `bottom`; visibility is a separate field.                                      |
| `overlay_style`               | `OverlayStyle`                     | Linux `none`; others `live`                 | `none`, `minimal`, or `live`; does not control model streaming.                                    |
| `debug_mode`                  | `bool`                             | `false`                                     | Debug UI and live logging features; CLI `--debug` overrides runtime behavior.                      |
| `log_level`                   | `LogLevel`                         | `debug`                                     | File log threshold: `trace`, `debug`, `info`, `warn`, `error`. Legacy numeric values are accepted. |
| `app_language`                | `String`                           | OS locale with `_` replaced by `-`, or `en` | UI language, separate from recognition language.                                                   |
| `theme`                       | `Theme`                            | `system`                                    | `system`, `light`, or `dark` appearance.                                                           |
| `experimental_enabled`        | `bool`                             | `false`                                     | Enables experimental UI surfaces.                                                                  |
| `keyboard_implementation`     | `KeyboardImplementation`           | Linux `tauri`; others `handy_keys`          | Select shortcut backend.                                                                           |
| `show_tray_icon`              | `bool`                             | `true`                                      | Tray visibility preference; `--no-tray` can override at startup.                                   |

Each `ShortcutBinding` has string fields `id`, `name`, `description`,
`default_binding`, and `current_binding`. Built-ins: `transcribe` is `ctrl+space`
on Windows/Linux and `option+space` on macOS; `transcribe_with_post_process` adds
`shift`; `cancel` is `escape`. Backend/compositor restrictions still apply.

## Audio and recognition

Usage: [audio manager](../src-tauri/src/managers/audio.rs),
[transcription manager](../src-tauri/src/managers/transcription.rs),
[audio feedback](../src-tauri/src/audio_feedback.rs), and
[text processing](../src-tauri/src/audio_toolkit/text.rs).

| Field                         | Rust type                      | Default   | Meaning                                                                                            |
| ----------------------------- | ------------------------------ | --------- | -------------------------------------------------------------------------------------------------- |
| `audio_feedback`              | `bool`                         | `false`   | Play recording feedback sounds.                                                                    |
| `audio_feedback_volume`       | `f32`                          | `1.0`     | Feedback playback volume.                                                                          |
| `sound_theme`                 | `SoundTheme`                   | `marimba` | `marimba`, `pop`, or `custom` feedback assets.                                                     |
| `selected_model`              | `String`                       | `""`      | Selected registry model ID; empty means no selection.                                              |
| `always_on_microphone`        | `bool`                         | `false`   | Keep microphone open between recordings.                                                           |
| `selected_microphone`         | `Option<String>`               | `None`    | Input device selection; absent uses default.                                                       |
| `selected_channel`            | `Option<u16>`                  | `None`    | Input channel; absent averages channels.                                                           |
| `clamshell_microphone`        | `Option<String>`               | `None`    | Alternate input for clamshell/lid-closed handling.                                                 |
| `selected_output_device`      | `Option<String>`               | `None`    | Feedback output device; absent uses default.                                                       |
| `translate_to_english`        | `bool`                         | `false`   | Request English translation where supported by the model.                                          |
| `selected_language`           | `String`                       | `auto`    | Recognition language intent; capability checks constrain choices.                                  |
| `custom_words`                | `Vec<String>`                  | `[]`      | Vocabulary used for custom-word correction.                                                        |
| `model_unload_timeout`        | `ModelUnloadTimeout`           | `min5`    | Idle model unloading policy; variants below.                                                       |
| `word_correction_threshold`   | `f64`                          | `0.18`    | Matching threshold for custom-word correction.                                                     |
| `mute_while_recording`        | `bool`                         | `false`   | Mute system output during recording.                                                               |
| `lazy_stream_close`           | `bool`                         | `false`   | Defer microphone stream closure on stop when applicable.                                           |
| `filler_word_removal_enabled` | `bool`                         | `true`    | Enable filler filtering in transcription output.                                                   |
| `custom_filler_words`         | `Option<Vec<String>>`          | `None`    | Override filler list; absent uses built-in behavior.                                               |
| `transcribe_accelerator`      | `TranscribeAcceleratorSetting` | `auto`    | transcribe-cpp compute preference: `auto`, `cpu`, `gpu`.                                           |
| `ort_accelerator`             | `OrtAcceleratorSetting`        | `auto`    | ONNX preference: `auto`, `cpu`, `cuda`, `directml`, `rocm`; availability depends on build/runtime. |
| `transcribe_gpu_device`       | `i32`                          | `-1`      | transcribe-cpp device index; `-1` means automatic.                                                 |
| `extra_recording_buffer_ms`   | `u64`                          | `0`       | Extra audio collection delay at stop before closing the recording.                                 |
| `vad_enabled`                 | `bool`                         | `true`    | Enable voice-activity filtering rather than retaining silence.                                     |

`ModelUnloadTimeout`: `never`, `immediately`, `min2`, `min5`, `min10`, `min15`,
`hour1`, and debug-only `sec15`. Accelerator settings are preferences, not a
guarantee that a given driver or engine supports the selected hardware.

## History and delivery

Usage: [history manager](../src-tauri/src/managers/history.rs),
[clipboard.rs](../src-tauri/src/clipboard.rs),
[input.rs](../src-tauri/src/input.rs), and
[paste_tx](../src-tauri/src/paste_tx/mod.rs).

| Field                        | Rust type                  | Default                         | Meaning                                                                                                       |
| ---------------------------- | -------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `history_limit`              | `usize`                    | `5`                             | Count limit used by `preserve_limit` recording cleanup.                                                       |
| `recording_retention_period` | `RecordingRetentionPeriod` | `preserve_limit`                | Cleanup policy: `never` means never delete; other choices are `preserve_limit`, `days3`, `weeks2`, `months3`. |
| `paste_method`               | `PasteMethod`              | Linux `direct`; others `ctrl_v` | Delivery strategy; see platform notes.                                                                        |
| `clipboard_handling`         | `ClipboardHandling`        | `dont_modify`                   | Preserve previous clipboard or leave transcript with `copy_to_clipboard`.                                     |
| `auto_submit`                | `bool`                     | `false`                         | Send configured submit key after delivery. Use cautiously in messaging/terminal apps.                         |
| `auto_submit_key`            | `AutoSubmitKey`            | `enter`                         | `enter`, `ctrl_enter`, or `cmd_enter`.                                                                        |
| `append_trailing_space`      | `bool`                     | `false`                         | Append a space to delivered text.                                                                             |
| `paste_delay_ms`             | `u64`                      | `60`                            | Pre-paste clipboard/injection delay in milliseconds.                                                          |
| `paste_delay_after_ms`       | `u64`                      | `60`                            | Post-keystroke wait before legacy clipboard restoration.                                                      |
| `reliable_paste`             | `bool`                     | `false`                         | Debug-gated macOS/Windows receipt-sequenced clipboard restoration.                                            |
| `typing_tool`                | `TypingTool`               | `auto`                          | Linux tool selection: `auto`, `wtype`, `kwtype`, `dotool`, `ydotool`, `xdotool`.                              |
| `external_script_path`       | `Option<String>`           | `None`                          | Script used by `external_script` paste method.                                                                |

`PasteMethod`: `ctrl_v`, `direct`, `none`, `shift_insert`, `ctrl_shift_v`,
`external_script`. These are not equally supported by every target application.
Cleanup removes eligible unsaved history database entries and their WAV files;
saved entries are excluded. This is not a secure-erasure guarantee. See
[platform notes](platform-notes.md) for delivery details.

## Post-processing

Usage: [actions.rs](../src-tauri/src/actions.rs),
[llm_client.rs](../src-tauri/src/llm_client.rs), and
[post-processing guide](post-processing.md).

| Field                             | Rust type                  | Default                                                                   | Meaning                                                        |
| --------------------------------- | -------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `post_process_enabled`            | `bool`                     | `false`                                                                   | Enables the post-processing shortcut/configuration.            |
| `post_process_provider_id`        | `String`                   | `openai`                                                                  | Active provider ID.                                            |
| `post_process_providers`          | `Vec<PostProcessProvider>` | Built-in provider list                                                    | Provider URLs, labels and protocol capability metadata.        |
| `post_process_api_keys`           | `SecretMap`                | Each built-in ID → `""`                                                   | Provider credentials; serialized strings, debug-redacted only. |
| `post_process_models`             | `HashMap<String, String>`  | Each built-in ID → `""`, except Apple Intelligence → `Apple Intelligence` | Selected model per provider.                                   |
| `post_process_prompts`            | `Vec<LLMPrompt>`           | One `default_improve_transcriptions` prompt                               | Editable refinement templates.                                 |
| `post_process_selected_prompt_id` | `Option<String>`           | `None`                                                                    | Selected prompt ID; configure one before expecting refinement. |

`PostProcessProvider` fields: string `id`, `label`, `base_url`; boolean
`allow_base_url_edit`, `supports_structured_output`; optional string
`models_endpoint`. `LLMPrompt` fields are strings `id`, `name`, `prompt`.
The built-in prompt requests spelling/punctuation cleanup, numeric formatting,
spoken punctuation conversion, filler removal, preservation of original language
and meaning, and text-only output. Exact defaults, URLs and platform-gated
providers are in `default_post_process_providers()` / `default_post_process_prompts()`
in [settings.rs](../src-tauri/src/settings.rs).
