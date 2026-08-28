---
name: granola-transcript
description: Pull a Granola meeting transcript into the repo, cache it, and verify the AI summary against it claim by claim. Use when asked to pull, cache, import, re-pull, refresh, or verify a meeting transcript, to check a summary's figures, dates or attributions against what was actually said, or to publish or share a cached transcript to main or into other branches. Also use when another skill needs a transcript fetched and checked. Fetches the bulky transcript inside a subagent so it never loads into the main context, writes it to a cache directory, and works from the cached file afterwards. Re-pulls on request, or when a meeting was resumed or extended after the last pull.
---

# Granola transcript

Fetch a meeting transcript once, cache it in the repo, and verify the AI-generated summary against it — without ever loading the full transcript into the working context.

The principle the whole skill serves: **meeting summaries are hypotheses; transcripts are ground truth.** Verify every figure, attribution, date and named commitment against the raw transcript before it goes into a note or in front of a person.

## Why this exists

`get_meeting_transcript` returns the whole transcript (15k–30k tokens for a long meeting) straight into context. Loading that every time you want to check one figure is the expensive way to hold that principle.

The cheap way: fetch once inside a subagent that writes the transcript to disk and hands back only a small payload; keep the summary in context as a checklist; then grep the on-disk file for each claim. Because the transcript is committed, the fetch happens once ever — every later session on any machine reads a local file for free.

## Configuration

The skill needs one thing from the host repo: **where cached transcripts live.** Default `meetings/transcripts/`.

If the host repo's `CLAUDE.md` (or equivalent) names a different cache directory, note conventions, frontmatter schema, or commit rules, those win over the defaults here. Check for them before writing anything. Likewise, if the repo carries its own guidance on verifying sources or handling meeting records, read it first — it supplements the verification rules below rather than being replaced by them.

## Procedure

### 1. Resolve the meeting, and decide whether to fetch

Use `list_meetings` with `time_range: custom` and explicit `custom_start` / `custom_end`, or `query_granola_meetings` for **ID discovery only**. Allow ~90 minutes of sync lag after a meeting ends before concluding it was not recorded.

Then decide, based on the cache at `<cache-dir>/YYYY-MM-DD-<slug>.md`:

- **Not cached yet.** Fetch it (step 2).
- **Cached, and you only need to read it.** Skip the fetch; read the summary section off disk. That is the once-ever payoff.
- **Cached, but a re-pull is called for.** Re-fetch and overwrite. Re-pull whenever the user asks ("pull it again", "refresh the transcript"), and whenever the meeting could have changed since the cached `pulled:` date: it was adjourned and resumed, ran long, or had context added afterwards. **Granola keeps appending to the same meeting**, so an early pull can be a truncated version of what is there now. If you cannot tell whether it grew, re-pull; the cost is one fetch.

A re-pull overwrites the cache file and updates `pulled:`. **If the transcript grew, re-verify.** The distilled note was checked against the shorter version and may now be missing or contradicting later content. Say so rather than assuming the old distillation still holds.

### 2. Fetch in an isolated subagent

Spawn a subagent for the fetch so the transcript never enters the main context. Its whole job:

- `get_meetings` for the summary and metadata, `get_meeting_transcript` for the verbatim text.
- Apply any name-normalisation aliases the host repo defines to the transcript and attendee list before writing.
- Write the cache file (format below), stamping `pulled:` with today's date. On a re-pull, overwrite in place and update `pulled:`.
- Return **only**: the file path, the attendee list, the summary verbatim, and a **claim checklist** — one line per figure, attribution, date, and named commitment in the summary.

The subagent's context absorbs the bulk. The main context gets back a small payload it can work from.

### 3. Verify claim by claim, against the file

For each item on the checklist, `grep` the cached transcript for the figure or name and read a small window around the hit. Mark each:

- **confirmed** — in the transcript as stated
- **clarified** — present but different
- **refuted** — contradicted
- **unlocatable** — not found, which means the summary invented the detail

Verification rules:

- **Attribute from content, not just speaker labels.** Diarisation mislabels speakers, and it mislabels them in ways that look clean. For anything about who owns, asked for, or committed to something, resolve it from what is actually said and **write the tell into the note**.
- **Resolve bare first names against the attendee list**, not against whoever is usually discussed.
- **Log third-party claims as "X said Y", never as "Y".**
- **Money and contractual figures get a harder test.** A transcript proves someone said a number, not that the number is right. Check those against the authoritative paperwork, not the transcript.

### 4. Distil into a note

