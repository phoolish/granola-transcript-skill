# granola-transcript-skill

A Claude Code plugin marketplace containing one skill: **`granola-transcript`**.

Fetch a [Granola](https://granola.ai) meeting transcript once, cache it in your repo, and verify the AI-generated summary against it claim by claim — without ever loading the full transcript into the working context.

## Why

AI meeting summaries invent detail. They fabricate figures, misattribute commitments, and merge two speakers into one — confidently, and in ways that read as clean. The transcript is the only ground truth.

But fetching a transcript costs 15–30k tokens, so verifying against it naively is expensive enough that people stop doing it. This skill makes it cheap:

- The bulky fetch happens **inside a subagent**, which writes the transcript to disk and returns only a small payload — a file path, the attendee list, and a checklist of claims to check.
- Verification runs by **grepping the on-disk file**, one claim at a time.
- Because the transcript is committed, the fetch happens **once ever**. Every later session reads a local file for free.

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

A Granola MCP server connected to your Claude Code session, exposing `list_meetings`, `get_meetings`, and `get_meeting_transcript`.

## Configuration

One setting: **where cached transcripts live**, defaulting to `meetings/transcripts/`.

The skill defers to the host repo. If your `CLAUDE.md` names a different cache directory, note conventions, frontmatter schema, or commit rules, those win over the defaults. If your repo carries its own guidance on verifying sources or handling meeting records, the skill reads that first and treats it as supplementing its own verification rules.

## A note on where transcripts land

Meeting transcripts routinely contain confidential and personal material. The skill caches them into whatever repo it runs in, and commits them. Point it at a repo whose access boundary actually suits the content.
