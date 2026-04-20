---
name: vivadicta-transcribe-flow
description: Transcribe a local audio file or YouTube URL end-to-end with vivadicta, handling blocking vs async, progress, and error recovery. Use when the user wants speech from an audio source saved into their VivaDicta history.
---

# vivadicta transcribe flow

Use this skill to turn an audio file or a YouTube URL into a searchable transcription in the user's VivaDicta history. Covers input-shape detection, blocking vs `--async`, progress output, and recovery paths.

## Preconditions

- VivaDicta.app installed at `/Applications/VivaDicta.app` and **running** (writes require the UI).
- User has a transcription model configured (Whisper local, Parakeet local, or a cloud provider with API key in Keychain).
- For YouTube URLs, the app's YouTube transcription is enabled (one of: captions → Apify → yt-dlp → Whisper).

## Pick the source and invocation

```bash
# Local file, blocking (default) - waits until done, prints the final transcription summary
vivadicta transcribe ~/Downloads/meeting.m4a

# YouTube URL, blocking
vivadicta transcribe "https://youtu.be/dQw4w9WgXcQ"

# Either source, async - returns the job UUID immediately
vivadicta transcribe ~/Downloads/meeting.m4a --async
```

Input auto-routes:

| Input shape | Routes to |
|---|---|
| `http://` / `https://` prefix | `transcribe_url` (YouTube) |
| `file://` prefix | strip scheme, treat as path |
| Absolute path (`/...`), `~/...`, `./`, `../`, or a bare filename that resolves against CWD | `transcribe_file` |
| `-` (stdin) | **rejected** (not supported in v1) |

Ambiguous or nonexistent paths exit 2 with a clear message.

## Blocking vs async

**Blocking (default):** the command waits up to `VIVADICTA_TIMEOUT_SECONDS` (default 600) polling every 2 s, then prints the full transcription summary on stdout (same shape as `vivadicta get`). Progress phase lines stream to stderr: `[vivadicta] downloading captions…` / `[vivadicta] transcribing (parakeet)…` / `[vivadicta] enhancing (claude)…`.

Prefer blocking when:
- The next step in the user's request needs the transcript (rewrite, grep, summarize).
- The audio is short (<1 min) or the transcription model is local.

**Async (`--async`):** returns a job UUID immediately. Poll with `vivadicta job <uuid>` when you want status.

```bash
JOB=$(vivadicta transcribe ~/Downloads/long-meeting.m4a --async --output json | jq -r '.job')
# ... do other work ...
vivadicta job "$JOB"          # one-shot check
```

Prefer async when:
- Audio is long and the user doesn't need the transcript right away.
- You want to dispatch multiple transcriptions in parallel (the UI app serialises by kind; one URL + one file can run concurrently).

## Progress output in JSON mode

When the user asked for JSON (`--output json` or pipe), blocking-mode progress becomes newline-delimited JSON events on **stderr** (stdout stays clean for the final payload):

```json
{"elapsed_ms":0,"event":"phase","message":"Starting transcribe_url…"}
{"elapsed_ms":4120,"event":"phase","message":"Downloading captions"}
{"elapsed_ms":12004,"event":"phase","message":"Enhancing (claude-sonnet-4-6)"}
{"elapsed_ms":18500,"event":"done","message":"Transcription saved."}
```

Scripts can `jq` stderr for live progress.

## Typical end-to-end flow

```bash
# 1. Transcribe and save
vivadicta transcribe "https://youtu.be/dQw4w9WgXcQ"

# 2. If you want to further process, the saved record is now the "latest" - grab its UUID
UUID=$(vivadicta recent 1 --output json | jq -r '.transcriptions[0].ref')

# 3. Rewrite with a preset (see vivadicta-search-and-rewrite)
vivadicta rewrite "$UUID" --preset summary

# 4. Export to Obsidian (see vivadicta-vault-export)
vivadicta get "$UUID" --output markdown > ~/vault/transcripts/$(date +%F).md
```

## Dry-run

Before hitting the UDS, preview the resolved plan:

```bash
vivadicta transcribe ~/Downloads/meeting.m4a --dry-run
# Action  : transcribe (dry-run)
# Route   : transcribe_file
# Resolved: /Users/you/Downloads/meeting.m4a
# Async   : no (blocking)
```

Works with the UI app stopped (no UDS call). Useful for verifying the user's input was parsed correctly before running a long job.

## Error recovery

| Exit | Error | What to do |
|---|---|---|
| 5 | `vivadicta_not_running` | Ask the user to launch VivaDicta, then retry. Don't loop-retry - the UDS socket appears only after launch. |
| 6 | `daily_limit_reached` | User is on the free tier and hit today's AI-enhancement quota. Transcription still completes; enhancement is skipped. Suggest Pro at https://vivadicta.com/pro. |
| 2 | `invalid_arguments` | Usually means the file doesn't exist or the URL scheme is wrong. Rerun with `--dry-run` to see how the input was parsed. |
| 1 | `internal_failure` (on a blocking transcribe) | The underlying job errored in the UI app (auth, network, model load). Check the `error` field in the JSON payload or stderr message. If the failure was transient (network) retry once; if it's consistent (auth) surface the message to the user. |
| timeout | `Timed out after 600s` | Job is still running in the UI. Get the handle from the error message and switch to `vivadicta job <uuid>`. Don't spawn a duplicate. |

## Guardrails

- Never pass `--async` when you're immediately consuming the transcript - the UUID alone is useless for the next step.
- When the user asks "transcribe X", prefer the blocking path so the transcript text is present in the agent's response.
- Don't try to transcribe a format the app doesn't handle (proprietary containers). Stick to audio formats `afinfo` can introspect; the app supports the usual m4a/mp3/wav/flac/aac list.
- If a YouTube URL needs cookies (private video), the CLI can't pass them - the UI app's cookie store is used. Ask the user to open the URL in their default browser first if the job fails with an auth-ish error.
- Progress phases are hints, not SLAs. Don't show exact timestamps to the user unless they asked.

See `vivadicta-cli-usage` for exit-code details and TTY behavior.
