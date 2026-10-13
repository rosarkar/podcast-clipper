# BUILD.md

## Purpose

This file is the working build log and development roadmap for the Podcast Clipper project.

`README.md` explains what the project is and how the pieces fit together.

`BUILD.md` is for active development:

- what was built
- what changed
- what broke
- how it was debugged
- what is working now
- what should be built next
- daily workflow
- future ideas
- technical debt
- experiments
- decisions we do not want to forget

The goal is to make it easy to return to the project after days or weeks away and immediately understand the current state.

---

# Project goal

Build a local-first clipping system that can turn long-form video into short-form TikTok content with as little manual work as possible.

Current target use case:

```text
YouTube / long-form Sonny Angel video
    ↓
download source
    ↓
transcribe locally
    ↓
AI finds the strongest Sonny-specific moments
    ↓
select candidate clips
    ↓
convert to TikTok format
    ↓
generate kinetic captions
    ↓
upload to TikTok drafts
    ↓
human reviews and posts
```

The first live account is intended to behave like a normal Sonny Angel fan account.

Early goal:

```text
build an audience of people who care about Sonny Angels
```

Longer-term commercial use may include routing relevant followers toward a Whatnot shop, but the account should not initially look or feel like a storefront.

The underlying software should eventually be reusable for other niche clipping accounts.

---

# Development principles

## 1. Local-first

Keep as much processing local as possible.

Current local components:

```text
FFmpeg
Whisper
Python
ASS subtitle generation
video rendering
```

Benefits:

- no per-video editing subscription
- no captioning credits
- lower recurring cost
- more control
- easier customization
- private source processing

AI inference for clip selection may use an external model API because transcript analysis is relatively cheap compared with video processing.

---

## 2. Human approval before publishing

For now, automation should stop before final public posting.

Preferred flow:

```text
AI proposes clips
    ↓
user approves
    ↓
pipeline renders
    ↓
TikTok inbox/draft
    ↓
user reviews and posts
```

This allows us to improve selection quality without accidentally publishing weak clips.

---

## 3. Build the simplest useful version first

Do not solve every possible edge case before the core workflow works.

Current pattern:

```text
prove one step
→ make it reliable
→ integrate it
→ move to the next bottleneck
```

This is how V1 was built.

---

## 4. Editorial quality matters more than full automation

The main bottleneck is no longer rendering.

The next bottleneck is:

```text
which 20–60 seconds should become a clip?
```

A system that automatically generates mediocre clips is less useful than one that surfaces excellent clips for quick approval.

---

# Current V1 status

As of the current checkpoint:

```text
[✓] Cut a source video by timestamp
[✓] Convert 16:9 source into 9:16 TikTok format
[✓] Run Whisper locally
[✓] Get word-level timestamps
[✓] Generate ASS captions
[✓] Highlight the currently spoken word in yellow
[✓] Use white text with strong black outline
[✓] Position captions in TikTok-safe lower-middle area
[✓] Correct recurring proper nouns such as Sonny Angel
[✓] Burn captions into final MP4
[✓] Authenticate with TikTok
[✓] Upload video through TikTok Content Posting API
[✓] Poll TikTok processing status
[✓] Deliver video to TikTok inbox/drafts
[✓] End-to-end one-command pipeline works

[ ] Automatically download source video
[ ] Transcribe complete source video
[ ] AI-based clip selection
[ ] Candidate clip scoring
[ ] Human clip approval interface
[ ] Better 16:9 → 9:16 framing
[ ] Batch multiple clips from one source
[ ] Automatic token refresh
[ ] Analytics / performance feedback loop
```

---

# Current one-command workflow

Example:

```bash
python3 run_pipeline.py podcast.mp4 00:01:00 00:01:30
```

Current flow:

```text
podcast.mp4
    ↓
run_pipeline.py
    ↓
FFmpeg cuts requested timestamps
    ↓
FFmpeg creates 1080×1920 crop
    ↓
pipeline_vertical.mp4
    ↓
make_ass.py
    ↓
Whisper transcription
    ↓
word timestamps
    ↓
proper noun correction
    ↓
captions.ass
    ↓
FFmpeg burns captions
    ↓
final_tiktok.mp4
    ↓
TikTok upload initialization
    ↓
MP4 upload
    ↓
status polling
    ↓
SEND_TO_USER_INBOX
```

---

# Current important files

## `run_pipeline.py`

Main orchestrator.

Responsibilities:

- accept input video
- accept start time
- accept end time
- create vertical clip
- call caption generator
- render captions
- upload final clip
- poll TikTok status

Example:

