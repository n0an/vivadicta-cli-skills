---
name: vivadicta-vault-export
description: Export a transcription to Obsidian-ready markdown with YAML frontmatter using vivadicta get --output markdown. Use when the user wants a dictation record as a markdown file in their vault or any shell pipeline expecting markdown.
---

# vivadicta vault export

Use this skill to write a VivaDicta transcription into a markdown file suitable for an Obsidian vault, a daily note, a blog draft, or any pipeline that consumes markdown.

## Preconditions

- VivaDicta.app installed at `/Applications/VivaDicta.app`. UI app does **not** need to be running - `get` is a read command.
- The target transcription exists in history. If the user said "today's dictation", confirm with `vivadicta recent 1` first.

## The one flag that matters

`--output markdown` on `get` produces YAML frontmatter + body. This is the Obsidian-ready form. Other subcommands don't support markdown output.

```bash
vivadicta get latest --output markdown
```

Output shape (stdout):

```markdown
---
title: "Call with Sam about the sprint plan"
date: 2026-04-18T14:32:00Z
duration_seconds: 522
duration_display: "08:42"
source: "Mac"
viva_mode: "Coding"
tags: ["sprint-planning", "work"]
transcription_model: "parakeet-tdt-0.6b-v3"
---

# Raw transcript

...full transcript text...

# Latest enhancement (Action Points)

...enhanced text if present...
```

The frontmatter keys are stable and Obsidian-friendly (quotable strings, ISO dates, arrays).

## Common patterns

**Save today's dictation into a dated file:**

```bash
vivadicta get latest --output markdown > ~/vault/transcripts/$(date +%F).md
```

**Save a specific transcription by query:**

```bash
vivadicta get "yesterday's call with sam" --output markdown > ~/vault/calls/sam-$(date +%F).md
```

**Include every AI variation** (Summary, Action Points, Email, …) as separate sections in the body:

```bash
vivadicta get <uuid> --output markdown --all-variations > ~/vault/notes/$(date +%F).md
```

**Pipe to `pbcopy` for a quick paste into an existing note:**

```bash
vivadicta get latest --output markdown | pbcopy
```

**Append to a daily note** (Obsidian Daily Notes plugin format):

```bash
vivadicta get latest --output markdown >> ~/vault/daily/$(date +%F).md
```

## Identifier forms

Same three shapes as `vivadicta-search-and-rewrite`:

- Keyword: `latest`, `last`, `most recent`, `newest`, `current`
- Natural-language query: `"yesterday's call with sam"`
- Raw UUID

When the user says "my last meeting", prefer `latest`. When they describe it, use a natural-language query. When they already have a UUID, pass it through.

## Chain with transcribe or rewrite

**Transcribe then export:**

```bash
vivadicta transcribe ~/Downloads/meeting.m4a
vivadicta get latest --output markdown > ~/vault/transcripts/$(date +%F).md
```

**Rewrite then export** (includes both original + rewritten variation):

```bash
UUID=$(vivadicta recent 1 --output json | jq -r '.transcriptions[0].ref')
vivadicta rewrite "$UUID" --preset summary > /dev/null
vivadicta get "$UUID" --output markdown --all-variations > ~/vault/notes/summary.md
```

## Automation hooks

**Raycast script command** (copy daily summary to clipboard):

```bash
#!/bin/bash
# @raycast.title Copy latest VivaDicta transcript
# @raycast.mode silent
vivadicta get latest --output markdown | pbcopy
```

**Nightly cron digest** - export every transcription from the last 24 h:

```bash
#!/bin/bash
for uuid in $(vivadicta search "" --since "24 hours ago" --output json | jq -r '.transcriptions[].ref'); do
  vivadicta get "$uuid" --output markdown > ~/vault/transcripts/"$uuid.md"
done
```

**Quick shell alias** for a common destination:

```bash
alias viva-to-vault='vivadicta get latest --output markdown > ~/vault/transcripts/$(date +%F).md'
```

## Guardrails

- `--output markdown` only works on `get`. Don't try it on `recent`, `search`, etc. - they error with `invalid_arguments` (exit 2).
- The body's "Latest enhancement" section appears only if the transcription has an `enhancedText`. Use `--all-variations` to include every saved variation explicitly.
- Overwriting vs appending: `>` overwrites, `>>` appends. Ask the user which they want if it's ambiguous (dated filenames are usually fine to overwrite within the same day; daily notes typically want `>>`).
- Tags in the frontmatter come from the user's tag system inside VivaDicta. They match what `vivadicta tags` shows and what Obsidian will pick up as tags automatically.
- Date in the frontmatter is ISO 8601 UTC. If the user wants local time, they'll need to transform it in the Obsidian template.

See `vivadicta-cli-usage` for output-format rules. See `vivadicta-search-and-rewrite` for producing the rewritten content before exporting.
