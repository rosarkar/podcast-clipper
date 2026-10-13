# Podcast Clipper

A local Python-based workflow for turning long-form video into TikTok-ready vertical clips with word-level captions and uploading them to TikTok as drafts.

This project is intentionally simple and local-first. It was built to make clipping faster without relying on paid clipping software.

---

## What the pipeline does

The current V1 workflow is:

```text
source video
    ↓
choose start + end timestamps
    ↓
FFmpeg cuts the clip
    ↓
FFmpeg center-crops to 9:16
    ↓
Whisper transcribes with word timestamps
    ↓
custom Python caption generator fixes terminology
    ↓
ASS captions are generated
    ↓
active spoken word is highlighted in yellow
    ↓
FFmpeg burns captions into the video
    ↓
final MP4 is uploaded to TikTok
    ↓
script polls TikTok until the draft is sent to the account inbox
```

The main command is:

```bash
python3 run_pipeline.py podcast.mp4 00:01:00 00:01:30
```

That command takes the section from `1:00` to `1:30`, creates the final vertical captioned clip, uploads it to TikTok, and checks its processing status.

---

# Current project structure

```text
podcast-clipper/
│
├── podcast.mp4
│
├── clip.py
├── make_ass.py
├── run_pipeline.py
├── tiktok_auth.py
├── tiktok_upload.py
├── tiktok_status.py
│
├── tiktok_tokens.json
│
├── pipeline_vertical.mp4
├── pipeline_vertical.srt
├── pipeline_vertical.json / words.json
├── captions.ass
├── final_tiktok.mp4
│
├── clip.mp4
├── clip_tiktok.mp4
├── clip_tiktok.srt
├── clip_tiktok_captioned.mp4
├── clip_tiktok_captioned_v2.mp4
└── caption_test*.mp4
```

Some of these are early test files and can eventually be cleaned up.

The important production files are:

```text
run_pipeline.py
make_ass.py
tiktok_auth.py
tiktok_tokens.json
```

---

# Core files

## `run_pipeline.py`

This is the main orchestrator.

It accepts:

```text
video file
start timestamp
end timestamp
```

Example:

```bash
python3 run_pipeline.py podcast.mp4 00:04:20 00:05:02
```

It performs five major steps.

### Step 1 — Create vertical clip

FFmpeg:

1. reads the source video
2. cuts between the supplied timestamps
3. center-crops the original 16:9 video
4. scales the result to 1080×1920

Current filter:

```text
crop=608:1080:656:0,scale=1080:1920
```

This assumes a 1920×1080 source.

Current limitation: the crop is fixed to the center of the original video. It does not yet follow faces or speakers automatically.

Output:

```text
pipeline_vertical.mp4
```

---

### Step 2 — Generate word-level captions

`run_pipeline.py` calls:

```bash
python3 make_ass.py
```

`make_ass.py` runs Whisper locally and requests word-level timestamps.

Whisper model:

```text
small
```

Language:

```text
English
```

Whisper outputs JSON containing segments and word timing information.

The script then converts those timestamps into an ASS subtitle file.

Output:

```text
captions.ass
```

---

### Step 3 — Render captions

FFmpeg burns:

```text
captions.ass
```

onto:

```text
pipeline_vertical.mp4
```

and creates:

```text
final_tiktok.mp4
```

The final video contains permanent captions and can be uploaded directly.

---

### Step 4 — Upload to TikTok

The script loads the TikTok access token from:

```text
tiktok_tokens.json
```

Then it initializes a TikTok Content Posting API upload using:

```text
FILE_UPLOAD
```

TikTok returns:

```text
upload_url
publish_id
```

The script then sends the entire MP4 to the supplied `upload_url`.

A successful transfer currently returns:

```text
HTTP status: 201
```

---

### Step 5 — Check TikTok processing status

After upload, the script checks TikTok every five seconds.

Typical sequence:

```text
PROCESSING_UPLOAD
PROCESSING_UPLOAD
PROCESSING_UPLOAD
SEND_TO_USER_INBOX
```

When TikTok returns:

```text
SEND_TO_USER_INBOX
```

the clip is available through the TikTok account's system notifications/inbox so it can be opened, reviewed, edited further, and posted.

---

# Caption system

## Why ASS instead of SRT?

The first version used SRT subtitles.

That worked for basic captions but did not provide enough styling control.

The project now uses:

```text
ASS — Advanced SubStation Alpha
```

because ASS allows:

- per-word colors
- font selection
- outline thickness
- positioning
- shadows
- word-level style changes
- future animation support

FFmpeg renders the ASS file using `libass`.

---

## Current caption style

The current visual style is:

