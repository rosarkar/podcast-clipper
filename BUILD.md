# Podcast Clipper — BUILD.md

_Last updated: Oct 13, 2026_

## Goal

Build a local-first clipping pipeline that can take a long-form YouTube/podcast video, identify the strongest short-form moments, convert them into TikTok-ready vertical clips, add kinetic captions, generate suggested post copy, and send finished videos to the authorized TikTok account's inbox/draft flow for manual review and posting.

The current target is a Sonny Angel fan account. The account should feel like a genuine collector/fan page rather than a commercial storefront.

---

## Current V2 Workflow

Current intended usage:

```bash
python3 process_source.py VIDEO TRANSCRIPT
```

Example:

```bash
python3 process_source.py podcast.mp4 source_transcript.txt
```

Current flow:

1. Load the source video.
2. Use a supplied timestamped transcript when available.
3. If no transcript is provided, fall back to local Whisper transcription.
4. Send the timestamped transcript to Kimi.
5. Kimi selects the top five short-form clips and returns:
   - rank
   - start/end timestamps
   - score
   - hook
   - category
   - reason
   - suggested TikTok caption
6. Display all five choices for review.
7. One confirmation starts the batch.
8. Each clip is rendered by `run_pipeline.py`.
9. Local Whisper generates word-level timing for the selected clip.
10. `make_ass.py` builds kinetic ASS captions.
11. Captions are burned into `final_tiktok.mp4`.
12. The video is uploaded through TikTok's inbox/draft Content Posting flow.
13. Finished local copies and caption/metadata files are stored in:
    - `outputs/<source_name>/`

---

## What Worked Today

### Kimi clip selection

Kimi successfully analyzed the full timestamped source transcript and selected five strong Sonny Angel clips.

The selection quality was good enough to use as the editorial layer. It correctly prioritized topics such as:

- Robby / secret figures
- fake vs real Sonny Angels
- blind-box weighing/shaking controversy
- expensive/custom figures
- beginner collecting/trading advice

Kimi also generated usable fan-account-style TikTok captions.

### Batch rendering

`process_source.py` successfully started processing multiple clips sequentially.

The first three clips:

- rendered successfully
- received word-level captions
- burned captions correctly
- uploaded successfully
- reached `SEND_TO_USER_INBOX`
- were saved locally as separate files

The fourth clip rendered successfully but TikTok rejected the next inbox initialization with:

```text
spam_risk_too_many_pending_share
```

This was not a rendering bug.

The TikTok account already had pending videos from earlier tests, so the available pending/draft capacity was reached.

### Important architecture lesson

Rendering and TikTok uploading should eventually be separated so a TikTok queue issue never prevents later clips from being generated.

Future behavior should be:

```text
render all clips
↓
save all local copies
↓
attempt TikTok uploads independently
↓
if TikTok queue is full, keep the local file and continue
```

---

## Future Visual Mode: Split-Screen Supporting Visuals

A strong future upgrade is a split-screen explainer layout inspired by short-form editing workflows in tools like CapCut.

Concept:

```text
┌─────────────────────┐
│ source video        │
│ speaker / product   │
├─────────────────────┤
│ supporting visual   │
│ image / B-roll /    │
│ screenshot / card   │
└─────────────────────┘
```

The purpose is to improve comprehension and retention by showing the viewer a relevant supporting visual while the speaker is talking.

Examples:

- Speaker mentions Robby → show a Robby image in the lower half.
- Speaker discusses fake vs real Sonny Angels → show a comparison image.
- Speaker says a figure sells for ~$400 → show the figure or a relevant price/reference screenshot.
- Speaker explains a collector term → show a simple text/graphic card.
- Speaker references something shown earlier → reuse a zoomed crop or still from the source video.

This should be implemented as an optional render mode, not a replacement for the current blurred 9:16 default.

Suggested future render modes:

```text
fit_blur
    full 16:9 source preserved over blurred 9:16 background

split_image
    source video on top
    one supporting still image on bottom

split_video
    source video on top
    supporting B-roll/video on bottom

smart_visual
    system automatically chooses or generates supporting visuals by segment
```

