---
name: manim-skill
description: Create voiced-over educational videos using Manim and ElevenLabs TTS. Use this skill when the user asks to create a video, make an animation, produce an explainer, or generate visual content about a topic. Also trigger when the user mentions manim, voiceover videos, or educational animations. If the user says something like "make a video about X" or "create a 30s explainer on Y", this is the skill to use.
---

# Manim Video Creator

Create voiced-over educational videos using Manim + ElevenLabs TTS.

## Setup

```bash
pip install manim "manim-voiceover-plus[elevenlabs]"
brew install sox  # macOS (apt install sox for Linux)
```

Requires `ELEVEN_API_KEY` env var.

## Core Pattern

The library is `manim-voiceover-plus` (not `manim-voiceover`). These are the exact imports:

```python
from manim import *
from manim_voiceover_plus import VoiceoverScene
from manim_voiceover_plus.services.elevenlabs import ElevenLabsService

class VideoName(VoiceoverScene):
    def construct(self):
        self.set_speech_service(
            ElevenLabsService(
                voice_id="JBFqnCBsd6RMkjVDRZzb",  # George
                model="eleven_turbo_v2_5",
            )
        )

        with self.voiceover(text="Narration text here.") as tracker:
            self.play(Create(shape), run_time=tracker.duration)
```

## Gotchas

- **Use `voice_id`, never `voice_name`** — name lookup is unreliable
- **`run_time=tracker.duration`** syncs animation length to audio. Without it, animations and voiceover drift.
- **Bookmarks** for word-level timing: `<bookmark mark='A'/>` in text, then `self.wait_until_bookmark("A")`
- **Short sentences** — TTS quality degrades with long narration blocks
- **JSONDecodeError** — delete `media/voiceover/` cache folder and re-render
- Audio is cached via SHA-256 hash in `media/voiceover/`. Same text + voice = reuses cache.

## Known Voice IDs

IDs can vary by account. Fetch with:
```python
from elevenlabs import ElevenLabs
client = ElevenLabs()
for v in client.voices.get_all().voices:
    print(f'{v.name}: {v.voice_id}')
```

Defaults:
- `JBFqnCBsd6RMkjVDRZzb` — George (warm storyteller)
- `EXAVITQu4vr4xnSDxMaL` — Sarah (mature, confident)
- `Xb7hH8MSUJpSbSDYk0k2` — Alice (clear educator)
- `pFZP5JQG7iQjIQuC4Bku` — Lily (expressive)

## Verification

After rendering, extract frames and verify visually:
```bash
VIDEO="media/videos/video/480p15/ClassName.mp4"
mkdir -p media/verification_frames
DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$VIDEO")
for i in $(seq 0 7); do
  TS=$(echo "$DURATION / 8 * $i" | bc -l)
  ffmpeg -y -ss "$TS" -i "$VIDEO" -vframes 1 -q:v 2 "media/verification_frames/frame_$(printf '%02d' $i).png" 2>/dev/null
done
```

Then use a subagent to read each frame and compare against expected visuals.
