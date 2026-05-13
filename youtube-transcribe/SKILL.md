---
name: youtube-transcribe
description: >
  Transcribes a video file using OpenAI Whisper (GPU-accelerated when available)
  and generates an SEO-friendly YouTube description with timestamped chapters.
  Trigger on: "transcribe this video", "create youtube description",
  "add timestamps", "make chapters", "video to text", "generate SRT",
  "whisper transcription", or any request combining a video file with
  youtube/description/transcript/timestamps/chapters. Also trigger when
  the user uploads or points to any video file and wants any kind of
  text output from it — subtitles, captions, chapters, or a description.
---

# YouTube Transcribe

## Step 0 — GPU + dependency check (always run before transcribing)
If `youtube-transcribe/transcript.json` doesn't already exist, run this check first:

```python
import importlib, sys

def check(pkg, import_name=None):
    try:
        m = importlib.import_module(import_name or pkg)
        return True, getattr(m, "__version__", "?")
    except ImportError:
        return False, None

whisper_ok, whisper_ver = check("openai-whisper", "whisper")
torch_ok, torch_ver = check("torch")

print(f"openai-whisper : {'✅ ' + whisper_ver if whisper_ok else '❌ missing'}")
print(f"torch          : {'✅ ' + torch_ver if torch_ok else '❌ missing'}")

if torch_ok:
    import torch
    cuda = torch.cuda.is_available()
    print(f"CUDA available : {'✅ ' + torch.version.cuda if cuda else '❌ no'}")
    if cuda:
        print(f"GPU            : {torch.cuda.get_device_name(0)}")
    else:
        print()
        print("⚠️  torch is installed but CUDA is NOT available.")
        print("   This almost always means you have the CPU-only torch wheel.")
        print("   Check your torch build:")
        print(f"   torch version  : {torch.__version__}")
        print(f"   torch.version.cuda : {torch.version.cuda}")
        print()
        print("   Fix: reinstall torch with CUDA support.")
        print("   → Go to https://pytorch.org/get-started/locally/")
        print("     Select your OS, pip, Python, and CUDA version.")
        print("     Run the generated command (it will look like:")
        print("     pip install torch --index-url https://download.pytorch.org/whl/cuXXX)")
        print()
        print("   Do NOT proceed until CUDA shows ✅ above.")
        sys.exit(1)
```

**Do not proceed to transcription if this script exits with an error.** Fix the torch install first.

## Step 1 — Find the video
Use the path the user gave. If none, glob `*.mp4 *.mkv *.mov *.avi` in the CWD.

## Step 2 — Set up output directory
Create a `youtube-transcribe/` folder next to the video file:
```
<video_dir>/
├── myvideo.mp4
└── youtube-transcribe/
    ├── transcribe.py
    ├── transcript.txt
    ├── transcript.srt
    ├── transcript.json
    └── youtube_description.txt
```
All output files go into this folder. Never write anything next to the original video.

## Step 3 — Check for existing transcript
If `youtube-transcribe/transcript.json` already exists, skip to Step 5.

## Step 4 — Write and run transcribe.py
Write this script to `youtube-transcribe/transcribe.py`, then run it with:
`python transcribe.py "<path_to_video>"` from inside the `youtube-transcribe/` folder.

