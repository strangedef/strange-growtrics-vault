# Strange Growtrics — Obsidian Vault

This is an Obsidian vault, not a software project. There is nothing to build, lint, or test here — treat it as a knowledge base of engineering/technical documentation for Strange Growtrics and its TaloTrace product.

## Structure

- One folder per topic/area, e.g. `TaloTrace/` — engineering notes for the TaloTrace product.
- Notes are individual Markdown files inside their topic folder (e.g. `TaloTrace/Authentication flow.md`).
- `.obsidian/` holds Obsidian app config (installed plugins, appearance, etc.) — not vault content.

## Note conventions

Follow the style of existing notes when adding or editing content:
- Numbered steps for sequential processes (e.g. an auth flow), with short sub-bullets under each step for detail.
- An "Important:" section at the end for caveats, security notes, or gotchas.
- Plain, direct language — no headers needed for short notes; use headers for longer/multi-section notes.

## When asked to add or edit notes

- Keep the folder-per-topic structure; put new notes in the relevant existing folder or create a new topic folder if it's a new area.
- Match the Markdown conventions above for consistency.
- Do not add build tooling, package files, or CI config — this vault has none and doesn't need any.
