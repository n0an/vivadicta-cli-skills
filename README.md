# vivadicta cli skills

Agent skills for [VivaDicta for Mac](https://vivadicta.com)'s command-line interface (`vivadicta`). Drop them into any stdio agent - Claude Code, Cursor, Codex, Goose, Warp, and [~40 more](https://github.com/vercel-labs/skills) - so the agent knows when and how to call the CLI for the user's dictation workflows.

Official pack maintained alongside the `vivadicta` CLI.

## Installation

Install the full pack globally (recommended):

```bash
npx skills add n0an/vivadicta-cli-skills --global
```

Install a single skill for a specific agent:

```bash
npx skills add n0an/vivadicta-cli-skills --skill vivadicta-transcribe-flow --agent claude-code
```

Preview what's in the pack without installing:

```bash
npx skills add n0an/vivadicta-cli-skills --list
```

Manage installed skills:

```bash
skills list             # see what's installed
skills update           # refresh to latest
skills remove           # remove a skill
```

The pack installs prose-only SKILL.md files; there's no runtime. Updating to a new skill-pack version is as lightweight as `git pull` on a dotfile.

## Prerequisites

Before these skills are useful, make sure the CLI itself is installed and reachable:

```bash
# Install the VivaDicta app (notarized DMG) from https://vivadicta.com first, then:
brew install n0an/tap/vivadicta
vivadicta recent 5
```

Write skills (`vivadicta-transcribe-flow`, `vivadicta-search-and-rewrite`) also require the VivaDicta UI app to be running, since writes route through a Unix socket into the running app.

## Available Skills

### vivadicta-cli-usage

Reference skill: canonical flags, output formats, env vars, exit codes, TTY behavior, and a map of every subcommand.

**Use when:**
- You need the correct `vivadicta` command, flag, or output format.
- You need to interpret an exit code or error payload.
- You want the discovery cheat sheet for the 11 subcommands.

**Example:**

```text
What vivadicta command lists my last 20 transcriptions as JSON and pipes the titles through jq?
```

### vivadicta-transcribe-flow

End-to-end workflow for transcribing a local audio file or a YouTube URL into the user's history. Handles input-shape detection, blocking vs `--async`, progress output, and error recovery.

**Use when:**
- The user wants to transcribe an audio file (m4a, mp3, wav, etc.).
- The user wants to transcribe a YouTube URL.
- You need to decide between blocking and async, or recover from a timed-out / failed transcription.

**Example:**

```text
Transcribe ~/Downloads/standup.m4a and give me the result.
```

### vivadicta-search-and-rewrite

Find a past transcription via natural-language query or the `latest` keyword, then rewrite it with an AI preset (summary, action points, email, translation, etc.).

**Use when:**
- The user wants to turn an existing dictation into action items, an email, a summary, or any other preset output.
- You need to resolve natural-language queries ("yesterday's call with sam") to a specific transcription.
- The user asks about available presets or gets an ambiguous-preset error.

**Example:**

```text
Rewrite yesterday's call with Sam as action items.
```

### vivadicta-vault-export

Export a transcription to Obsidian-ready markdown (YAML frontmatter + body) and pipe it into a vault, daily note, Raycast action, or cron digest.

**Use when:**
- The user wants a dictation record as a markdown file in their Obsidian vault.
- You need to append a transcript to a daily note or copy it to the clipboard.
- You're building a shell / Raycast / cron automation that consumes transcripts as markdown.

**Example:**

```text
Export my last dictation to ~/vault/transcripts as a dated markdown file.
```

## Versioning

Tagged milestones (`v1.0.0`, `v1.1.0`, …) for users who want to pin. The default branch (`main`) always reflects the latest guidance; day-to-day edits land without a tag bump.

To pin to a specific version:

```bash
npx skills add n0an/vivadicta-cli-skills@v1.0.0
```

## Related

- [VivaDicta for Mac](https://vivadicta.com) - the app itself, required for the CLI to do anything meaningful.
- [n0an/vivadicta-cli](https://github.com/n0an/vivadicta-cli) - the thin bash shim distributed via Homebrew that becomes `vivadicta` on `$PATH`.
- [n0an/vivadicta-mcp](https://github.com/n0an/vivadicta-mcp) - the parallel shim for the MCP server (`vivadicta-mcp`), a separate install path for MCP clients.
- [n0an/homebrew-tap](https://github.com/n0an/homebrew-tap) - formula source for both shims.

## License

MIT. See `LICENSE`.