```python
import sys, os, json, ssl
ssl._create_default_https_context = ssl._create_unverified_context

os.environ["PATH"] = r"C:\ffmpeg\bin" + os.pathsep + os.environ.get("PATH", "")

import torch
import whisper

# Hard-fail if GPU is not available — never silently fall back to CPU
if not torch.cuda.is_available():
    sys.exit(
        "❌ CUDA not available. Run the Step 0 check script to diagnose.\n"
        "   Reinstall torch with CUDA support before transcribing."
    )

VIDEO = sys.argv[1] if len(sys.argv) > 1 else None
if not VIDEO or not os.path.isfile(VIDEO):
    sys.exit(f"Error: file not found: {VIDEO!r}")

# Output goes into the same folder as this script (youtube-transcribe/)
OUT = os.path.dirname(os.path.abspath(__file__))
print(f"Device : cuda — {torch.cuda.get_device_name(0)}")

model = whisper.load_model("medium", device="cuda")
result = model.transcribe(VIDEO, verbose=True, language="en", fp16=True)

# Release GPU memory after transcription
del model
torch.cuda.empty_cache()

def ts_chapter(s):
    """M:SS or H:MM:SS for YouTube chapter timestamps."""
    h, r = divmod(int(s), 3600)
    m, sec = divmod(r, 60)
    return f"{h}:{m:02d}:{sec:02d}" if h else f"{m}:{sec:02d}"

def ts_srt(s):
    """HH:MM:SS,mmm for SRT subtitle timestamps (required format)."""
    h, r = divmod(int(s), 3600)
    m, sec = divmod(r, 60)
    ms = round((s - int(s)) * 1000)
    return f"{h:02d}:{m:02d}:{sec:02d},{ms:03d}"

with open(os.path.join(OUT, "transcript.txt"), "w", encoding="utf-8") as f:
    f.write(result["text"].strip())

with open(os.path.join(OUT, "transcript.srt"), "w", encoding="utf-8") as f:
    for i, s in enumerate(result["segments"], 1):
        f.write(f"{i}\n{ts_srt(s['start'])} --> {ts_srt(s['end'])}\n{s['text'].strip()}\n\n")

with open(os.path.join(OUT, "transcript.json"), "w", encoding="utf-8") as f:
    json.dump(result["segments"], f, indent=2, ensure_ascii=False)

print("Done. Files written:")
for fname in ["transcript.txt", "transcript.srt", "transcript.json"]:
    path = os.path.join(OUT, fname)
    print(f"  {path}  ({os.path.getsize(path):,} bytes)")
```

**Runtime:** 2–8 min on GPU. Tell the user to wait.

## Step 5 — Read the transcript
- Read `youtube-transcribe/transcript.txt` for full text.
- For chapter timestamps, run from inside `youtube-transcribe/`:
```bash
python -c "
import json
segs = json.load(open('transcript.json', encoding='utf-8'))
for i, s in enumerate(segs):
    if i % 8 == 0:
        m, sec = divmod(int(s['start']), 60)
        print(f'[{m}:{sec:02d}] {s[\"text\"][:80]}')
"
```

## Step 6 — Generate the YouTube description
Write to `youtube-transcribe/youtube_description.txt` and print it in chat.

Use this exact structure:

```
<Hook: 1–2 punchy lines with specific details — model names, numbers, surprising result.
This is visible BEFORE "Show more" so make it count.>

<2–3 sentence summary paragraph with SEO keywords woven in naturally.>

━━━━━━━━━━━━━━━━━━━━━━━
⏱️ TIMESTAMPS
━━━━━━━━━━━━━━━━━━━━━━━
0:00  Intro
M:SS  <Descriptive chapter title>
...   (8–15 chapters total)

━━━━━━━━━━━━━━━━━━━━━━━
🔗 RESOURCES
━━━━━━━━━━━━━━━━━━━━━━━
<Tools, models, links mentioned in the video. Keep heading as RESOURCES.>

━━━━━━━━━━━━━━━━━━━━━━━
📌 WHAT YOU'LL LEARN
━━━━━━━━━━━━━━━━━━━━━━━
✅ <Takeaway>
✅ <Takeaway>
✅ <Takeaway>
✅ <Takeaway>
✅ <Takeaway>

━━━━━━━━━━━━━━━━━━━━━━━
🏷️ TAGS
━━━━━━━━━━━━━━━━━━━━━━━
#Tag1 #Tag2 #Tag3 ...
```

Rules:
- WHAT YOU'LL LEARN must use `✅` (not `-` or `•`)
- TAGS must be `#hashtag` format (not comma-separated)
- Keep all section headings exactly as shown
- 10–15 hashtags, mix broad and specific
- Chapter timestamps must use the `M:SS` or `H:MM:SS` format (no leading zero on minutes)
