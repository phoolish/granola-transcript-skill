---
name: granola-transcript
description: Verify a Granola meeting summary against the transcript claim by claim, live or from a cached copy, and pull, cache, re-pull or publish transcripts when the host repo's cache policy calls for it. Use when asked to verify this summary, check this claim, or check a summary's figures, dates or attributions against what was actually said; when asked to pull, cache, import, re-pull or refresh a meeting transcript; or when asked to publish or share a cached transcript to main or into other branches. Also use when another skill needs a meeting summary checked. Reads a cached file first if one exists, verifies live by searching inside the transcript when the policy allows, and otherwise fetches the transcript inside a subagent so it never loads into the main context.
---

# Granola transcript

Verify an AI-generated meeting summary against what was actually said, claim by claim, without loading the full transcript into the working context. Cache the transcript in the repo when that is worth doing.

The principle the whole skill serves: **meeting summaries are hypotheses; transcripts are ground truth.** Verify every figure, attribution, date and named commitment against the raw transcript before it goes into a note or in front of a person.

## Why this exists

A full `get_meeting_transcript` pull costs 15k to 30k tokens for a long meeting. Loading that every time you want to check one figure is the expensive way to hold that principle.

There are two cheaper ways, and the skill uses both:

- **Verify live.** `search_transcripts` finds the passages inside one meeting's transcript that match a few keywords, and `get_meeting_transcript` with `chunk_ids` reads just those chunks. A claim costs about 5k tokens and no transcript copy lands on disk (just one paraphrased ledger row per meeting).
- **Fetch once and cache.** A subagent pulls the whole transcript, writes it to the repo and hands back a small payload. Every later session on any machine greps a local file for free.

Which one runs is the host repo's call, through the cache policy below.

## Configuration

Everything here is read from the host repo's `CLAUDE.md` (or equivalent). A repo that declares nothing gets the defaults, and the defaults behave exactly as the skill always has: every verified transcript is cached.

- **Cache directory.** Where cached transcripts live. Default `meetings/transcripts/`.
- **Cache policy.** One of:
  - `always` (default). Every transcript the skill verifies is fetched and cached first, then verified from the file.
  - `on-citation`. Verify live by default. Fetch and cache only when one of the host repo's caching triggers is met, or the user asks.
- **Caching triggers** (only read under `on-citation`). The host repo lists its own. If it names none, use these:
  - a durable note (meeting note, person note, decision record) will cite the transcript as its source
  - the content is sensitive or contractual (money, terms, commitments someone may be held to, HR matters)
  - the meeting was captured by someone other than the user, so it can vanish from their account
  - the transcript shows deleted turns, or its `transcript_version` changed since it was last checked
- **Ledger file.** Where live verifications are logged. Default `verified-live.md` inside the cache directory.
- **Dual-cache policy.** Whether the repo deliberately stores the same transcript body under more than one path (per-branch or per-topic copies). Default: no. The bulk integrity pass treats two identical transcript bodies as corruption unless the repo declares this.

If the host repo names note conventions, a frontmatter schema, name-normalisation aliases or commit rules, those win over the defaults here. Check for them before writing anything. Likewise, if the repo carries its own guidance on verifying sources or handling meeting records, read it first. It supplements the verification rules below rather than being replaced by them.

## Granola tools

Check the live tool schemas before relying on any argument here. As of this version:

- `search_meetings`: resolve a meeting by keyword, participant or date range. Returns metadata and IDs, not content.
- `get_meetings`: summary, attendees and metadata for up to ten meeting IDs.
- `search_transcripts`: ranked passages inside transcripts. Pass `meeting_ids` and `keywords`. Each hit carries `chunk_ids` and a `transcript_version`.
- `get_meeting_transcript`: with `chunk_ids` plus `transcript_version`, reads those chunks (and returns neighbour IDs). Without `chunk_ids`, a full pull, paged by `max_chars` and `next_offset`.

