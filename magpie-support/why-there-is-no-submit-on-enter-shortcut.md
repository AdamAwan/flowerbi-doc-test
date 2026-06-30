---
title: Why There Is No 'Submit on Enter' Shortcut
status: draft
---

# Why There Is No 'Submit on Enter' Shortcut

The Markdown Magpie web console does not map the `Enter` key to submit a question. This is an intentional design choice that follows the convention of multi-line text inputs.

## The ask input is a text area

The question box in the web UI is an HTML `<textarea>`, not a single-line `<input>`. A text area supports long, formatted questions that may include line breaks for readability. Pressing `Enter` inside a text area inserts a newline rather than submitting the form.

To submit a question, use:

- **Ctrl+Enter** (Windows/Linux)
- **Cmd+Enter** (macOS)
- Or click the **submit button** next to the input field.

This behaviour is standard across chat and documentation interfaces (e.g., Slack, GitHub Issues, ChatGPT) and prevents accidental submission while typing a multi-line question.

## Keyboard shortcuts are contextual

The Markdown Magpie pitch deck (`presentation/index.html`) defines keyboard navigation shortcuts ( →, Space, PageDown, etc.) for advancing slides. These shortcuts are **not** part of the main web console. The console follows standard browser form behaviour, and no separate shortcut configuration is exposed in the current UI.

If you would like to request a configurable shortcut or a “submit on Enter (single‑line)” mode, please open a feature request in the repository.

## Related resources

- [Pitch Deck README – Keyboard Navigation](https://github.com/AdamAwan/markdown-magpie/blob/main/presentation/README.md) – describes slide navigation shortcuts.
- [Markdown Magpie Web Console](https://github.com/AdamAwan/markdown-magpie/tree/main/apps/web) – the Next.js application that hosts the ask interface.