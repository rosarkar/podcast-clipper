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
5. Kimi selects the top five short-form clips and returns rank, start/end timestamps, score, hook, category, reason, and a suggested TikTok caption.
6. Display all five choices for review.
7. One confirmation starts the batch.
8. Each clip is rendered by `run_pipeline.py`.
9. Local Whisper generates word-level timing for the selected clip.
10. `make_ass.py` builds kinetic ASS captions.
11. Captions are burned into `final_tiktok.mp4`.
12. The video is uploaded through TikTok's inbox/draft Content Posting flow.
13. Finished local copies and caption/metadata files are stored in `outputs/<source_name>/`.

---

## What Worked Today

### Kimi clip selection

Kimi successfully analyzed the full timestamped source transcript and selected five strong Sonny Angel clips. It correctly prioritized topics such as Robby / secret figures, fake vs real Sonny Angels, blind-box weighing/shaking controversy, expensive/custom figures, and beginner collecting/trading advice.

Kimi also generated usable fan-account-style TikTok captions.

### Batch rendering

`process_source.py` successfully started processing multiple clips sequentially.

The first three clips rendered successfully, received word-level captions, burned captions correctly, uploaded successfully, reached `SEND_TO_USER_INBOX`, and were saved locally as separate files.

The fourth clip rendered successfully but TikTok rejected the next inbox initialization with:

```text
spam_risk_too_many_pending_share
```

This was not a rendering bug. The TikTok account already had pending videos from earlier tests, and the available draft/inbox capacity was reached.

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

## 9:16 Framing Problem

### Old behavior

The original vertical conversion used:

```text
crop=608:1080:656:0,scale=1080:1920
```

This center-cropped the 1920x1080 source into a narrow vertical strip. Left/right content was discarded, which is especially bad for product/collector videos.

### New desired default

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

---

## Final Change Needed for the 9:16 Default

`run_pipeline.py` has already been changed to use the blurred-background composition.

Before treating it as finished, make two small optimizations inside `create_vertical_clip()`.

### 1. Seek before decoding

Use:

```python
"-ss", start_time,
"-i", input_file,
```

rather than:

```python
"-i", input_file,
"-ss", start_time,
```

This lets FFmpeg seek into the source before decoding and is much faster for clips later in a long video.

### 2. Use the faster blur setting

Use:

```text
boxblur=20:1
```

instead of:

```text
boxblur=30:20
```

The heavier blur worked but was unnecessarily slow.

### Final intended filter

```text
[0:v]split=2[bg][fg];
[bg]scale=1080:1920:force_original_aspect_ratio=increase,
crop=1080:1920,
boxblur=20:1[blurred];
[fg]scale=1080:-2:force_original_aspect_ratio=decrease[foreground];
[blurred][foreground]overlay=(W-w)/2:(H-h)/2,
setsar=1[v]
```

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

The standalone `fit_blur_test.mp4` did not have captions because it was only a framing test. The normal production pipeline still adds captions afterward.

---

## Definition of Done for the 9:16 Change

The new framing can be considered complete when all of the following are true:

- `run_pipeline.py` passes `python3 -m py_compile run_pipeline.py`
- `create_vertical_clip()` uses the blur/overlay composition
- `-ss` appears before `-i`
- blur is `boxblur=20:1`
- output is 1080x1920
- entire 16:9 source frame is visible
- no horizontal stretching
- kinetic captions still render correctly in the normal pipeline
- one full end-to-end clip succeeds after TikTok queue space is available

No other major code is required for this to become the default layout for normal 16:9 YouTube sources.

---

## Transcript Strategy

Preferred source-selection transcript order:

1. official/user-provided timed captions
2. YouTube auto-generated timed captions
3. local Whisper fallback

For Kimi selection, segment-level timestamps are sufficient.

For the final selected clips, continue using local Whisper because the kinetic captions need accurate word-level timing.

Important: a supplied transcript must contain timestamps. Plain untimed transcript text is not enough for Kimi to reliably choose source clip boundaries.

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

Current improvement needed:

- save the rendered local clip first
- catch TikTok upload errors
- print the error
- continue rendering remaining clips

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

Do not commit `KIMI_API_KEY`.

---

## Next Build Session

Priority order:

1. Finalize the two small FFmpeg optimizations:
   - `-ss` before `-i`
   - `boxblur=20:1`
2. Run one normal full-pipeline smoke test with captions after TikTok queue space is available.
3. Make `process_source.py` resilient to TikTok queue/upload failures so later clips still render.
4. Add `yt-dlp` input:
   - source video download
   - subtitle/auto-caption retrieval when available
5. Normalize SRT/VTT/YouTube captions into the transcript format Kimi expects.
6. Build:
   ```bash
   python3 process_youtube.py "YOUTUBE_URL"
   ```
7. Eventually add source-video discovery/agent logic only after the one-video workflow is stable.

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
→ Kimi selects best clips
→ user reviews selections once
→ render full-frame blurred 9:16 clips
→ local Whisper word timing
→ kinetic captions
→ save all local outputs
→ upload available clips to TikTok inbox
→ user manually reviews/posts
```

---

## Current Project Status

The difficult pieces are now proven individually and together:

- local clipping
- vertical formatting
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
- full-frame blurred 9:16 layout

The remaining work is mostly reliability, source acquisition, and workflow polish rather than proving the core system can work.