```bash
python3 run_pipeline.py podcast.mp4 00:01:00 00:01:30
```

---

## `make_ass.py`

Caption generator.

Responsibilities:

- run Whisper locally
- request word timestamps
- parse Whisper JSON
- correct known terminology
- group words into short caption chunks
- create per-word ASS timing events
- highlight active word
- write `captions.ass`

Current visual style:

```text
Font: Avenir Next Condensed
Text: white
Active word: yellow
Outline: black
Approximate words on screen: 4
Position: lower-middle
```

Current terminology correction includes:

```text
Sunny Angel  → Sonny Angel
Sunny Angels → Sonny Angels
```

---

## `tiktok_auth.py`

Handles TikTok OAuth.

Responsibilities:

- create PKCE verifier/challenge
- launch TikTok authorization
- receive callback on localhost
- exchange authorization code for tokens
- save tokens locally

Token file:

```text
tiktok_tokens.json
```

This file must never be committed publicly.

---

## `tiktok_upload.py`

Older standalone upload proof of concept.

Its functionality is now mostly integrated into:

```text
run_pipeline.py
```

Keep temporarily for troubleshooting.

---

## `tiktok_status.py`

Older standalone TikTok status checker.

Its functionality is now integrated into:

```text
run_pipeline.py
```

Keep temporarily for troubleshooting.

---

## `clip.py`

Original FFmpeg clipping proof of concept.

Not required by the final integrated pipeline, but useful as a simple reference implementation.

---

# Current generated files

These are temporary/output files rather than core source code.

```text
pipeline_vertical.mp4
words.json
captions.ass
final_tiktok.mp4
```

Older development/test outputs may also exist:

```text
clip.mp4
clip_tiktok.mp4
clip_tiktok.srt
clip_tiktok_captioned.mp4
clip_tiktok_captioned_v2.mp4
caption_test.mp4
caption_test_v2.mp4
pipeline_vertical.srt
```

These can eventually be cleaned up.

---

# Current dependencies

## Python

Tested with:

```text
Python 3.9.6
```

---

## FFmpeg

Using Homebrew `ffmpeg-full`.

Reason:

The normal FFmpeg install did not include the subtitle rendering support needed for ASS subtitles.

Required FFmpeg filters:

```text
ass
subtitles
```

Check:

```bash
ffmpeg -filters | grep subtitles
```

---

## Whisper

Installed locally.

Current executable path:

```text
/Users/rohitsarkar/Library/Python/3.9/bin/whisper
```

Current model:

```text
small
```

Normal warning:

```text
FP16 is not supported on CPU; using FP32 instead
```

This is expected on CPU and is not an error.

---

# TikTok setup

TikTok developer app:

```text
Podcast Clipper
```

Products:

```text
Login Kit
Content Posting API
```

Scopes:

```text
user.info.basic
video.upload
```

Desktop OAuth callback:

```text
http://127.0.0.1:3455/callback/
```

Current posting approach:

```text
Upload to TikTok / inbox
```

not:

```text
Direct Post
```

This preserves manual review before publishing.

---

# Security checklist

Before uploading the full source code to GitHub:

```text
[ ] Ensure tiktok_tokens.json is ignored
[ ] Ensure TikTok client secret is not hardcoded in committed source
[ ] Add .env support if necessary
[ ] Add .gitignore
[ ] Ignore local MP4 files
[ ] Ignore generated ASS/SRT/JSON files
[ ] Ignore __pycache__
[ ] Check git history for accidental secrets
```

Suggested `.gitignore`:

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

---

# Known issue: vertical framing

## Problem

The current vertical conversion takes a fixed center crop from a 1920×1080 source.

Current filter:

```text
crop=608:1080:656:0,scale=1080:1920
```

This means the left and right portions of the original frame are removed.

That works when the important subject is centered.

It fails when:

- a Sonny Angel is being held near the edge
- a product box is on one side
- text appears on the left/right
- two people are speaking
- the creator points to something outside the center
- a collection/display spans most of the source frame

---

## Planned framing modes

### Mode 1 — `center`

Current behavior.

```text
16:9 source
    ↓
take vertical slice from center
    ↓
fill entire 9:16 screen
```

Advantages:

- fills screen
- visually native to TikTok
- simple
- fast

Disadvantage:

- cuts off side content

---

### Mode 2 — `fit_blur`

Recommended next improvement.

```text
full 16:9 source
    ↓
scale entire frame so nothing is lost
    ↓
place it over a blurred 9:16 background
```

Advantages:

- preserves everything in original frame
- safe for product videos
- no AI/vision required
- relatively easy with FFmpeg