## Procedure

### 1. Resolve the meeting, then decide how to verify it

Resolve the meeting with `search_meetings`: a date range (`from_date` / `to_date`) plus keywords or a participant. Use it for ID discovery only, then `get_meetings` for the summary. Allow about 90 minutes of sync lag after a meeting ends before concluding it was not recorded. If `get_meeting_transcript` reports `transcript_pending`, wait and retry; `not_happened_yet` means retry after the meeting.

Then make a three-way decision. **Always check for a cached file first**, at `<cache-dir>/YYYY-MM-DD-<slug>.md` (or by `granola_id` across the cache directory), whatever the policy.

1. **Cached.** Read the file. Skip the fetch. That is the once-ever payoff. Re-pull instead (route 3) when:
   - the user asks ("pull it again", "refresh the transcript")
   - the live `transcript_version` differs from the one in the file's frontmatter. One `search_transcripts` call on that meeting returns the current version cheaply.
   - the file predates `transcript_version` tracking and the meeting could have grown since `pulled:` (adjourned and resumed, ran long, context added later). **Granola keeps appending to the same meeting**, so an early pull can be a truncated version of what is there now. If you cannot tell, re-pull; the cost is one fetch.
2. **Verify live.** Not cached, the policy is `on-citation`, no caching trigger is met, and the user has not asked for a copy. Go to step 3 against the live backend. If a trigger turns up partway through (the transcript shows deleted turns, the claims turn out to be contractual, the user decides a note will cite it), stop and switch to route 3.
3. **Fetch and cache.** Not cached and the policy is `always`; or a caching trigger is met; or the user asked; or a re-pull is called for. Go to step 2, then verify against the file.

If the meeting is already in the ledger, check its row before verifying live. Same `transcript_version` and the claim already checked: reuse the finding. Version changed: re-verify, and update the row.

A re-pull overwrites the cache file and updates `pulled:` and `transcript_version:`. **If the transcript changed, re-verify.** The distilled note was checked against the older version and may now be missing or contradicting later content. Say so rather than assuming the old distillation still holds.

### 2. Fetch in an isolated subagent

Spawn a subagent for the fetch so the transcript never enters the main context. Its whole job:

- **Work in a private scratch directory.** Concurrent fetches that share a directory and pick the same obvious names (`raw.json`, `build.py`, `summary.txt`) overwrite each other mid-task, and the result is a cache file with correct frontmatter and `granola_id` over a **different** meeting's transcript. That is invisible to every later check, because grep then answers confidently from the wrong meeting. Each subagent creates its own scratch directory and writes **every** intermediate file there. Never write intermediates to a shared path.
- `get_meetings` for the summary and metadata, `get_meeting_transcript` (no `chunk_ids`) for the verbatim text. A full pull is paged: follow `next_offset` until the transcript is complete, keeping the same `include_timestamps` value on every page. Record the `transcript_version` returned.
- Apply any name-normalisation aliases the host repo defines to the transcript and attendee list before writing.
- **Keep deleted turns as they are.** When a user trims a recording, Granola can return individual turns as `Speaker X: [DELETED]`. Write those markers verbatim. They are not a failed fetch and not a reason to retry. Report that the transcript has deleted turns so the index can say so. (This is different from `transcript_deleted`, which means the whole transcript is gone: outcome EMPTY.)
- Write the cache file (format below), stamping `pulled:` with today's date and `transcript_version:` with the version fetched. On a re-pull, overwrite in place and update both.
- **Prove the file is the right meeting before reporting done.** Re-read the cache file just written; confirm its `granola_id` equals the id this subagent was handed, and that the body is on-topic for the title. A summary-vs-transcript check cannot catch a whole-transcript swap. Only this can. State the result in the report: not "written", but "written and re-read: granola_id matches, body on-topic".
- Return **only**: the file path, the outcome (see Outcomes), the identity-check result, the `transcript_version`, whether there are deleted turns, the value verdict, the attendee list drawn from the transcript, the summary verbatim, and a **claim checklist**: one line per figure, attribution, date, and named commitment in the summary.