Recommended rollout:

1. `split_image`
   - user supplies one image manually
   - easiest and safest version
2. asset-library mode
   - select from a local folder of reusable images
3. Kimi-assisted visual suggestions
   - Kimi returns a visual type/query with each clip
4. automatic visual retrieval/generation
   - only after the core workflow is stable

Potential future metadata from Kimi:

```json
{
  "visual_type": "comparison_image",
  "visual_query": "fake vs real Sonny Angel packaging",
  "visual_reason": "supports the spoken warning and makes the clip easier to understand"
}
```

Avoid building the fully automatic version first.

Manual or local-asset support will be faster to debug and avoids introducing unnecessary copyright, watermark, quality, and search problems.

---

## Keeping Kimi Inference Costs Low

Current Kimi usage is already reasonably efficient, but this should remain a design constraint so a small credit balance can last for many source videos.

The user estimated today's experimentation used roughly $0.50 of Kimi credits.

Future work should focus on avoiding repeated inference rather than assuming a large recurring budget.

### Core cost rule

Use Kimi only for tasks where language-model judgment adds real value:

```text
Kimi:
    rank clips
    judge hooks
    categorize clips
    write short post captions

Local/free tools:
    download/source handling
    transcript extraction
    Whisper transcription
    timestamp processing
    clip rendering
    caption timing
    video encoding
    filtering
    validation
```

Do not use Kimi for mechanical work that Python, FFmpeg, yt-dlp, or Whisper can do locally.

### 1. One Kimi call per source video

The preferred workflow is:

```text
one compact timestamped transcript
→ one Kimi request
→ 3–5 ranked clips + captions + metadata
```

Avoid separate calls for:

- clip ranking
- caption writing
- category labeling
- hook writing

Return all of those fields in one JSON response.

### 2. Use segment-level transcripts for selection

Do not send full Whisper word-level JSON to Kimi.

For source selection, use compact lines such as:

```text
[03:03-03:15] There are so many fakes out there...
[03:15-03:28] Sonny Angels are typically priced...
```

Word-level timing is only needed later for the selected clips and should remain local.

This can reduce prompt size dramatically.

### 3. Prefer YouTube captions over full-source Whisper when available

For Kimi selection:

```text
YouTube timed captions
→ compact transcript
→ Kimi
```

Use local full-source Whisper only as a fallback.

This reduces local compute time and avoids unnecessarily verbose transcript output.

### 4. Cache every Kimi response

If `clip_candidates.json` already exists for the same source/transcript, do not call Kimi again unless explicitly requested.

Future improvement:

```text
source video hash + transcript hash
→ cached Kimi result
```

Possible CLI:

```bash
python3 process_source.py podcast.mp4 source_transcript.txt
```

should automatically reuse the existing selection when the source and transcript have not changed.

Optional override:

```bash
--rerank
```

This is one of the highest-value cost-saving improvements.

### 5. Separate rendering retries from inference

A failed FFmpeg render or TikTok upload must never trigger Kimi again.

Once clip choices are saved:

```text
clip_candidates.json
```

all rendering/upload retries should operate from that saved file.

Examples:

```bash
python3 render_candidates.py clip_candidates.json
python3 upload_outputs.py outputs/podcast/
```

This prevents paying for the same editorial decision twice.

### 6. Keep prompts short and stable

Avoid repeatedly sending long instructions explaining the entire project.

The Kimi prompt should contain only:

- account/audience goal
- clip criteria
- required JSON schema
- timestamped transcript

Move stable instructions into a concise prompt template.

Avoid sending:

- README content
- build history
- long explanations of the pipeline
- raw Whisper metadata
- previous model answers unless needed

### 7. Ask for fewer clips when appropriate

Five clips is useful for strong source videos, but not every source needs five.

Possible future option:

```bash
--clips 3
```

For weak or short videos, selecting three instead of five can reduce output tokens and avoid forcing mediocre clips.

A later heuristic could estimate source length and choose something like:

```text
< 8 min     → 3 clips
8–20 min    → 5 clips
20+ min     → 5–8 clips
```

while keeping the number configurable.