Disadvantage:

- original video occupies less vertical space
- less immersive than a perfect smart crop

This should probably become the default for Sonny Angel content.

---

### Mode 3 — `smart_follow`

Future improvement.

Possible approach:

```text
detect face / person / important object
    ↓
estimate subject location
    ↓
move vertical crop horizontally over time
```

Potential technologies:

```text
OpenCV
MediaPipe
face detection
object tracking
scene analysis
```

This is not necessary for the next build.

---

# Next major feature: source acquisition

## Goal

Accept a YouTube URL instead of requiring the user to manually download a source video.

Target command:

```bash
python3 process_youtube.py "YOUTUBE_URL"
```

Possible tool:

```text
yt-dlp
```

Planned flow:

```text
YouTube URL
    ↓
yt-dlp
    ↓
source.mp4
```

Important:

Only use source material the user has permission/right to download and reuse.

---

# Next major feature: full-video transcription

Currently Whisper transcribes only the selected clip.

For automatic clip discovery we need:

```text
whole source video
    ↓
one full transcript
    ↓
timestamps preserved
```

The source should only be transcribed once.

Potential output:

```text
source_transcript.json
```

Example structure:

```json
[
  {
    "start": 0.0,
    "end": 8.2,
    "text": "..."
  },
  {
    "start": 8.2,
    "end": 14.8,
    "text": "..."
  }
]
```

Word-level timestamps can also be preserved if needed.

---

# Next major feature: AI clip selection

## Goal

Use an inexpensive model API to evaluate the transcript and identify strong short-form moments.

Current idea:

```text
Kimi
```

or another inexpensive model available through the user's API credits.

The model should not simply identify generally interesting moments.

For the Sonny Angel account it should optimize specifically for:

```text
Sonny Angel relevance
collector curiosity
strong opening hook
rare / secret / Robby discussion
interesting opinion
surprise
emotion
display ideas
collecting advice
release discussion
value / rarity discussion
comment potential
self-contained story
clear ending
```

---

# Proposed clip-scoring rubric

Initial weights to experiment with:

```text
Sonny relevance          25%
Hook strength            20%
Collector curiosity      15%
Comment potential        15%
Standalone clarity       10%
Novelty                  10%
Purchase-adjacent intent  5%
```

This is only a starting point.

We should change the weights based on real TikTok performance.

---

# Desired AI response format

Do not ask the model for free-form prose.

Request structured output.

Example:

```json
{
  "clips": [
    {
      "start": "00:03:14",
      "end": "00:03:47",
      "score": 9.3,
      "hook": "Why Robby is so hard to pull",
      "reason": "Strong collector curiosity and self-contained explanation"
    },
    {
      "start": "00:08:21",
      "end": "00:08:58",
      "score": 8.8,
      "hook": "The best way to display Sonny Angels",
      "reason": "Useful collector advice with good comment potential"
    }
  ]
}
```

Structured JSON makes it easy for Python to consume the result.

---

# Planned human approval step

Initial version:

```text
AI returns top candidates
    ↓
Terminal prints list
    ↓
user chooses clip numbers
    ↓
approved clips are rendered
```

Example:

```text
Candidate 1
00:03:14 → 00:03:47
Score: 9.3
Why: rare Robby discussion

Candidate 2
00:08:21 → 00:08:58
Score: 8.8
Why: display advice

Candidate 3
00:12:02 → 00:12:39
Score: 8.5
Why: collector opinion

Render which clips?
> 1,3
```

This is enough for V2.

No GUI is necessary yet.

---

# Planned V2 architecture

```text
YouTube URL
    ↓
download_youtube()
    ↓
source_video.mp4
    ↓
transcribe_full_video()
    ↓
source_transcript.json
    ↓
select_clips_with_ai()
    ↓
clip_candidates.json
    ↓
show candidates
    ↓
human approval
    ↓
for each approved clip:
    ↓
run_pipeline.py
    ↓
TikTok inbox
```

---

# Possible future V3

Once clip selection is reliable:

```text
YouTube URL
    ↓
download
    ↓
transcribe
    ↓
AI selects 5 candidates
    ↓
automatic quality filters
    ↓
render all candidates
    ↓
upload all to drafts
```

The user can then choose which drafts to post from TikTok.

---

# Possible future V4: feedback loop

Eventually the system could learn from post performance.

Store metrics such as:

```text
views
average watch time
completion rate
likes
comments
shares
saves
followers gained
```

Then compare those metrics with:

```text
clip topic
clip length
opening hook
caption style
source creator
AI score
content category
```