- approximately four words visible at a time
- white text
- strong black outline
- active spoken word highlighted yellow
- captions positioned in the lower-middle safe area
- word highlighting synchronized to Whisper word timestamps

The active word changes exactly when Whisper says that word is spoken.

This produces a familiar TikTok / Shorts kinetic-caption look without requiring external caption software.

Current font:

```text
Avenir Next Condensed
```

Fallback behavior depends on what fonts are installed on the Mac.

---

## Proper noun correction

Whisper may misunderstand brand names or niche terminology.

Example:

```text
Sunny Angels
```

was being transcribed instead of:

```text
Sonny Angels
```

`make_ass.py` contains a correction step that operates on the word list before subtitle styling is added.

Current example rule:

```text
Sunny Angel  → Sonny Angel
Sunny Angels → Sonny Angels
```

Future terminology can be added to the same correction function.

Examples that may be useful later:

```text
person names
podcast names
company names
crypto terminology
product names
acronyms
```

---

# TikTok authentication

## TikTok developer configuration

The project uses a TikTok Developer app called:

```text
Podcast Clipper
```

A Sandbox configuration was created for testing.

Products enabled:

```text
Login Kit
Content Posting API
```

Scopes:

```text
user.info.basic
video.upload
```

The clipping TikTok account was added as a Sandbox Target User.

Direct Post is currently OFF.

The workflow uploads clips as drafts instead of automatically publishing them publicly.

---

## Desktop OAuth redirect

The configured desktop redirect URI is:

```text
http://127.0.0.1:3455/callback/
```

`tiktok_auth.py` temporarily starts a local HTTP server on port:

```text
3455
```

TikTok redirects back to this address after the user approves authorization.

---

# `tiktok_auth.py`

This script handles TikTok OAuth.

It:

1. generates a random OAuth `state`
2. generates a PKCE verifier
3. creates the PKCE SHA-256 challenge
4. opens TikTok's authorization page
5. waits for TikTok to redirect to the local callback server
6. verifies the returned state
7. receives the authorization code
8. exchanges the code for TikTok tokens
9. stores them locally

Output:

```text
tiktok_tokens.json
```

The token file contains fields similar to:

```text
access_token
refresh_token
expires_in
refresh_expires_in
open_id
scope
token_type
```

Do not commit this file to GitHub.

---

# Important security notes

These values should NEVER be committed to the public repository:

```text
TikTok client secret
access tokens
refresh tokens
tiktok_tokens.json
```

The current project should eventually have a `.gitignore` containing:

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

If the TikTok client key/client secret are currently hardcoded inside `tiktok_auth.py`, move them into environment variables before making the repository public.

A future structure could use:

```text
.env
```

with:

```text
TIKTOK_CLIENT_KEY=...
TIKTOK_CLIENT_SECRET=...
```

and Python could read them using environment variables.

---

# Local dependencies

## Python

Originally tested with:

```text
Python 3.9.6
```

Check:

```bash
python3 --version
```

---

## FFmpeg

Installed through Homebrew.

The regular FFmpeg formula did not include subtitle rendering, so the project uses:

```bash
brew install ffmpeg-full
```

`ffmpeg-full` includes:

```text
libass
```

which is necessary for ASS subtitle rendering.

The shell PATH was updated so `ffmpeg-full` is used first.

Check:

```bash
ffmpeg -filters | grep subtitles
```

Expected filters include:

```text
ass
subtitles
```

---

## Whisper

Installed locally with:

```bash
pip3 install openai-whisper
```

On this Mac, the Whisper executable was installed at:

```text
/Users/rohitsarkar/Library/Python/3.9/bin/whisper
```

`make_ass.py` currently references that path directly.

Whisper may display:

```text
FP16 is not supported on CPU; using FP32 instead
```

This is normal when running on CPU and does not indicate a failure.

---

# Useful commands

Go to the project:

```bash
cd ~/podcast-clipper
```

List files:

```bash
ls
```

Open the folder in Finder:

```bash
open .
```

Run the complete pipeline:

```bash
python3 run_pipeline.py podcast.mp4 00:01:00 00:01:30
```

Open the final rendered clip:

```bash
open final_tiktok.mp4
```

Test Python syntax:

```bash
python3 -m py_compile run_pipeline.py
```

Generate captions manually:

```bash
python3 make_ass.py
```

Inspect corrected terminology:

```bash
grep -i "sonny" captions.ass
```

Authenticate TikTok again:

```bash
python3 tiktok_auth.py
```

---

# Early test scripts

These were useful while building the system but are no longer central to the final workflow.

## `clip.py`

Original proof of concept.

It demonstrated that Python could call FFmpeg and cut:

```text
podcast.mp4
```

between fixed timestamps.

It later added the center 9:16 crop.

