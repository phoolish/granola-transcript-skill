# granola-transcript-skill

A Claude Code plugin marketplace containing one skill: **`granola-transcript`**.

Verify an AI-generated [Granola](https://granola.ai) meeting summary against the transcript, claim by claim, without loading the full transcript into the working context. Cache the transcript in your repo when that is worth doing.

## Why

AI meeting summaries invent detail. They fabricate figures, misattribute commitments, and merge two speakers into one. They do it confidently, and in ways that read as clean. The transcript is the only ground truth.

A full transcript pull costs 15k to 30k tokens, so verifying against it naively is expensive enough that people stop doing it. This skill makes it cheap two ways:

- **Verify live.** Search inside one meeting's transcript for a few sharp keywords and read only the matching chunks. About 5k tokens a claim, no transcript copy on disk. Every verification leaves one paraphrased row in a ledger file.
- **Fetch once and cache.** The bulky fetch happens **inside a subagent**, which writes the transcript to disk and returns only a small payload (a file path, the attendee list, and a checklist of claims). Verification then greps the on-disk file. Because the transcript is committed, the fetch happens **once ever**.

Either way, a cached file is always checked first.

## Install

```
/plugin marketplace add phoolish/granola-transcript-skill
/plugin install granola-transcript@granola-transcript-skill
```

Or wire it into a project so every clone (including cloud sessions) picks it up, by adding to that repo's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "granola-transcript-skill": {
      "source": { "source": "github", "repo": "phoolish/granola-transcript-skill" }
    }
  },
  "enabledPlugins": { "granola-transcript@granola-transcript-skill": true }
}
```

## Requirements

A Granola MCP server connected to your Claude Code session, exposing `search_meetings`, `get_meetings`, `search_transcripts`, and `get_meeting_transcript` (with `chunk_ids` support for live verification).

## Configuration

The skill defers to the host repo. Declare any of these in your `CLAUDE.md`; a repo that declares nothing gets the defaults, which cache every verified transcript, same as earlier versions.

| Setting | Default | What it does |
|---|---|---|
| Cache directory | `meetings/transcripts/` | Where cached transcripts live. |
| Cache policy | `always` | `always` caches every transcript it verifies. `on-citation` verifies live and caches only when a caching trigger is met or you ask. |
| Caching triggers | see below | Only read under `on-citation`. |
| Ledger file | `verified-live.md` in the cache directory | One row per live-verified meeting. |
| Dual-cache | no | Whether identical transcript bodies under two paths are deliberate. |

Default caching triggers under `on-citation`: a durable note will cite the transcript; the content is sensitive or contractual; someone other than you captured the meeting; or the transcript shows deleted turns or a changed `transcript_version`.

If your `CLAUDE.md` names note conventions, a frontmatter schema, or commit rules, those win over the defaults. If your repo carries its own guidance on verifying sources or handling meeting records, the skill reads that first and treats it as supplementing its own verification rules.

## A note on where transcripts land

Meeting transcripts routinely contain confidential and personal material. The skill caches them into whatever repo it runs in, and commits them. The live ledger paraphrases them too. Point it at a repo whose access boundary actually suits the content.