### 8. Pre-filter obvious weak transcript regions locally

Before Kimi, local rules can remove:

- intro housekeeping
- sponsor reads
- long pauses
- outro/social plugs
- repeated sections
- segments with almost no speech

This means Kimi sees fewer tokens and focuses on higher-signal material.

Do not over-engineer this initially.

Simple chapter/outro heuristics are enough.

### 9. Reuse one inference for multiple render styles

If one clip is later rendered as:

```text
fit_blur
split_image
split_video
```

reuse the same clip selection and caption metadata.

Visual experimentation should never require rerunning Kimi.

### 10. Benchmark cheaper models only if needed

If Kimi offers a less expensive model that still reliably:

- follows the JSON schema
- understands timestamps
- ranks short-form clips well
- writes usable captions

benchmark it on the same transcript and compare results.

Do not switch purely because it is cheaper if selection quality drops materially.

The goal is lowest cost per usable clip, not lowest token price.

### Cost-efficient target architecture

```text
YouTube URL
→ captions/transcript locally
→ compact transcript
→ ONE Kimi call
→ cache clip_candidates.json
→ everything else local
→ unlimited rerenders without additional Kimi cost
```

That should make a small Kimi balance last much longer.

---

## 9:16 Framing Problem

### Old behavior

The original vertical conversion used:

```text
crop=608:1080:656:0,scale=1080:1920
```

This center-cropped the 1920x1080 source into a narrow vertical strip.

Problem:

- left/right content was discarded
- products or objects shown near the sides disappeared
- this is especially bad for product/collector videos

### New default

For ordinary horizontal 16:9 YouTube videos, the default should be:

- 1080x1920 output
- preserve the complete original 16:9 frame
- center the full source frame
- fill unused vertical space with a blurred enlarged copy of the same video
- no stretching
- captions remain burned over the final 9:16 composition

Conceptually:

```text
┌─────────────────────┐
│ blurred background  │
│                     │
│ ┌─────────────────┐ │
│ │ full 16:9 video │ │
│ └─────────────────┘ │
│                     │
│ blurred background  │
└─────────────────────┘
```

The visual test succeeded.

A full captioned production render also succeeded locally.

The final output preserved the entire horizontal frame, used the blurred background correctly, and kept the kinetic captions readable.

---

## 9:16 Default Implementation

`run_pipeline.py` now uses the blurred-background composition.

The intended filter is:

```text
[0:v]split=2[bg][fg];
[bg]scale=1080:1920:force_original_aspect_ratio=increase,
crop=1080:1920,
boxblur=20:1[blurred];
[fg]scale=1080:-2:force_original_aspect_ratio=decrease[foreground];
[blurred][foreground]overlay=(W-w)/2:(H-h)/2,
setsar=1[v]
```

The intended FFmpeg ordering is:

```python
"-ss", start_time,
"-i", input_file,
```

rather than seeking after opening the source.

This lets FFmpeg seek before decoding and is much faster when clipping later parts of long videos.

### Final intended `create_vertical_clip()` behavior

The function should:

1. seek to `start_time`
2. open the source video
3. create the blurred 9:16 composition
4. preserve source audio
5. write `pipeline_vertical.mp4`

Then the existing pipeline continues unchanged:

```text
pipeline_vertical.mp4
→ Whisper
→ captions.ass
→ final_tiktok.mp4
→ TikTok
```

The standalone `fit_blur_test.mp4` did not have captions because it was only a framing test.

The normal production pipeline still adds captions afterward.

---

## Definition of Done for the 9:16 Change

The new framing can be considered complete when all of the following are true:

- `run_pipeline.py` passes:

```bash
python3 -m py_compile run_pipeline.py
```

- `create_vertical_clip()` uses the blur/overlay composition
- `-ss` appears before `-i`
- blur is `boxblur=20:1`
- output is 1080x1920
- entire 16:9 source frame is visible
- no horizontal stretching
- kinetic captions still render correctly in the normal pipeline
- local production render has been visually verified

No other major code is required for this to become the default layout for normal 16:9 YouTube sources.

---

## Captions

Current caption flow:

