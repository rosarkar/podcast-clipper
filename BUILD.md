Yes. This is the right reliability fix to do before stopping for today.

The behavior we want is:

```text
render clip
→ burn captions
→ save permanent local copy
→ try TikTok
→ if TikTok works: great
→ if TikTok fails: print warning and continue
→ process next clip
```

That means a TikTok limit can no longer kill an entire five-clip batch.

### 1. Patch `run_pipeline.py`

Paste this whole block into Terminal:

```bash
cd ~/podcast-clipper && \
cp run_pipeline.py run_pipeline.py.before_nonfatal_tiktok && \
python3 - <<'PY'
from pathlib import Path
import re

path = Path("run_pipeline.py")
text = path.read_text()

pattern = re.compile(
    r'(?P<indent>^[ \t]*)publish_id\s*=\s*upload_to_tiktok\(\)\s*\n'
    r'(?P=indent)check_status\(publish_id\)',
    re.MULTILINE
)

replacement = r'''\g<indent>print()
\g<indent>print("===================================")
\g<indent>print("STEP 4: Uploading to TikTok")
\g<indent>print("===================================")

\g<indent>try:
\g<indent>    publish_id = upload_to_tiktok()

\g<indent>    if publish_id:
\g<indent>        check_status(publish_id)
\g<indent>    else:
\g<indent>        print()
\g<indent>        print("TikTok upload did not complete.")
\g<indent>        print("Local video is safely saved. Continuing.")

\g<indent>except SystemExit as e:
\g<indent>    print()
\g<indent>    print("TikTok upload failed or was blocked.")
\g<indent>    print("Local video is safely saved. Continuing.")
\g<indent>    print(f"TikTok exit code: {e}")

\g<indent>except Exception as e:
\g<indent>    print()
\g<indent>    print("TikTok upload failed.")
\g<indent>    print("Local video is safely saved. Continuing.")
\g<indent>    print(f"Error: {e}")'''

updated, count = pattern.subn(replacement, text, count=1)

if count != 1:
    raise SystemExit(
        "Could not safely find the TikTok upload/status block. "
        "No changes written."
    )

path.write_text(updated)

print("Updated TikTok upload handling.")
print("TikTok failures will no longer kill the render pipeline.")
PY

python3 -m py_compile run_pipeline.py && \
echo "Syntax check passed."
```

You want to see:

```text
Updated TikTok upload handling.
TikTok failures will no longer kill the render pipeline.
Syntax check passed.
```

You **do not need to test it tonight**. We already know TikTok is rejecting API uploads right now, and this patch is deliberately isolated around that final delivery step.

The important consequence is that `process_source.py` can now call `run_pipeline.py` five times and a TikTok failure should return control normally, allowing clip #2, #3, #4, #5 to keep rendering.

---

And here is the **full updated `BUILD.md`**. You can replace the current GitHub file with everything below.