Over time, the selection model could be prompted using examples of clips that actually performed well.

Conceptually:

```text
AI prediction
    ↓
post
    ↓
real TikTok performance
    ↓
update what "good clip" means
```

This is much later.

---

# Sonny account strategy

Current TikTok account goal:

```text
look like a genuine Sonny Angel fan account
```

Do not initially position it as:

```text
shop
Whatnot seller
commercial account
```

Initial audience:

```text
Sonny Angel collectors
people interested in Sonny Angels
people discovering Sonny Angels
people who enjoy unboxings / rare pulls / collecting
```

Early content priorities:

```text
rare pulls
Robby
secret figures
collections
display ideas
series rankings
collector opinions
new releases
funny collector moments
value / rarity
which-one-would-you-pick prompts
collecting tips
```

Early success metric:

```text
Would a Sonny Angel fan follow this page?
```

not:

```text
Did this video make a sale?
```

---

# Future account architecture

The clipping system should eventually support multiple accounts.

Possible structure:

```text
accounts/
│
├── sonnylover808.json
├── another_niche.json
└── another_clip_account.json
```

Each account config could eventually define:

```text
account name
TikTok credentials/token file
caption style
proper noun dictionary
clip-selection prompt
content categories
minimum clip duration
maximum clip duration
crop mode
AI model
hashtags
description style
```

Example:

```json
{
  "account": "sonnylover808",
  "crop_mode": "fit_blur",
  "clip_min_seconds": 20,
  "clip_max_seconds": 55,
  "caption_group_size": 4,
  "highlight_color": "yellow",
  "topics": [
    "Sonny Angel",
    "Robby",
    "secret figures",
    "collections",
    "display ideas"
  ]
}
```

Do not build this abstraction until there is a second real account.

---

# Debugging guide

## Pipeline Python syntax error

Run:

```bash
python3 -m py_compile run_pipeline.py
```

No output means syntax is valid.

For caption script:

```bash
python3 -m py_compile make_ass.py
```

---

## Whisper warning about FP16

Message:

```text
FP16 is not supported on CPU; using FP32 instead
```

Status:

```text
NORMAL
```

Ignore unless another error follows.

---

## Whisper transcribes "Sunny Angel"

Check generated captions:

```bash
grep -i "sonny" captions.ass
```

If nothing appears:

1. check `fix_proper_nouns()`
2. confirm correction occurs before ASS markup
3. inspect `words.json`
4. check punctuation variants

---

## Captions do not render

Check FFmpeg subtitle support:

```bash
ffmpeg -filters | grep subtitles
```

Expected:

```text
ass
subtitles
```

If missing, confirm `ffmpeg-full` is first in PATH.

---

## Wrong font

FFmpeg/libass may substitute another font if the requested font is unavailable.

Check installed font names or inspect FFmpeg font selection logs.

The pipeline should still work even if font substitution occurs.

---

## Video is missing content from the sides

Cause:

```text
fixed center crop
```

Current solution:

```text
none yet
```

Next solution:

```text
implement fit_blur crop mode
```

---

## TikTok upload succeeds but video is not in normal drafts

This is expected for the current API route.

The video may appear through:

```text
TikTok system notifications / inbox
```

Typical terminal status:

```text
SEND_TO_USER_INBOX
```

---

## TikTok upload remains PROCESSING_UPLOAD

The script polls every few seconds.

If it remains there:

1. wait longer
2. run the standalone status checker if needed
3. verify the uploaded MP4 is valid
4. check TikTok API response/error fields

---

## TikTok token expires

Current V1 does not automatically refresh tokens.

Short-term fix:

```bash
python3 tiktok_auth.py
```

Future fix:

```text
automatic refresh-token flow
```

---

## TikTok authentication fails

Check:

```text
redirect URI
client key
client secret
requested scopes
sandbox target user
local callback port
```

Configured callback:

```text
http://127.0.0.1:3455/callback/
```

Never paste secrets or token files into support chats.

---

## Full pipeline fails halfway through

Work backward through the generated files.

### Step 1

Does this exist?

```text
pipeline_vertical.mp4
```

If not, problem is probably FFmpeg clipping/cropping.

### Step 2

Does this exist?

```text
words.json
```

If not, problem is probably Whisper.

### Step 3

Does this exist?

```text
captions.ass
```

If not, problem is probably caption-generation Python.

### Step 4

Does this exist?

```text
final_tiktok.mp4
```

If not, problem is probably FFmpeg ASS rendering.

### Step 5

If final video exists but upload fails:

```text
problem is TikTok/API/auth layer
```