1. selected clip becomes `pipeline_vertical.mp4`
2. local Whisper runs with word timestamps
3. output JSON becomes `words.json`
4. `make_ass.py` builds `captions.ass`
5. FFmpeg burns ASS captions into `final_tiktok.mp4`

Current style:

- roughly 4 words visible at once
- white bold text
- strong black outline
- current spoken word highlighted yellow
- lower-middle safe area
- no bubble/pill
- Avenir Next Condensed
- large TikTok-friendly size

### Proper noun correction

Whisper often transcribes:

```text
Sunny Angel
Sunny Angels
```

Instead of:

```text
Sonny Angel
Sonny Angels
```

`make_ass.py` includes a proper-noun correction step before ASS formatting.

This successfully fixes the final burned captions.

---

## Transcript Strategy

Preferred source-selection transcript order:

1. official/user-provided timed captions
2. YouTube auto-generated timed captions
3. local Whisper fallback

For Kimi selection, segment-level timestamps are sufficient.

For the final selected clips, continue using local Whisper because the kinetic captions need accurate word-level timing.

Important:

A supplied transcript must contain timestamps.

Plain untimed transcript text is not enough for Kimi to reliably choose source clip boundaries.

---

## TikTok Behavior

Current integration uses the TikTok Content Posting API inbox/draft workflow.

Successful upload sequence:

```text
initialize
→ upload HTTP 201
→ PROCESSING_UPLOAD
→ SEND_TO_USER_INBOX
```

TikTok caption text cannot currently be pre-filled through this inbox workflow, so the system saves Kimi's suggested caption locally for manual copy/paste when posting.

### Current TikTok limitation encountered

TikTok returned:

```text
spam_risk_too_many_pending_share
```

when the account already had too many pending inbox/draft shares.

This should be treated as a queue/full-state issue, not a rendering failure.

### Current improvement needed

Do not crash the entire batch if TikTok rejects one upload.

Instead:

- save the rendered local clip first
- catch TikTok upload errors
- print the error
- continue rendering remaining clips

Desired behavior:

```text
Clip 1 → rendered → uploaded
Clip 2 → rendered → uploaded
Clip 3 → rendered → uploaded
Clip 4 → rendered → TikTok queue full → saved locally
Clip 5 → rendered → saved locally
```

TikTok should become an optional delivery step rather than a dependency for rendering completion.

---

## Files

Core files:

```text
process_source.py
run_pipeline.py
make_ass.py
transcribe_source.py
find_clips.py
tiktok_auth.py
tiktok_upload.py
tiktok_status.py
```

Runtime/intermediate files:

```text
pipeline_vertical.mp4
words.json
captions.ass
final_tiktok.mp4
source_segments.json
source_transcript.txt
clip_candidates.json
```

Output structure:

```text
outputs/
  podcast/
    clip_01.mp4
    clip_01_caption.txt
    clip_01_info.json
    clip_02.mp4
    ...
```

---

## Security Before Public GitHub Upload

Do not commit credentials or generated media.

Recommended `.gitignore`:

```gitignore
tiktok_tokens.json
.env
*.mp4
*.srt
*.ass
words.json
pipeline_vertical.json
__pycache__/
.DS_Store
```

Move any hardcoded TikTok client secret out of source code and into environment variables before making the repository public.

Do not commit:

```text
KIMI_API_KEY
TikTok client secret
TikTok access tokens
TikTok refresh tokens
```

---

## Next Build Session

Priority order:

1. Make `process_source.py` resilient to TikTok queue/upload failures so later clips still render.
2. Ensure rendered local files are saved before TikTok upload is attempted.
3. Reuse cached `clip_candidates.json` so retries never use Kimi credits.
4. Add `yt-dlp` input:
   - source video download
   - subtitle/auto-caption retrieval when available
5. Normalize SRT/VTT/YouTube captions into the transcript format Kimi expects.
6. Build:

```bash
python3 process_youtube.py "YOUTUBE_URL"
```

7. Add optional render modes later:
   - `fit_blur`
   - `split_image`
   - `split_video`
8. Eventually add source-video discovery/agent logic only after the one-video workflow is stable.

---