The subagent's context absorbs the bulk. The main context gets back a small payload it can work from.

### 3. Verify claim by claim

Build the checklist from the summary if step 2 did not hand one back: one line per figure, attribution, date, and named commitment. Then check each item against whichever backend step 1 chose and mark it:

- **confirmed**: in the transcript as stated
- **clarified**: present but different
- **refuted**: contradicted
- **unlocatable**: not found after a proper search, which means the summary invented the detail

The verdicts mean the same thing on both backends.

**Against the cached file.** `grep` the file for the figure or name and read a small window around the hit. If grep does not settle it, read a wider window, still on the file.

**Live.** For each claim:

1. `search_transcripts` with `meeting_ids` set to this one meeting and two to four sharp keywords: the figure, a proper noun, an unusual term. A common word ("project", "next week", a first name everyone uses) crowds out the specific ones and buries the passage you want.
2. Read one to three of the returned chunks with `get_meeting_transcript`, passing `chunk_ids` and the `transcript_version` from the search. If the claim straddles a chunk boundary, read the neighbour too.
3. Mark the verdict from what the chunks say.

Live rules, on top of the general rules below:

- **One Granola call at a time.** The account is rate-limited across every session using it, not just this one. Do not fan out parallel searches or chunk reads. On a rate-limit error, wait the `retry_after_seconds` it gives and retry that one call.
- **Never answer from the search fragment.** The roughly 240-character excerpt in a search hit is a pointer, not evidence. Read the chunk.
- **A search miss is not `unlocatable`.** Literal keyword search misses paraphrase, numbers spoken as words, and misspelt names. Try different keywords, read wider around the nearest related hit, or cache the transcript and grep it. Only call a claim unlocatable when a reasonable search has genuinely come up empty, and say what was searched.
- **Attribution is as weak live as it is in a cached file.** A chunk carries the same diarisation labels a full pull does. Every attribution rule below applies unchanged.
- **Long checklists can go to a subagent.** Chunk reads are small, so a handful of claims can run in the main context. For a long checklist, hand the live loop to one subagent (still one call at a time) and take back only the verdict table.

Verification rules, for both backends:

- **Sanity-check the meeting's identity, not just its claims.** Granola sometimes attaches a title and attendee list to a recording of a **different** meeting. One run carried a ten-person client attendee list for a meeting where none of them speak. Before verifying claims, confirm the content matches the title and that listed participants actually appear. If they do not, do not assert the attendee list. When content and title disagree, **content decides where the file belongs**: keep the original title in frontmatter, but name the file for what the meeting actually is.
- **Attribute from content, not just speaker labels.** Diarisation mislabels speakers, and it mislabels them in ways that look clean. Some transcripts carry only generic labels (Speaker A/B, or an audio source like Microphone and System audio) and no names at all. For anything about who owns, asked for, or committed to something, resolve it from what is actually said and **write the tell into the note**.
- **The attendee list is a claim, not a fact.** It routinely lists only the note-taker when five to ten people spoke, lists invitees who never joined, and occasionally splits one person in two across two email addresses. Build the list from the transcript: add every speaker the transcript names, drop invitees who never speak, and collapse a person who appears twice.
- **Resolve bare first names against the attendee list**, not against whoever is usually discussed.
- **Do not let repo vocabulary become a transcript claim.** A subagent that has read the host repo's docs absorbs its terms and can write one into what reads as a quote. For example, "DPO sign-off" where the transcript said "deletion policies" and "data retention" and never said DPO. This is the same error the skill exists to prevent, arriving by a route it does not otherwise name. Keep what was **said** separate from what you **resolved** using repo context, and never substitute the repo's preferred term for the words actually used.
- **Log third-party claims as "X said Y", never as "Y".**
- **Money and contractual figures get a harder test.** A transcript proves someone said a number, not that the number is right. Check those against the authoritative paperwork, not the transcript.