This isolation approach should be used before changing unrelated parts of the system.

---

# Development workflow for each session

At the beginning of a build session:

```text
1. Open README.md
2. Open BUILD.md
3. Read "Current checkpoint"
4. Run the last known working command if necessary
5. Work on only the next bottleneck
```

During development:

```text
1. Change one subsystem at a time
2. Test it independently
3. Confirm output
4. Integrate it
5. Test end to end
6. Update BUILD.md
```

At the end of a session:

```text
1. Record what changed
2. Record what now works
3. Record any unresolved bugs
4. Record the exact next task
5. Save useful commands
6. Commit working source code when appropriate
```

---

# Daily build log

## 2026-10-13 — V1 end-to-end pipeline

### Goal

Create a working local system that turns a source video section into a captioned TikTok-ready clip and sends it to TikTok.

### Built

```text
FFmpeg clipping
vertical conversion
local Whisper transcription
word timestamps
ASS caption generation
active-word highlighting
proper noun correction
TikTok OAuth
TikTok video upload
TikTok status polling
one-command pipeline
```

### Caption result

Current design:

```text
white bold captions
black outline
yellow active word
~4 words at a time
lower-middle placement
```

Timing matched the spoken audio well enough for V1.

### Proper noun test

Whisper originally produced:

```text
Sunny Angels
```

The subtitle correction layer successfully converted it to:

```text
Sonny Angels
```

Confirmed directly inside:

```text
captions.ass
```

### TikTok result

Pipeline successfully reached:

```text
SEND_TO_USER_INBOX
```

and the video appeared in TikTok.

### V1 conclusion

Video-generation/upload pipeline is considered:

```text
GOOD ENOUGH / COMPLETE FOR NOW
```

Do not keep polishing it before solving clip selection.

---

# Next session plan

## Priority 1 — improve framing

Implement:

```text
fit_blur
```

before processing a large number of real clips.

Reason:

The existing fixed center crop can remove important Sonny/product content from the sides.

Success condition:

```text
entire original 16:9 frame remains visible
final output is 1080×1920
background fills vertical frame
captions remain readable
```

---

## Priority 2 — YouTube input

Install/test:

```text
yt-dlp
```

Create a simple source downloader.

Target:

```bash
python3 download_youtube.py "URL"
```

Output:

```text
source.mp4
```

Do not integrate everything immediately.

First verify downloading works independently.

---

## Priority 3 — full transcription

Create a script that transcribes the entire downloaded video only once.

Target:

```bash
python3 transcribe_source.py source.mp4
```

Output:

```text
source_transcript.json
```

Need:

```text
segment start time
segment end time
text
```

---

## Priority 4 — AI clip selection

Connect the selected model API.

Target:

```bash
python3 find_clips.py source_transcript.json
```

Output:

```text
clip_candidates.json
```

Start with approximately:

```text
5–10 candidates
20–60 seconds each
```

Each candidate should contain:

```text
start
end
score
hook
reason
```

---

## Priority 5 — approval + rendering

Show candidate list in Terminal.

Allow:

```text
1,2,5
```

Then send those timestamp ranges through the existing working pipeline.

---

# Current checkpoint

The project is currently at:

```text
V1 RENDERING + TIKTOK DELIVERY COMPLETE
```

The next bottleneck is:

```text
SOURCE → BEST TIMESTAMPS
```

The recommended order is:

```text
1. fix framing with fit_blur
2. add YouTube downloading
3. transcribe entire source
4. connect AI clip selector
5. return structured candidates
6. add human approval
7. batch through existing renderer
```

Do not rewrite the existing TikTok upload system unless it breaks.

Do not redesign captions unless real posting data gives a reason.

The main objective now is to increase:

```text
quality clips produced per unit of user effort
```

while preserving enough human control to learn what performs.

---

# Session-note template

Copy this block for future build days.

```markdown
## YYYY-MM-DD — Session title

### Goal

What were we trying to accomplish?

### Changes

What files changed?

### Built

What new functionality works?

### Tests

What commands were run?

What output confirmed success?

### Bugs encountered

What failed?

### Root cause

Why did it fail?

### Fix

What resolved it?

### Remaining issues

What still needs work?

### Decisions

Any architecture/product decisions worth remembering?

### Next task

What should we do first in the next session?

### Last known working command

```bash
COMMAND_HERE
```
```

---

# Guiding product question

Whenever deciding what to build next, ask:

> Does this reduce the effort required to consistently produce good clips?

If yes, it is probably useful.

If it only makes the system more technically impressive without improving clip quality, speed, reliability, or audience growth, it can wait.
