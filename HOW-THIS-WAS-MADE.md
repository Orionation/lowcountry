# Lowcountry — How This Was Made

A short overview of how this album came together, for anyone curious about the approach.

**Listen:** https://orionation.github.io/lowcountry/
**Runtime:** 33:02, 15 tracks

---

## The short version

A country/hip-hop album about Charleston, SC. Lyrics written collaboratively with an AI
assistant, music generated with the **ElevenLabs Music API**, with an optional cloned voice
available for vocals. Tracks were assembled and mastered locally with **ffmpeg**, cover art
made with an image model, and the whole thing published free on **GitHub Pages**.

---

## The stack, at a high level

| Layer | Tool |
|---|---|
| Music generation | ElevenLabs Music API |
| Section control | ElevenLabs **composition plans** (per-section styles — the key technique) |
| Voice clone | ElevenLabs Instant Voice Cloning |
| Voice on vocals | ElevenLabs speech-to-speech |
| Stem separation | **demucs** (open source) |
| Audio editing/mastering | **ffmpeg** |
| Cover art | an image-generation model |
| Hosting | GitHub Pages |

One useful gotcha: music generation needs the `music_generation` permission enabled on your
ElevenLabs API key — it's **not** on by default, and a key without it returns a 401. You toggle
it in the ElevenLabs API-key settings.

---

## The one technique worth knowing: composition plans

A single text prompt casts *one* set of voices for a whole song. That's why early attempts at
"a country singer, then a guest rapper" kept failing — the rapper got absorbed into the country
vocal. The fix is a **composition plan**: instead of one prompt, you give a list of sections,
each with its own positive/negative styles (e.g. one chunk "male country lead, southern drawl,"
the next "hip-hop verse, no country vocal"). That per-section control is what makes genre
crossovers actually work.

---

## Want the full details?

The complete build — the exact workflow, prompt/plan structures, and the audio-surgery steps —
is best walked through directly. Jonathan set most of this up and is happy to share what's
actually useful for a similar project. Reach out to him and he'll point you in the right direction.
