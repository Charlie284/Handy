# Handy documentation

Handy records speech and runs recognition locally. Optional LLM post-processing
can send transcript text to a configured service; it is not required for dictation.
See [post-processing](post-processing.md) before enabling a remote provider.

## For users

- [Project README](../README.md): installation, shortcuts, CLI, troubleshooting, and licensing.
- [Settings reference](settings.md): every persisted application setting and its fresh-install default.
- [Models](models.md): recognition engines, downloads, storage, and custom models.
- [Post-processing](post-processing.md): providers, prompts, privacy, and failure behavior.
- [Platform notes](platform-notes.md): permissions, typing tools, overlays, and paste compatibility.

## For contributors

- [Build instructions](../BUILD.md): prerequisites and platform-specific development setup.
- [Contribution guide](../CONTRIBUTING.md): contribution workflow and project policies.
- [Translation guide](../CONTRIBUTING_TRANSLATIONS.md): localization contributions.
- [Architecture](architecture.md): backend/frontend boundaries and recording lifecycle.
- [Model extension guide](models.md#adding-a-model): catalog and runtime integration.

## For maintainers and coding assistants

- [Repository guidance](../AGENTS.md) and [Claude entry point](../CLAUDE.md).
- [Crush assistant notes](../CRUSH.md): additional development commands and coding conventions.
- [Architecture](architecture.md) and [settings persistence](settings.md#persistence-and-compatibility).
- [Pull request template](../.github/PULL_REQUEST_TEMPLATE.md): required PR sections.
- [License](../LICENSE).

These guides describe the checked-in implementation, not a promise that every
engine, device, compositor, or target application behaves identically. Source
links identify the implementation behind each topic. Root build and contribution
guides remain authoritative for setup and contribution policy.