## Target One-Command Experience

Longer-term target:

```bash
python3 process_youtube.py "YOUTUBE_URL"
```

Expected flow:

```text
YouTube URL
→ download authorized source
→ obtain timed captions if available
→ Whisper fallback if needed
→ compact transcript
→ ONE Kimi inference
→ cache clip selections
→ user reviews selections once
→ render full-frame blurred 9:16 clips
→ local Whisper word timing
→ kinetic captions
→ save all local outputs
→ upload available clips to TikTok inbox
→ user manually reviews/posts
```

---

## Longer-Term Agent Workflow

Eventually:

```text
agent discovers promising source videos
→ checks source / creator / rights criteria
→ downloads authorized video
→ fetches captions
→ Kimi ranks the strongest moments
→ pipeline renders clips
→ TikTok inbox receives available drafts
→ user reviews/posts
```

Potential later feedback loop:

```text
TikTok performance
→ hook/category performance data
→ improve future clip-selection rules
```

Do not build this until the single-source workflow is reliable.

---

## Development Principles

### 1. Local first

Use local/free tools wherever possible.

Prefer:

- FFmpeg
- Whisper
- yt-dlp
- Python
- cached JSON/files

Use paid inference only where judgment is needed.

### 2. Save before external actions

Always save a finished local artifact before:

- uploading to TikTok
- calling another API
- starting another clip

External failures should never destroy completed work.

### 3. Cache expensive decisions

Once Kimi chooses clips, save the result permanently.

Rendering changes should not trigger new inference.

### 4. Keep the stable pipeline stable

Do not broadly rewrite working components.

Prefer targeted changes.

Current stable pieces include:

- Whisper caption timing
- ASS generation
- proper noun correction
- TikTok OAuth
- TikTok inbox upload
- TikTok status polling
- FFmpeg caption burn
- blurred 9:16 framing

### 5. Build one abstraction only when needed

Do not build multi-account or multi-template infrastructure before a second real use case exists.

The Sonny Angel account is the current proving ground.

---

## Debugging Guide

### FFmpeg says `Late SEI is not implemented`

This warning has appeared repeatedly with the test YouTube source.

It has not prevented successful rendering.

Treat it as non-fatal unless video output actually fails.

### Whisper says:

```text
FP16 is not supported on CPU; using FP32 instead
```

Expected on the current Mac CPU setup.

Not an error.

### TikTok upload returns HTTP 201

Good.

Continue polling status.

### TikTok status reaches:

```text
SEND_TO_USER_INBOX
```

Success.

Check TikTok notifications/inbox.

### TikTok returns:

```text
spam_risk_too_many_pending_share
```

Pending inbox/draft capacity is full.

Do not rerun Kimi.

Keep the local clip and retry upload later.

### Captions say `Sunny Angel`

Check:

```bash
grep -i "sonny" captions.ass
```

If ASS contains `Sonny`, the proper noun correction is working even if raw Whisper console output says `Sunny`.

### Video sides are missing

Old crop mode is being used.

The normal 16:9 default should now use `fit_blur`, not:

```text
crop=608:1080:656:0
```

### Video render is unexpectedly slow

Check that FFmpeg seeks before decoding:

```text
-ss START
-i VIDEO
```

and that the blur is:

```text
boxblur=20:1
```

rather than the earlier heavier blur.

---

## Current Project Status

The difficult pieces are now proven individually and together:

- local clipping
- vertical formatting
- full-frame blurred 9:16 layout
- kinetic word highlighting
- proper-noun correction
- TikTok API upload
- upload-status polling
- full-source transcription
- Kimi API connection
- Kimi editorial clip selection
- Kimi caption generation
- multi-clip batch rendering
- multi-clip TikTok inbox delivery
- local production render with blurred layout + captions

The remaining work is mostly:

```text
reliability
source acquisition
cost control
render-mode expansion
workflow polish
```

rather than proving the core system can work.

The next major milestone is:

```text
YouTube URL
→ one Kimi call
→ several cached clip selections
→ several local TikTok-ready videos
→ upload when TikTok capacity is available
```

without repeated inference or manual source preparation.
```