### 4. Write up the result

Where the result goes depends on how durable it is.

**A distilled note** (a meeting note, a 1:1 log, a person note, whatever the host repo uses) is a caching trigger in its own right. If the transcript was verified live, cache it before writing the note. The note goes in its proper home, **not** the cache file, which stays raw ground truth. It carries:

- A review marker until a human has checked it (e.g. `status: needs-review` in frontmatter, plus a visible callout).
- A `source:` naming the cache file and the `granola_id`.
- A `Summary says / Transcript says / verdict` table for anything clarified, refuted, or unlocatable.

**A lighter output** (a task line, an answer in chat, a ledger row) does not need a cache file behind it. It may cite the `granola_id` and the date it was verified instead.

A live verification always writes its ledger row (see Outcomes), whatever else it produces.

### 5. Commit the transcript and the note as two separate commits

The transcript cache is shared ground truth; the distilled note is branch-local work still under review. Keep them in **two commits** so the transcript can travel to main and to other branches on its own, without dragging the unreviewed note along.

Check `git diff --cached --name-only` before each commit, and scope every commit with `git commit -- <paths>`:

1. **The transcript first, alone.** `git commit -- <cache-dir>/YYYY-MM-DD-<slug>.md`. That single file and nothing else. This is the portable unit that step 6 and the sharing section move around; keeping it to one file is what makes it cherry-pick cleanly with no chance of conflict. Note its SHA. The next steps replay it.
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

Only the raw transcript and its index entry go to main. The distilled note is **not** in this PR. The interpretation stays on the feature branch under review. If a re-pull changed the transcript, publish the newer file the same way.

## Outcomes

Every meeting the skill touches ends in one of these, and the report says which:

- **CACHED**: written, re-read, identity confirmed. The normal path under `always`.
- **VERIFIED-LIVE**: checked live, nothing cached. The normal path under `on-citation` when no trigger is met. Writes one row to the ledger (below).
- **EMPTY**: the meeting recorded nothing usable, or Granola reports `no_transcript` or `transcript_deleted`. Write no file; report it so the gap is explained rather than looking like a miss.
- **BLOCKED**: the transcript exists but cannot be reproduced: an output content filter fires when the text is written out. The no-retype fallback (save the tool result straight to a file) works only when the result is large enough to be saved as a file, not when it comes back inline. Record the `granola_id` and the reason so the meeting stays re-fetchable, and write no partial file.

Also assign each meeting a **value verdict**: `keep` (has verifiable substance) or `low-value` (a test recording, an empty hold, a two-line check-in), so a bulk run hands back deletion candidates. Report the low-value ones; let the human delete.

### The ledger

A live verification leaves no transcript behind, so the ledger is its record. Path: `<cache-dir>/verified-live.md` unless the host repo names another. One row per meeting:

```markdown
| date | title | granola_id | transcript_version | checked | speaker labels | summary said | transcript has | verdict |
|------|-------|------------|--------------------|---------|----------------|--------------|----------------|---------|
| YYYY-MM-DD | <title> | <uuid> | <version> | YYYY-MM-DD | named / generic / mixed | <claim, paraphrased> | <what was said, paraphrased> | 7 confirmed, 1 clarified, 1 unlocatable |
```

- `speaker labels`: `named` if the transcript names speakers, `generic` if it only has Speaker A/B or audio-source labels, `mixed` if some of each. It tells a later reader how much to trust any attribution in the row.
- `summary said` / `transcript has`: the claims that came back clarified, refuted or unlocatable, one short clause each. Confirmed claims are counted in the verdict, not listed.
- **Paraphrase. No verbatim quotes.** The ledger is not a partial transcript cache. If exact words matter, that is a reason to cache.
- Re-verifying a meeting updates its row in place: new `transcript_version`, new `checked` date, new findings.
- **When a ledgered meeting is later cached**, move its findings to the cache index entry for that transcript and delete the ledger row. A meeting lives in one place or the other, never both.