```markdown
# Podcast Clipper — BUILD.md

_Last updated: Oct 13, 2026_

## Goal

Build a local-first content production system that can take long-form source media, identify strong short-form moments, convert them into platform-ready vertical clips, add kinetic captions, generate post copy, preserve finished local outputs, and optionally deliver them to the correct social account.

The system is being built incrementally, with local/free tools doing as much work as possible and paid model inference used only where judgment adds meaningful value.

The initial proving ground is a Sonny Angel TikTok fan account.

Longer term, the system may support several distinct content channels with separate configurations, including:

- Sonny Angel / collector content
- lofi / ambient / white-noise content
- podcast / entertainment clipping

Each channel should ultimately have its own:

- source criteria
- content-selection rules
- visual template
- caption style
- account credentials
- upload destination
- posting cadence

Do not prematurely generalize the architecture until the current single-channel workflow is reliable.

---

# Current Working Pipeline

Current manual-assisted workflow:

```text
source video
→ timestamped transcript
→ Kimi selects strong clips
→ review selections
→ FFmpeg cuts each clip
→ 16:9 source becomes 9:16
→ local Whisper produces word timestamps
→ ASS kinetic captions
→ finished MP4 saved locally
→ TikTok upload attempted
→ manual posting fallback when necessary
```

Current command:

```bash
python3 process_source.py VIDEO TRANSCRIPT
```

Example:

```bash
python3 process_source.py podcast.mp4 source_transcript.txt
```

For manually chosen timestamps:

```bash
python3 run_pipeline.py VIDEO START END
```

Example:

```bash
python3 run_pipeline.py \
"SONNY ANGEL HAUL UNBOXING! WE PULLED A ROBBY - Deion's Days (1080p).mp4" \
00:04:56 \
00:05:25
```

---

# What Is Working

The difficult core pieces have now been proven.

## Source clipping

FFmpeg successfully cuts specified sections from horizontal source videos.

Fast seeking is performed before decoding:

```text
-ss START
-i VIDEO
```

Clip length is calculated as:

```text
duration = END - START
```

and FFmpeg uses:

```text
-t DURATION
```

This avoids the previous bug where moving `-ss` before `-i` while retaining `-to` caused clips to render far beyond the requested endpoint.

Example:

```text
04:56 → 05:25
```

correctly produces:

```text
29 seconds
```

---

# Default 9:16 Video Layout

The old layout center-cropped horizontal YouTube videos:

```text
crop=608:1080:656:0,scale=1080:1920
```

This discarded the left and right sides of the frame.

That was unacceptable for:

- product videos
- unboxings
- videos where hands move around the frame
- multi-person shots
- visual demonstrations

## New default: `fit_blur`

Horizontal 16:9 source video is preserved in full.

The composition is:

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

Output:

```text
1080x1920
9:16
```

Current FFmpeg filter:

```text
[0:v]split=2[bg][fg];
[bg]
scale=1080:1920:force_original_aspect_ratio=increase,
crop=1080:1920,
boxblur=20:1
[blurred];
[fg]
scale=1080:-2:force_original_aspect_ratio=decrease
[foreground];
[blurred][foreground]
overlay=(W-w)/2:(H-h)/2,
setsar=1
[v]
```

This has now been successfully tested on real Sonny Angel source videos.

The full horizontal frame remains visible and the blurred background makes the 9:16 canvas feel intentional.

---

# Caption Pipeline

Selected clips are sent through local Whisper.

Current sequence:

```text
pipeline_vertical.mp4
→ Whisper
→ pipeline_vertical.json
→ words.json
→ make_ass.py
→ captions.ass
→ FFmpeg burn
→ final_tiktok.mp4
```

Current caption style:

- approximately four words visible at once
- bold white text
- strong black outline
- currently spoken word highlighted yellow
- lower-middle safe area
- no caption bubble
- Avenir Next Condensed
- large short-form-friendly sizing

The timing has been visually tested and works well.

---

# Sonny Angel Proper-Noun Corrections

Whisper sometimes mishears Sonny-specific terms.

Existing correction:

```text
Sunny Angel
→
Sonny Angel
```

The Robby test revealed another recurring error:

```text
Ravi
→
Robby
```

The caption-processing layer should correct known domain-specific terms after Whisper transcription and before ASS markup.

This allows Whisper to remain generic while channel-specific vocabulary is corrected locally for free.

Future channel configurations may maintain their own vocabulary maps.

Example:

```text
channels/
    sonny/
        vocabulary.json
```

Possible entries:

```json
{
  "Sunny Angel": "Sonny Angel",
  "Sunny Angels": "Sonny Angels",
  "Ravi": "Robby"
}
```

---

# Permanent Output Files

`final_tiktok.mp4` remains the temporary working output.

Finished videos are also copied to permanent filenames before attempting TikTok upload.

Permanent filename structure includes:

- source filename
- clip start timestamp
- clip end timestamp

Example:

```text
outputs/
  SONNY ANGEL HAUL UNBOXING! WE PULLED A ROBBY - Deion's Days (1080p)/
    SONNY ANGEL HAUL UNBOXING! WE PULLED A ROBBY - Deion's Days (1080p)_000456-000525.mp4
```

If exactly the same clip is rendered again:

```text
..._000456-000525_v2.mp4
```

then:

```text
..._000456-000525_v3.mp4
```

etc.

Finished work should never be silently overwritten.

---

# Render-All-Even-If-TikTok-Fails

This is now a core reliability requirement.

TikTok delivery must never determine whether local production succeeds.

Desired architecture:

```text
render clip
→ burn captions
→ save permanent local copy
→ attempt TikTok upload
→ success OR failure
→ return normally
→ process next clip
```

TikTok upload errors are caught and printed rather than terminating the pipeline.

This is especially important for batch jobs.

Desired five-clip behavior:

```text
Clip 1
render
save
upload succeeds

Clip 2
render
save
upload succeeds

