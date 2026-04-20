---
name: vivadicta-cli-usage
description: Run vivadicta commands against the user's VivaDicta for Mac history from the shell. Use when picking a subcommand, flag combo, or output format for the vivadicta CLI.
---

# vivadicta cli usage

Use this skill when you need to run or design a `vivadicta` command. Covers canonical flags, output formats, env vars, exit codes, and the full subcommand map.

## Preconditions

- VivaDicta.app installed at `/Applications/VivaDicta.app`.
- `vivadicta` on `$PATH` (install via `brew install n0an/tap/vivadicta`).
- UI app running is only required for write commands (`rewrite`, `transcribe`, `job`). Reads work standalone.

## Command discovery

Prefer `--help` over guessing:

```bash
vivadicta --help
vivadicta search --help
vivadicta transcribe --help
```

`--help` is authoritative. This skill lists the common shape; anything finer-grained lives behind `--help`.

## Subcommand map

All 11 subcommands, one line each.

Reads (no UI app required):

- `vivadicta recent [N]` - last N transcriptions, newest first. Default `N=10`, capped at 50.
- `vivadicta search <query>` - full-text search. Filters: `--since`, `--from/--to`, `--tag`, `--source`, `--mode`, `--limit` (default 20, max 100), `--full`.
- `vivadicta get <id-or-query>` - one transcription. Accepts raw UUID, keyword (`latest`, `last`, `most recent`, `newest`, `current`), or natural-language query (`"yesterday's call with sam"`). Flags: `--format plain|markdown` (in-summary text), `--all-variations`.
- `vivadicta presets [--all]` - list AI rewrite presets. `--all` includes hidden.
- `vivadicta tags` - list user tags + source tags with counts.
- `vivadicta modes [--all]` - list Viva Modes (app-/URL-aware profiles). `--all` includes disabled.
- `vivadicta vocab` - list custom vocabulary words.
- `vivadicta replacements [--all]` - list post-transcription find-and-replace rules.

Writes (UI app must be running):

- `vivadicta rewrite <id-or-query> --preset <alias>` - rewrite with an AI preset. Blocking. `--dry-run` prints plan without hitting the UI app.
- `vivadicta transcribe <file-or-url> [--async]` - transcribe an audio file or YouTube URL. Auto-routes: `http(s)://` → URL, `file://` / path / `~/` / `./` / bare filename that resolves → file. Blocking default polls every 2 s until done; `--async` returns a job UUID.
- `vivadicta job <uuid>` - single-shot status check for a job from `transcribe --async`.

Modes is **read-only** in v1 - create or edit Viva Modes from the UI app, then `vivadicta modes` surfaces them.

## Global flags

Apply to every subcommand:

- `--output <text|table|json|markdown>` - output format. `markdown` is supported only by `get`. Default is TTY-aware (see next section).
- `--pretty` - pretty-print JSON. Only valid when the effective format is `json`.
- `--quiet` / `-q` - suppress stderr chrome (progress phases, hints).
- `--verbose` / `-v` - include extra fields in human output.
- `--help` / `-h` - per-subcommand help.
- `--version` - print `CFBundleShortVersionString` from the installed app.

## Output format resolution (TTY-aware)

1. Explicit `--output <fmt>` wins.
2. Otherwise `VIVADICTA_OUTPUT=<fmt>` env var.
3. Otherwise stdout TTY detection:
   - TTY → `text`
   - Pipe / redirect → `json`
4. Override detection with `VIVADICTA_FORCE_TTY=1` or `VIVADICTA_FORCE_NON_TTY=1` (CI determinism).

JSON shape matches the matching MCP tool's response. One value differs: `ref` carries a raw UUID from the CLI (vs a session handle like `t1` in MCP). Follow-up `vivadicta` commands accept either.

## Env vars

| Var | Purpose |
|---|---|
| `VIVADICTA_OUTPUT` | Default `--output` format. Overrides TTY detection. |
| `VIVADICTA_FORCE_TTY=1` | Force text/table output even when piped. |
| `VIVADICTA_FORCE_NON_TTY=1` | Force JSON output even on a TTY. |
| `VIVADICTA_TIMEOUT_SECONDS` | Blocking-write poll timeout (default 600). |
| `NO_COLOR=1` | Standard convention, disables ANSI (v1 ships no color anyway). |

## Exit codes

For scripting:

| Exit | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic internal failure |
| 2 | `invalid_arguments` (malformed arg, unsupported flag combination, validation failure) |
| 3 | `preset_not_found` / `preset_ambiguous` (fuzzy match failed or returned multiple candidates) |
| 4 | `transcription_not_found` / `transcription_ambiguous` (resolver failed) |
| 5 | `vivadicta_not_running` (write command with UI app stopped) |
| 6 | `daily_limit_reached` (write only) |
| 10 | `storage_unavailable` (launch the UI app at least once before using reads) |
| 64 | Usage error (unknown subcommand, bad enum value, `--pretty` without `--output json`) |

Branch on these in scripts:

```bash
vivadicta get "$QUERY" --output json > /tmp/out.json
case $? in
  0)  jq '.title' /tmp/out.json ;;
  4)  echo "no match, retry with a UUID" >&2 ;;
  5)  echo "open VivaDicta first" >&2; exit 1 ;;
  10) echo "launch the app at least once" >&2; exit 1 ;;
  *)  echo "unexpected exit" >&2; exit 1 ;;
esac
```

## Output piping

The TTY-aware default makes pipes ergonomic out of the box:

```bash
vivadicta recent 5 | jq '.transcriptions[].title'
vivadicta search "sprint" --since yesterday --output json | jq '.count'
vivadicta get latest --output markdown > ~/notes/today.md
```

In a TTY the same commands produce aligned tables.

## Error output

Errors print to **stderr** with `vivadicta: <message>` + optional `hint:` line. When `--output json` is active, errors become a structured payload for `jq`:

```json
{"status":"error","code":4,"error":"transcription_not_found","message":"No transcription matches '...'","hint":"did you mean ...","candidates":[...]}
```

## Guardrails

- For writes, check that the UI app is running first (`pgrep -f VivaDicta.app` is enough). Otherwise exit 5 is the expected failure; don't retry.
- For reads right after a UI-side edit, expect up to a few seconds of CloudKit sync lag before the new record appears in CLI output.
- If asked to create or edit presets / tags / modes / vocab / replacements: v1 doesn't support this from the CLI. Point the user to the UI app.
- For long-running tasks (`transcribe`, `rewrite`), default to blocking so the user's next step has the result. Pass `--async` only when the user explicitly wants to background the job.