---

## `tiktok_upload.py`

Standalone proof of concept for TikTok draft uploads.

It:

1. initialized an upload
2. received an upload URL
3. uploaded the local MP4
4. returned HTTP 201

Its functionality now exists inside `run_pipeline.py`.

---

## `tiktok_status.py`

Standalone TikTok status checker.

It was used to confirm that successful uploads moved from:

```text
PROCESSING_UPLOAD
```

to:

```text
SEND_TO_USER_INBOX
```

Its functionality also now exists inside `run_pipeline.py`.

---

# Generated files

These files are outputs rather than source code.

## `pipeline_vertical.mp4`

Temporary 1080×1920 crop produced from the selected source section.

## `words.json`

Whisper transcription data including word-level timestamps.

## `captions.ass`

Styled word-level captions generated from Whisper timestamps.

## `final_tiktok.mp4`

The final captioned vertical video that is sent to TikTok.

---

# Known limitations

## Fixed center crop

The current crop assumes that the important content is near the center.

Future improvement:

```text
face detection
speaker tracking
dynamic crop positioning
```

---

## Manual clip selection

The user currently supplies:

```text
start timestamp
end timestamp
```

This is the biggest feature still missing.

The next major phase of development is:

```text
full podcast
    ↓
transcript
    ↓
AI analyzes transcript
    ↓
candidate clips
    ↓
rank candidates
    ↓
choose best timestamps
    ↓
send each timestamp range through run_pipeline.py
```

---

## TikTok posting is draft-based

The current project uses:

```text
Upload to TikTok
```

rather than Direct Post.

The result is sent to the authorized TikTok account's inbox where the user finishes the publishing step.

This is intentional for V1 because it preserves human review before publishing.

---

## Access token expiration

`tiktok_tokens.json` will eventually expire.

The project currently does not automatically refresh expired access tokens.

Future improvement:

```text
detect token expiration
    ↓
use refresh_token
    ↓
save new token data automatically
```

---

# Next development phase: automatic clip selection

The video-production pipeline is now considered V1 complete.

The next objective is to reduce this:

```bash
python3 run_pipeline.py podcast.mp4 00:01:00 00:01:30
```

to something closer to:

```bash
python3 find_clips.py podcast.mp4
```

The planned content-selection system should:

1. transcribe the complete podcast
2. preserve accurate timestamps
3. split the transcript into candidate windows
4. identify self-contained moments
5. score each candidate for short-form potential
6. rank the strongest clips
7. return exact start/end timestamps
8. optionally run the best selections through the existing production pipeline

Potential clip-scoring dimensions:

```text
Hook strength
Standalone clarity
Novelty
Emotional intensity
Conflict / disagreement
Surprise
Shareability
Recognizable subject
Story completeness
Useful insight
```

Eventually the goal could be:

```text
python3 process_podcast.py podcast.mp4
```

which might produce:

```text
Candidate 1
00:18:42 → 00:19:17
Score: 9.2

Candidate 2
00:31:08 → 00:31:46
Score: 8.8

Candidate 3
00:47:13 → 00:47:58
Score: 8.5
```

The user can approve clips, after which the existing rendering/upload pipeline handles everything else.

---

# Long-term architecture

```text
                ┌─────────────────────┐
                │   Long-form video   │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │      Whisper        │
                │ Full transcription  │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  AI Clip Selector   │
                │ score + timestamps  │
                └──────────┬──────────┘
                           │
                    User approval
                           │
                           ▼
                ┌─────────────────────┐
                │   run_pipeline.py   │
                └──────────┬──────────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          FFmpeg        Whisper       make_ass
           crop          words         captions
             └─────────────┼─────────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  final_tiktok.mp4   │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ TikTok Posting API  │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ TikTok Inbox Draft  │
                └─────────────────────┘
```

---

# V1 status

As of the current checkpoint:

```text
[✓] Cut source video by timestamp
[✓] Convert horizontal video to vertical 9:16
[✓] Local Whisper transcription
[✓] Word-level timestamps
[✓] Correct recurring proper nouns
[✓] Generate ASS subtitles
[✓] Highlight currently spoken word
[✓] Burn captions into video
[✓] Produce final TikTok-ready MP4
[✓] TikTok OAuth
[✓] TikTok sandbox target account
[✓] Upload local video to TikTok
[✓] Poll processing status
[✓] Deliver draft to TikTok inbox

[ ] Automatic clip selection
[ ] Speaker-aware dynamic cropping
[ ] Automatic OAuth token refresh
[ ] Batch generation of multiple clips
[ ] Performance feedback / analytics loop
[ ] Optional Direct Post
```

The current production and TikTok draft-upload pipeline is working end to end.