Clip 3
render
save
TikTok blocks upload
continue

Clip 4
render
save
TikTok blocks upload
continue

Clip 5
render
save
TikTok blocks upload
continue
```

The batch should complete with all five local videos regardless of TikTok status.

---

# TikTok API Behavior

Current integration uses TikTok's Content Posting API inbox/draft workflow.

Successful sequence:

```text
initialize upload
→ HTTP 201
→ PROCESSING_UPLOAD
→ SEND_TO_USER_INBOX
```

A successful inbox upload can then be completed from TikTok.

## Pending-share / anti-spam limit

During testing TikTok returned:

```text
spam_risk_too_many_pending_share
```

This occurred after several pending uploads already existed.

The practical lesson is:

```text
TikTok API delivery is rate/capacity constrained.
Local rendering must not be.
```

When the API refuses another upload:

```text
finished local file remains available
→ manually upload via TikTok desktop
```

Manual desktop upload is an acceptable fallback.

Current practical production time with manual upload is approximately:

```text
5–10 minutes per clip
```

which is acceptable for the current prototype.

---

# Current Sonny Account Workflow

Current production loop:

```text
find promising Sonny source video
→ obtain transcript
→ choose clip manually or via Kimi
→ run pipeline
→ inspect finished MP4
→ upload via API if available
→ otherwise upload through TikTok desktop
→ add caption
→ credit original creator
→ post
```

Creator credit should be included when using third-party source material.

Credit is good attribution practice but does not itself replace permission or other legal authorization to reuse copyrighted material.

---

# Kimi Integration

Kimi currently provides the editorial judgment layer.

Current model:

```text
kimi-k3
```

Current API style:

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["KIMI_API_KEY"],
    base_url="https://api.moonshot.ai/v1",
)
```

Kimi receives a timestamped transcript and returns structured JSON containing:

```text
rank
start
end
score
hook
category
reason
caption
```

Kimi has successfully identified useful Sonny Angel clips including:

- rare pulls
- Robby moments
- fake-vs-real advice
- collector controversy
- duplicates
- collecting tips

---

# Keeping Kimi Inference Costs Low

Inference cost should remain an explicit architectural constraint.

Approximately $0.50 of Kimi credit was used during one development/testing session.

That is manageable, but the system should be designed so a small balance can last for a long time.

## Core principle

Use Kimi for judgment.

Use local tools for mechanics.

```text
Kimi:
    identify promising moments
    rank hooks
    classify clips
    generate short captions

Local/free:
    downloads
    transcript extraction
    Whisper
    timestamp conversion
    video cutting
    caption timing
    FFmpeg
    output naming
    retries
    uploads
```

---

## One Kimi request per source

Preferred architecture:

```text
one compact transcript
→ ONE Kimi request
→ multiple ranked clips
→ caption suggestions
→ save JSON
```

Do not make separate inference calls for:

```text
clip ranking
caption writing
hook writing
categorization
```

Return everything in one structured response.

---

## Cache Kimi results

Once inference succeeds:

```text
clip_candidates.json
```

becomes a permanent editorial artifact.

Rendering the same source again must not call Kimi again.

Retries caused by:

- FFmpeg
- captions
- upload limits
- visual-template experiments
- TikTok failures

must all reuse the existing JSON.

Future ideal cache key:

```text
source hash
+
transcript hash
=
Kimi result
```

Potential behavior:

```bash
python3 process_source.py VIDEO TRANSCRIPT
```

checks the cache automatically.

Only:

```bash
--rerank
```

should deliberately call Kimi again.

---

## Compact transcripts

Do not send word-level Whisper JSON to Kimi.

For clip selection, use lines such as:

```text
[04:42-04:55] ...
[04:55-05:08] ...
[05:08-05:22] ...
```

Word-level timing is only necessary after a clip has already been selected.

This keeps prompt tokens much lower.

---

## Prefer existing YouTube captions

Preferred transcript order:

```text
1. official YouTube captions
2. YouTube auto-generated captions
3. local full-source Whisper
```

Full-source Whisper should be a fallback rather than the default when usable timed captions already exist.

Selected short clips can still use local Whisper for precise kinetic-caption timing.

---

## Remove low-signal transcript sections locally

Potential future preprocessing can remove:

- sponsor sections
- long intros
- housekeeping
- outros
- repeated calls to subscribe
- music-only segments
- long silent sections

This reduces Kimi input tokens.

Do not make the filtering overly aggressive.

