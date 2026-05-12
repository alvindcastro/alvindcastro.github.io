---
title: "Script to Video"
date: 2026-05-12T00:00:00Z
---

# script-to-video: A Plain-Text to Narrated MP4 Pipeline

The goal was simple: write a script, run one command, get a video. No timeline editor, no keyframe wrangling, no manually syncing audio to slides. Just a text file and a terminal.

The stack is TypeScript, Remotion for rendering, and ElevenLabs for text-to-speech. The pipeline runs five steps in sequence, each producing an artifact the next step consumes.

## The Pipeline

`build-spec` parses `prompts/script.txt` into a structured scene graph written to `src/data/video.json`. It estimates scene duration from narration length at roughly 0.065 seconds per character, which is close enough for layout previewing in Remotion Studio but will drift from real audio. That drift is the problem the next steps exist to solve.

`voice` sends the narration text to ElevenLabs and writes two files: `public/audio/voiceover.mp3` and `voiceover-alignment.json`, a character-level alignment file that maps every character in the narration to its position in the audio stream.

`retime` reads the alignment file, walks through each scene's narration text, finds where it first appears in the audio stream, and rewrites `video.json` with accurate `startSec` and `durationSec` values for every scene. The original spec is backed up before being overwritten, and if you regenerate audio, you run `retime` again; it reads the current alignment file and rewrites everything from scratch, so nothing drifts.

`captions` chunks the alignment data into word groups and writes `src/data/captions.json`, which Remotion uses to drive the word-synced caption overlay at the bottom of the frame.

`render` calls the Remotion CLI with the correct frame count derived from the total audio duration and produces `out/video.mp4`.

## Script Format

The format asks nothing complicated of the writer. A `[Section · Headline]` header opens a new scene, lines that follow become narration joined into a single block, and lines starting with `- ` or `* ` become animated bullet points that slide in staggered from the left. Blank lines are ignored.

```
[Overview · How it works]
The pipeline converts a plain-text script into a narrated MP4 in five steps.
- Parse scenes from script
- Generate voiceover via ElevenLabs
- Sync timing to actual audio
- Build word-synced captions
- Render final MP4
```

## The Timing Problem in Detail

Character-count estimation assumes a fixed speaking rate and ignores punctuation pauses, proper noun pronunciation, and natural cadence variation. For a short script the drift is small; for a longer one it accumulates until scenes slide out of sync with the audio.

The alignment file ElevenLabs returns solves this cleanly. Each entry maps a character offset in the narration string to a timestamp in the audio, and `retime` uses that mapping to anchor each scene to where its narration actually starts. The final scene gets extended to cover the audio tail plus a hold buffer so the video does not cut off abruptly. The timing is always derived from the real audio, not an estimate, and the only cost is remembering to run `retime` again after regenerating audio.

## Remotion Studio as a Development Loop

Because `build-spec` runs without spending ElevenLabs credits, you can iterate on scene structure and layout quickly. Run `build-spec`, open Remotion Studio with `npm run dev`, scrub through scenes, adjust narration or bullets, and repeat until the layout is right, then run `voice` to generate audio. This keeps the feedback loop fast and the credit spend deliberate.

## Repo

[github.com/alvindcastro/script-to-video](https://github.com/alvindcastro/script-to-video)