Write the verified output to its proper home in the host repo — a meeting note, a 1:1 log, a person note, whatever that repo uses. **Not** the cache file, which stays raw ground truth.

The distilled note carries:

- A review marker until a human has checked it (e.g. `status: needs-review` in frontmatter, plus a visible callout).
- A `source:` naming the cache file and the `granola_id`.
- A `Summary says / Transcript says / verdict` table for anything clarified, refuted, or unlocatable.

### 5. Commit the transcript and the note as two separate commits

The transcript cache is shared ground truth; the distilled note is branch-local work still under review. Keep them in **two commits** so the transcript can travel to main and to other branches on its own, without dragging the unreviewed note along.

Check `git diff --cached --name-only` before each commit, and scope every commit with `git commit -- <paths>`:

1. **The transcript first, alone.** `git commit -- <cache-dir>/YYYY-MM-DD-<slug>.md` — that single file and nothing else. This is the portable unit that step 6 and the sharing section move around; keeping it to one file is what makes it cherry-pick cleanly with no chance of conflict. Note its SHA — the next steps replay it.
2. **The distilled note second**, in its own `git commit -- <note paths>`. This one stays on the feature branch until it has been reviewed.

### 6. Publish the transcript to main

A cached transcript is ground truth every branch may need, so it should not wait on the feature branch merging. Land the transcript commit on `main` by itself as soon as it is cached:

1. `git fetch origin main`.
2. Cut a throwaway branch from current main and cherry-pick **only** the transcript commit:
   `git switch -c transcripts/YYYY-MM-DD-<slug> origin/main`, then `git cherry-pick <transcript-sha>`.
3. If the cache directory has an index/README, add this transcript's one-line entry there and commit it separately. Cut fresh from main, it cannot conflict.
4. `git push -u origin transcripts/YYYY-MM-DD-<slug>`.
5. Open a PR against `main` and merge it. The raw transcript is immutable, so there is nothing to gate on; the PR exists for the audit trail, not review. **If `main` is protected or the repo's conventions require review, follow those instead of self-merging.**
6. `git switch -` back to the feature branch, and delete the throwaway branch once merged.

Only the raw transcript and its index entry go to main. The distilled note is **not** in this PR — the interpretation stays on the feature branch under review. If a re-pull changed the transcript, publish the newer file the same way.

## Sharing a cached transcript into other branches

Publishing to main reaches every *new* branch, and any branch that later syncs main. It does not reach back into branches that already diverged. When the user names specific existing branches that need a transcript now, cherry-pick the transcript commit onto each — and nothing else.

For each branch named:

1. `git fetch origin <branch> main`.
2. `git switch -C _share origin/<branch>`.
3. `git cherry-pick <transcript-sha>` — the same commit that went to main. If the branch already carries it, the cherry-pick comes up empty; skip that branch.
4. `git push origin _share:<branch>`.
5. `git switch -` back, and delete `_share`.

Only the transcript file rides along, so the branch's own PR stays about its own work. Because the identical commit is on main too, git dedupes with no conflict when the branch eventually merges, and it drops out of that branch's diff the first time the branch syncs main.

**Never fan out to a branch the user did not name.**

## The cache file format

Path: `<cache-dir>/YYYY-MM-DD-<slug>.md`. The small part goes first so it is cheap to read; the bulk is grep-only.

```yaml
---
title: "Transcript: <topic>"
date: YYYY-MM-DD
type: transcript
granola_id: <uuid>
pulled: YYYY-MM-DD        # when last fetched; bump on re-pull
attendees: [<person>, ...]
status: raw
---
```

Match the host repo's frontmatter and tagging conventions where it has them.

Then:

- `## Summary (AI-generated, hypothesis, unverified)` — the summary verbatim, in a warning callout.
- `## Transcript` — verbatim, **speaker labels left intact.** The labels are what attribution depends on, so do not strip them.

## Constraints

- The cache file is ground truth, never a distilled note. Do not edit the transcript text or "tidy" it. Corrections belong in the distilled note, with a source.
- An unlocatable claim is a finding, not a gap to fill. Say the summary asserted it and the transcript does not carry it.
- Do not load the full transcript into the main context to "double check". If grep did not settle it, read a wider window — still on the file.
- A re-pull is the only thing that should overwrite a cache file, and only with a fresh fetch of the same meeting. Never hand-edit a transcript to "update" it.
- Transcripts can contain confidential and personal material. Cache them only in a repo whose access boundary suits the content, and keep the distilled note in draft until a human has reviewed it.
