name: ai-clipping
description: Turn a long-form video or audio transcript (podcast, interview, livestream, lecture, webinar, YouTube video) into a ranked list of short-form clips ready for TikTok, Instagram Reels and YouTube Shorts. Finds the strongest moments, gives exact start/end timestamps, a hook, title, caption, hashtags and a virality score for each clip, and outputs both a readable table and machine-readable JSON. Use this skill whenever the user pastes or uploads a transcript, SRT/VTT file or timestamped text and mentions clips, clipping, shorts, reels, TikToks, highlights, "best moments", repurposing, or "cut this into short videos", even if they never say the word "clipping".
summary: Turns any podcast, interview or livestream transcript into a ranked list of TikTok, Reels and Shorts clips with exact timestamps, hooks, titles, captions, hashtags and virality scores.
version: 1.0.0
tags: [clipping, shorts, reels, tiktok, video-repurposing, transcript, content-creation, social-media]
keywords: [ai clipping, video clips, viral clips, podcast clips, highlights, repurpose video, srt, vtt]
use_cases: [Find the best 5-8 moments in a podcast episode, Convert a livestream transcript into short-form clips, Write hooks and captions for each clip, Export clip timestamps as JSON for editors or ffmpeg]
---

# AI Clipping

Turn a long transcript into short clips that people actually watch to the end. This skill is analysis only: it decides *what* to cut and *how to package it*. It does not cut video. The output (timestamps + JSON) is meant to be handed to an editor, CapCut, Descript, or an ffmpeg script.

## Why this matters

A clip succeeds or fails in the first 2-3 seconds and by whether it feels complete on its own. Most bad clips are either (a) a good idea that starts with filler, or (b) a fragment that only makes sense if you watched the full episode. Every decision below exists to avoid those two failures.

## Inputs you can handle

- SRT or VTT files
- Plain text with timestamps (`[00:12:34] text`, `12:34 - text`, etc.)
- Speaker-labelled transcripts (`Host:`, `Guest:`)
- Plain text with **no** timestamps (see "No timestamps" below)

If the user uploaded a file and its content is not already in context, read it from `/mnt/user-data/uploads/` first.

## Workflow

### 1. Get the context (ask only if it changes the result)

Defaults, if the user doesn't say otherwise: platform = TikTok/Reels/Shorts, clip length = 20-60 seconds, language = English, 5-8 clips returned.

Ask a single short question only when something important is genuinely missing, for example the niche or audience of a transcript that doesn't make it obvious. Otherwise proceed with the defaults and state them in one line so the user can correct you.

### 2. Read the whole transcript before choosing anything

Skim the full transcript first to understand the arc: what the main topics are, who the speakers are, where the energy changes. Moments only look strong or weak relative to the rest of the video.

### 3. Collect candidate moments (aim for 2-3x the number you will return)

Look for these moment types. Read `references/scoring.md` for the detailed rubric and `references/example.md` for a worked example of the expected output.

- **Contrarian take**: a claim that goes against common belief
- **Story with a payoff**: setup, tension, resolution within one minute
- **Surprising fact or number**
- **Emotional peak**: laughter, vulnerability, anger, a confession
- **Actionable tip**: a concrete step someone can use today
- **Conflict or debate**: disagreement, pushback, a clash
- **Quotable line**: a short, punchy statement worth screenshotting

### 4. Score and choose

Score each candidate with the rubric in `references/scoring.md` (hook, completeness, emotion/novelty, shareability, standalone clarity). Keep the top N and drop near-duplicates: two clips making the same point should become one.

Be honest with scores. If the transcript is weak, say so and return fewer clips rather than padding the list. A list of 4 strong clips is more useful than 8 mediocre ones.

### 5. Set precise boundaries

For each chosen clip:

- **Start** at the beginning of the sentence that works as the hook, not at the start of the topic. Cut the warm-up ("so, um, yeah, I think...") unless it is the hook.
- **End** right after the thought completes, ideally on the punchline or the conclusion. Stop before the speaker starts rambling or moving to the next topic.
- Use the transcript's own timestamps. Never invent or interpolate a timestamp that the transcript doesn't support. If a segment boundary falls mid-line, use the nearest listed timestamp and say so in `notes`.
- Keep within the target duration. Slightly longer is fine if cutting would break the thought; say why in `notes`.
- **Standalone test**: read the clip as if you've never seen the video. Does every "it", "that", and "he" have a referent inside the clip? If not, move the start earlier, or flag that on-screen text or a one-line intro is needed.

### 6. Package each clip

For every clip write:

- **Title**: under 60 characters, specific, creates curiosity or promises value. Avoid vague titles like "Great advice".
- **Hook line**: the exact words (or on-screen text) for the first 3 seconds. Prefer the speaker's own words. If the best line is later in the clip, suggest a **cold open** (start the clip with that line, then jump back) and mark it as optional.
- **Caption**: 1-3 short sentences, matching the tone of the content, ending with a light call to action or question when natural.
- **Hashtags**: 4-6, a mix of broad and niche, no spam stacking.
- **Why it works**: one sentence, concrete.
- **Score** (1-10) and **clip type** from the list above.

### 7. Output

Always give both parts, in this order.

**Part A: summary table**

| # | Time | Dur | Type | Score | Title |
|---|------|-----|------|-------|-------|

Then, for each clip, a short block with hook, caption, hashtags and why it works.

**Part B: JSON** (one block, valid JSON, no comments):

```json
{
  "source": "episode title or filename if known",
  "platform": "tiktok_reels_shorts",
  "clips": [
    {
      "rank": 1,
      "start": "00:12:34",
      "end": "00:13:21",
      "duration_seconds": 47,
      "type": "contrarian_take",
      "score": 9,
      "title": "...",
      "hook": "...",
      "cold_open_suggestion": null,
      "caption": "...",
      "hashtags": ["#...", "#..."],
      "why_it_works": "...",
      "notes": ""
    }
  ]
}
```

Use `HH:MM:SS` (add `.mmm` only if the source has milliseconds). `cold_open_suggestion` is either `null` or an object `{ "line_start": "...", "line_end": "..." }`.

End with a one-line offer: for example to re-rank for a different platform, tighten durations, or write the ffmpeg commands if they want to cut the files themselves.

## Edge cases

**No timestamps.** Don't fabricate any. Identify the best moments, quote the first and last words of each so the user can find them, set `start` and `end` to `null` in the JSON, and ask whether they can provide an SRT/VTT export (YouTube, Descript, Whisper and most podcast tools can produce one).

**Very long transcripts (2+ hours).** Work through it in sections, keep a running shortlist, then do a final global ranking so clips from the beginning aren't favoured over later ones.

**Multiple speakers.** Prefer clips where the speaker is clearly identifiable. If a clip needs a question from the host to make sense, include that question and keep it short.

**Sensitive content.** If a strong moment contains hate speech, personal attacks on named people, medical or financial claims that could mislead, or private information, flag it in `notes` instead of silently promoting it. Choosing the viral moment is the job; deciding whether to publish it is the creator's call.

**Transcript errors.** Auto-generated transcripts contain mistakes. Quote the transcript faithfully in `hook` but you may flag obvious mis-transcriptions in `notes`.

**Non-English transcripts.** This skill is tuned for English. If the transcript is in another language, still do the analysis, but say that hook and hashtag wording may need native review.