Interesting payoff moments must remain available.

---

## Variable number of clips

Not every source needs five clips.

Future option:

```bash
--clips 3
```

Possible heuristic:

```text
short source
→ 3 candidates

medium source
→ 5 candidates

long source
→ 5–8 candidates
```

The goal is useful clips, not forcing a fixed quota.

---

## Reuse inference across visual styles

Once a clip is selected, it may later be rendered using:

```text
fit_blur
split_image
split_video
```

All render variants should reuse the same Kimi result.

Changing visuals must cost:

```text
$0 additional inference
```

---

# Future Visual Mode: Split-Screen Supporting Visuals

A future visual upgrade is a split-screen explainer format.

Concept:

```text
┌─────────────────────┐
│ source video        │
│                     │
├─────────────────────┤
│ supporting visual   │
│ image / B-roll /    │
│ screenshot / card   │
└─────────────────────┘
```

The supporting visual reinforces what is being discussed.

Examples:

```text
speaker mentions Robby
→ lower half shows Robby image

speaker discusses fake Sonnies
→ lower half shows comparison image

speaker mentions $400 collectible
→ lower half shows relevant figure/reference

speaker explains collector terminology
→ lower half shows text card
```

Potential render modes:

```text
fit_blur
split_image
split_video
smart_visual
```

Recommended rollout:

```text
Phase 1
manual image supplied by user

Phase 2
local reusable asset library

Phase 3
Kimi suggests supporting visual query/type

Phase 4
automatic visual retrieval or generation
```

Do not build automatic visual sourcing first.

---

# YouTube URL Ingestion — Next Major Feature

This is the next development milestone.

Target command:

```bash
python3 process_youtube.py "YOUTUBE_URL"
```

Desired behavior:

```text
YouTube URL
→ yt-dlp
→ download source video
→ fetch captions if available
→ normalize captions into timestamped transcript
→ Whisper fallback if necessary
→ Kimi selects clips once
→ cache selections
→ user reviews
→ render all clips
→ save all clips
→ attempt uploads
```

This removes the current manual steps of:

```text
download video manually
move video manually
copy transcript manually
```

Target implementation tool:

```text
yt-dlp
```

This feature is planned for the next build session.

Only download/reuse source content where authorized or otherwise legally permitted.

---

# Multi-Channel Direction

Long-term, the system may run several content pipelines.

Initial target set:

```text
1. Sonny Angel channel
2. lofi / ambient / white-noise channel
3. podcast / entertainment clipping channel
```

Examples of entertainment source categories may include highly viewed creator/podcast material, subject to rights/permission requirements.

The system should eventually route finished content to the appropriate destination automatically.

Concept:

```text
content source
→ channel configuration
→ channel-specific processing
→ channel-specific output
→ correct account
```

Potential configuration:

```text
channels/
    sonny.json
    lofi.json
    podcast_clips.json
```

Example conceptual configuration:

```json
{
  "channel": "sonny",
  "render_mode": "fit_blur",
  "caption_style": "kinetic",
  "num_clips": 5,
  "platform": "tiktok"
}
```

Another:

```json
{
  "channel": "lofi",
  "render_mode": "longform",
  "duration": "8h",
  "platform": "youtube"
}
```

Do not implement this abstraction until at least two workflows are functioning in reality.

---

# Lofi / White-Noise Pipeline Concept

This is a separate production workflow from clipping.

Possible future flow:

```text
audio source/generation
→ looping or evolving ambient composition
→ long-duration visual
→ render multi-hour video
→ thumbnail/title/description
→ upload to YouTube
```

Potential content:

- lofi beats
- rain
- fireplace ambience
- café noise
- sleep noise
- white/brown noise
- study ambience

Important:

Use audio and visuals the user owns, creates, licenses, or has permission to publish.

This pipeline should not share unnecessary complexity with the clipping pipeline.

---

# Future Automated Daily Workflow

Long-term target:

```text
scheduler starts job
↓
find or load source
↓
obtain transcript
↓
one model inference
↓
rank clips
↓
render all outputs
↓
save everything
↓
route to account
↓
upload where available
↓
human reviews exceptions
```

Eventually the human role should move from:

```text
producer
```

to:

```text
reviewer/editor
```

Instead of spending roughly an hour producing content each day, the goal is to spend approximately:

```text
10–15 minutes
```

reviewing a queue of already-generated outputs.

---

# Reliability Principles