Commit ledger changes on their own, separate from distilled notes. The ledger paraphrases meeting content, so it needs the same access boundary as the cache.

## Running this in bulk

Bulk import always caches, whatever the policy. It is the explicit "give me copies" request. Bulk is where the failures above actually bite. For an import of many meetings at once:

1. **Build a manifest first.** One row per meeting: `granola_id`, title, date, target cache path. Everything downstream keys off it, and the integrity pass checks against it.
2. **Batch three or four meetings per subagent.** A transcript is 15k to 30k tokens; ten per agent truncates. Three or four fits.
3. **Isolate scratch per agent** (step 2). This is what stops concurrent fetches from overwriting each other's intermediates.
4. **Each subagent reports per meeting:** outcome (CACHED / EMPTY / BLOCKED), the identity self-check result, `transcript_version`, whether there are deleted turns, attendees drawn from the transcript, and the value verdict.
5. **Run a mandatory integrity pass before committing anything.** Checksum every transcript body across the whole set. **Identical bodies mean cross-contamination** (two cache files with correct-but-different frontmatter over the same transcript) unless the host repo deliberately dual-caches (see Configuration). Any collision is a corrupted fetch: re-fetch those meetings before committing.
6. Commit only after the integrity pass is clean, following the two-commit rule.

## Sharing a cached transcript into other branches

Publishing to main reaches every *new* branch, and any branch that later syncs main. It does not reach back into branches that already diverged. When the user names specific existing branches that need a transcript now, cherry-pick the transcript commit onto each, and nothing else.

For each branch named:

1. `git fetch origin <branch> main`.
2. `git switch -C _share origin/<branch>`.
3. `git cherry-pick <transcript-sha>`: the same commit that went to main. If the branch already carries it, the cherry-pick comes up empty; skip that branch.
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
transcript_version: <version>   # from the fetch; re-verify when the live one differs
pulled: YYYY-MM-DD              # when last fetched; bump on re-pull
attendees: [<person>, ...]
status: raw
---
```

Match the host repo's frontmatter and tagging conventions where it has them.

When the content contradicts the Granola title (see the identity sanity-check), keep the original in `title:` and slug the **filename** for what the meeting actually is; note the mismatch. When the attendee list cannot be trusted, populate `attendees:` from transcript speakers only, or leave it out. Do not copy Granola's list uncritically.

Then:

- `## Summary (AI-generated, hypothesis, unverified)`: the summary verbatim, in a warning callout.
- `## Transcript`: verbatim, **speaker labels left intact.** The labels are what attribution depends on, so do not strip them. `[DELETED]` turn markers stay too.

If the transcript has deleted turns, say so in its cache index entry, so a reader knows the gaps are the recorder's trim and not a bad fetch.

## Constraints

- The cache file is ground truth, never a distilled note. Do not edit the transcript text or "tidy" it. Corrections belong in the distilled note, with a source.
- An unlocatable claim is a finding, not a gap to fill. Say the summary asserted it and the transcript does not carry it.
- Do not load the full transcript into the main context to "double check". If grep or a chunk read did not settle it, read a wider window (still on the file, or more chunks), or cache it through a subagent.
- A re-pull is the only thing that should overwrite a cache file, and only with a fresh fetch of the same meeting. Never hand-edit a transcript to "update" it.
- Transcripts can contain confidential and personal material. Cache them only in a repo whose access boundary suits the content, and keep the distilled note in draft until a human has reviewed it. The same goes for the ledger.
- A calendar-driven capture will eventually pull in something genuinely personal: unrelated to any work the repo covers, carrying other people's private details. One will. Do not commit it, and do not ledger it either. Report it and let the human decide.
