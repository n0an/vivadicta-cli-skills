---
name: vivadicta-search-and-rewrite
description: Find a past transcription by natural-language query or keyword and rewrite it with an AI preset using vivadicta. Use when the user wants to turn existing dictation into action items, an email, a summary, or any other preset output.
---

# vivadicta search and rewrite

Use this skill to locate an existing transcription in the user's VivaDicta history and apply an AI preset to it. Covers natural-language IDs, preset resolution, and the end-to-end flow.

## Preconditions

- VivaDicta.app installed at `/Applications/VivaDicta.app` and **running** (rewrite requires the UI).
- User has an AI provider configured with an API key (keychain) or a local MLX model loaded.
- Some history exists. If `vivadicta recent 1` is empty, there's nothing to rewrite.

## Identifier forms

`rewrite` (and `get`) accept three identifier shapes. Prefer in this order:

1. **Keyword:** when the user clearly means the newest entry.
   ```bash
   vivadicta rewrite latest --preset summary
   # also: last, most recent, newest, current
   ```
2. **Natural-language query:** when the user describes the transcription.
   ```bash
   vivadicta rewrite "yesterday's call with sam" --preset action_points
   vivadicta rewrite "coding session this morning" --preset professional
   ```
3. **Raw UUID:** when you already have an ID from a prior command.
   ```bash
   vivadicta rewrite 1701DB3C-AF1F-4191-A791-5DABA81D8096 --preset email
   ```

Session handles (`t1`, `t2`) from MCP don't carry across one-shot CLI invocations - use the UUID instead.

## Resolver behavior

The natural-language matcher:

- Splits the query into tokens (≥3 chars), lowercases, and searches `text + enhancedText + powerModeName`.
- Also extracts a timeframe hint ("yesterday", "last week") and filters by it.
- Ranks candidates; if the top score is ≥3x the second score, picks the top. Otherwise returns `transcription_ambiguous` (exit 4) with the top 5 previews on stderr.

When ambiguous, the error lists candidates. Re-run with a more specific phrase or with a raw UUID from the candidates list.

## Preset resolution

Preset aliases are fuzzy-matched too: `summary` / `Summary` / `sumary` all resolve to the Summary preset. `rewrite` returns `preset_ambiguous` (exit 3) with candidates if the match is unclear.

Discover available aliases:

```bash
vivadicta presets --output json | jq -r '.presets[] | "\(.alias)\t\(.name)"'
# regular           Regular
# summary           Summary
# action_points     Action Points
# email             Email
# ...
```

Built-in aliases include: `regular`, `summary`, `action_points`, `takeaways`, `mind_map`, `key_points`, `professional`, `casual`, `email`, `chat`, `coding`, `rewrite`, `short`, `elaborated`, `simplify`, `translate_en`, `translate_ru`, `translate_es`, `translate_zh`, plus more translate_* variants and user-created presets (typically prefixed `custom_`).

## End-to-end flow

```bash
# Discover the transcription
vivadicta search "sprint planning" --since "last week" --limit 5

# If exactly one matches, rewrite directly with a natural-language query
vivadicta rewrite "sprint planning last week" --preset action_points

# If multiple, grab the UUID from the search output
UUID=$(vivadicta search "sprint planning" --limit 1 --output json | jq -r '.transcriptions[0].ref')
vivadicta rewrite "$UUID" --preset action_points
```

The rewrite result prints the original + rewritten text. The rewritten version is also saved as a `TranscriptionVariation` on the record - follow-up `vivadicta get <uuid> --all-variations` will include it.

## Dry-run

Preview the resolved transcription + preset without hitting the UI app:

```bash
vivadicta rewrite "yesterday's call" --preset summary --dry-run
# Action       : rewrite (dry-run)
# Transcription: 1701DB3C-AF1F-4191-A791-5DABA81D8096
# Title        : Call with Sam about the sprint plan
# Preset       : summary
```

Useful for confirming the resolver picked the right transcription + preset before the AI call.

## Common patterns

**"Make action items from today's call":**

```bash
vivadicta rewrite latest --preset action_points
```

**"Rewrite my dictation as a professional email":**

```bash
vivadicta rewrite latest --preset email --output markdown
```

**"Translate yesterday's note to Russian":**

```bash
vivadicta rewrite "yesterday's note" --preset translate_ru
```

**Chain with export** (see `vivadicta-vault-export`):

```bash
UUID=$(vivadicta recent 1 --output json | jq -r '.transcriptions[0].ref')
vivadicta rewrite "$UUID" --preset summary > /dev/null
vivadicta get "$UUID" --output markdown --all-variations > ~/vault/summary.md
```

## Error recovery

| Exit | Error | What to do |
|---|---|---|
| 4 | `transcription_not_found` | Broaden the query or drop the timeframe filter. Confirm there's history with `vivadicta recent 5`. |
| 4 | `transcription_ambiguous` | Error lists candidates. Re-run with a more specific phrase or a UUID from the candidates. |
| 3 | `preset_not_found` / `preset_ambiguous` | Run `vivadicta presets` and pick an alias from the list. |
| 5 | `vivadicta_not_running` | Launch VivaDicta, then retry. |
| 6 | `daily_limit_reached` | Free-tier quota reached. Offer https://vivadicta.com/pro. |

## Guardrails

- Don't spam `rewrite` in a retry loop - each call is a paid AI request.
- If the user didn't name a preset, ask rather than defaulting. "summary" vs "action_points" vs "email" produce different outputs; the wrong choice wastes a call.
- For regeneration of the same preset the user already applied, `rewrite` will re-run the AI (not cached). Call this out if it seems unintended.
- When presenting the result to the user, show both the preset used and the UUID so they can re-run or reference it.

See `vivadicta-cli-usage` for exit codes and output formats. See `vivadicta-vault-export` for piping rewrites into Obsidian.