## 1. Local success comes first

A finished local MP4 is the definition of successful rendering.

Third-party API failure is a distribution issue.

---

## 2. Save before uploading

Always:

```text
render
→ save permanent file
→ external upload
```

Never:

```text
render
→ upload
→ save
```

---

## 3. External failures must be non-fatal

Failures from:

- TikTok
- future YouTube APIs
- rate limits
- network outages

must not destroy or halt completed local work.

---

## 4. Cache paid inference

Never repay for an editorial decision that already exists.

---

## 5. Keep stable components stable

Current proven pieces include:

- FFmpeg source clipping
- fast seek + duration clipping
- blurred 9:16 format
- Whisper word timing
- ASS caption generation
- kinetic caption burn
- proper noun corrections
- TikTok authentication
- TikTok upload
- TikTok status polling
- Kimi structured clip selection
- unique output filenames

Prefer targeted changes over broad rewrites.

---

# Current Core Files

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

Current intermediate files include:

```text
pipeline_vertical.mp4
pipeline_vertical.json
words.json
captions.ass
final_tiktok.mp4
source_segments.json
source_transcript.txt
clip_candidates.json
```

Permanent outputs live under:

```text
outputs/
```

---

# Security

Before making source code public, ensure secrets are excluded.

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

Never commit:

```text
KIMI_API_KEY
TikTok client secret
TikTok access token
TikTok refresh token
```

Move any currently hardcoded client secrets into environment variables before publishing the repository.

---

# Common Debugging Notes

## Whisper warning

```text
FP16 is not supported on CPU; using FP32 instead
```

Expected.

Not an error.

---

## FFmpeg Late SEI warnings

Warnings such as:

```text
Late SEI is not implemented
```

have appeared with downloaded source files.

Rendering has still completed successfully.

Treat as non-fatal unless actual output fails.

---

## Clip unexpectedly runs for several minutes

Check that the command uses:

```text
-ss START
-i VIDEO
-t DURATION
```

Do not combine input-side `-ss` with the old absolute `-to END` logic.

---

## TikTok queue failure

```text
spam_risk_too_many_pending_share
```

Do not rerender.

Do not rerun Kimi.

Use the existing local MP4 and either:

- retry API delivery later
- manually upload using TikTok desktop

---

## Wrong Sonny terminology in captions

Correct domain terms locally before ASS generation.

Current known examples:

```text
Sunny → Sonny
Ravi → Robby
```

---

# Next Build Session

Highest-priority items:

1. Verify a full multi-clip batch continues after TikTok upload failure.
2. Add YouTube URL ingestion using `yt-dlp`.
3. Automatically retrieve timed YouTube captions where available.
4. Normalize VTT/SRT captions for Kimi.
5. Use Whisper only when usable captions are unavailable.
6. Cache Kimi inference based on source/transcript.
7. Build the first version of:

```bash
python3 process_youtube.py "URL"
```

After that:

8. Begin separating channel configurations.
9. Prototype second content workflow.
10. Add optional split-screen visual mode.
11. Eventually automate source discovery and scheduled production.

---

# Near-Term Target

The next major version should reduce the workflow to:

```text
paste YouTube URL
→ wait
→ review selected clips
→ receive finished local videos
→ post
```

Then:

```text
paste YouTube URL
```

becomes:

```text
automatically discover candidate source
```

and eventually daily content generation can run with minimal intervention.

---

# Current Project Status

The project has moved beyond proof-of-concept.

The following are now demonstrated:

```text
source video
→ selected moment
→ correct duration
→ 9:16 blurred composition
→ Whisper
→ kinetic captions
→ unique permanent MP4
→ TikTok delivery attempt
```

Kimi has also demonstrated successful editorial selection from long timestamped transcripts.

The remaining work is primarily:

```text
automation
reliability
source ingestion
inference caching
multi-channel routing
distribution
```

rather than proving that automated clipping itself is possible.

Next major milestone:

```text
YouTube URL
→ one model call
→ multiple cached selections
→ multiple finished local clips
→ upload wherever capacity is available
```
```

That leaves us in a very good place for tomorrow: **YouTube URL in → finished clips out** is the next major simplification.

And I agree with the broader direction, with one caveat: for the MrBeast/podcast-style channel, build around sources you have permission or another valid right to republish. For the lofi/white-noise channel, using your own or properly licensed/generated audio and visuals gives you a much cleaner path to scaling and monetization.
