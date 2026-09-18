# Lowcountry — Production Brief for Orion

Everything needed to answer questions about this album and to make more of it.
Audience for answers: curious friends, not engineers. Lead with the short version.

---

## 1. The 30-second answer

Jonathan made a 16-track country/hip-hop album about Charleston, SC called **Lowcountry**.
Lyrics written collaboratively with Claude, music generated via the ElevenLabs Music API,
his own cloned voice available for vocals, assembled with ffmpeg on his MacBook Air,
published free on GitHub Pages.

**Listen:** https://orionation.github.io/lowcountry/
**Artist name:** Orion & The Holy City Band (placeholder)
**Runtime:** 33:02, 16 tracks

---

## 2. Stack

| Layer | Tool |
|---|---|
| Music generation | ElevenLabs Music API, `music_v2` — `POST /v1/music` |
| Section control | ElevenLabs **composition plans** — per-section styles, the key unlock |
| Voice clone | ElevenLabs Instant Voice Cloning — `POST /v1/voices/add`, from a 1:53 phone memo |
| Voice on vocals | `POST /v1/speech-to-speech/{voice_id}`, model `eleven_multilingual_sts_v2` |
| Stem separation | **demucs** (Meta, open source) — `demucs --two-stems=vocals` |
| Audio surgery | **ffmpeg** — mastering, splicing, silence removal, onset detection |
| Cover art | OpenAI `gpt-image-1` |
| Hosting | GitHub Pages via `gh` CLI — repo `Orionation/lowcountry` |
| Machine access | MCP server on the MacBook Air over Tailscale |

ElevenLabs account: **Creator tier**, instant + professional cloning enabled.
Music generation needs the `music_generation` permission on the API key — NOT on by default;
a key missing it returns 401 `missing_permissions`. Toggle at elevenlabs.io/app/settings/api-keys.

Key: `~/.openclaw/openclaw.json` → `talk.providers.elevenlabs.apiKey`
Jonathan's cloned voice ID: `~/Downloads/voice_id.txt`

---

## 3. Composition plans — the important part

A single text prompt casts ONE set of voices for a whole song. That's why early attempts at
"country singer, then a guest rapper" kept failing — the rapper got absorbed into the country vocal.

A plan is a list of chunks, each with its own styles:

```json
{"composition_plan": {"chunks": [
  {"text": "", "duration_ms": 7000,
   "positive_styles": ["acoustic guitar intro","no vocals"],
   "negative_styles": ["drums","vocals"],
   "context_adherence": "high",
   "conditioning_ref": null, "condition_strength": null},
  {"text": "[Verse]\n...", "duration_ms": 21000,
   "positive_styles": ["male country lead","thick southern drawl"],
   "negative_styles": ["rapping","female vocal"]}
]}, "model_id": "music_v2", "seed": 51101}
```

Gotchas learned the hard way:

- `seed` works with `composition_plan` but is **rejected with a plain `prompt`** (422).
- `seed` does **NOT** make output deterministic. Same plan + same seed twice = different songs.
  Verified by md5. There is no way to reproduce a melody you liked.
- `duration_ms` is a hard budget. If a lyric overruns, the next chunk starts **on top of it** and
  you get overlapping vocals. A 4-line verse needs ~21–22s at 130–146 BPM. Give it room.
- Plans leave **silent seams** between chunks. Always strip them (see §5).
- Negative styles work better than positive ones. Naming what you don't want — "no fiddle",
  "no 6/8", "no gospel choir", "no sea shanty" — fixed more problems than any positive styling.

---

## 4. Available, not yet used

- **Inpainting** — regenerate one section of an existing track, keep the rest. The real fix for
  "great melody, one bad line." We re-rolled whole songs instead and lost melodies Jonathan liked.
- **Audio reference** — ~30s clip guides style/production/tempo, not melody.
- **Finetunes** — train a personalized model on tracks you own. 16 tracks may be enough.

---

## 5. ffmpeg recipes actually used

Strip chunk seams + master:
```bash
ffmpeg -y -i in.mp3 -af "silenceremove=start_periods=1:start_threshold=-55dB:start_silence=0:\
stop_periods=-1:stop_threshold=-55dB:stop_duration=0.2,loudnorm=I=-10:TP=-1.0:LRA=11" \
  -c:a libmp3lame -b:a 192k out.mp3
```
Loudness targets: **-9** party tracks, **-10** mid, **-11** quiet (Waitin' On The Sun).

Find where the vocal starts — drums spike, vocals sustain, so look for several consecutive
windows above threshold in the mid band, not single peaks:
```bash
ffmpeg -ss $t -t 0.2 -i x.mp3 -af "highpass=f=400,lowpass=f=3500,volumedetect" -f null -
```
Used to land the first vocal at exactly 5.0s by trimming the head.

Splice a separately-generated rap verse into an instrumental break:
```bash
ffmpeg -y -i a.wav -i b.wav -filter_complex "[0][1]acrossfade=d=0.3:c1=tri:c2=tri" out.wav
```

Full-album video for YouTube — keep keyframes far apart or the file balloons (1fps with
`-g 250` gave 38MB for 20 min; `-g 4` gave 321MB):
```bash
ffmpeg -y -loop 1 -framerate 1 -i cover.png -i album.mp3 \
  -vf "scale=1920:1920:force_original_aspect_ratio=increase,crop=1920:1080,format=yuv420p" \
  -c:v libx264 -tune stillimage -preset medium -crf 26 -r 1 -g 250 -keyint_min 250 \
  -sc_threshold 0 -c:a aac -b:a 192k -shortest -movflags +faststart out.mp4
```

---

## 6. Voice cloning + putting Jonathan on a track

1. Clone: `POST /v1/voices/add`, multipart field `files`, 1–5 min of clean speech.
   A 1:53 iPhone voice memo was enough to be recognizable. More audio = better.
2. Split: `demucs --two-stems=vocals -o /tmp/vswap track.mp3` → `vocals.wav` + `no_vocals.wav`
3. Convert: `POST /v1/speech-to-speech/{voice_id}` with `vocals.wav`
4. Remix: `amix` the new vocal over `no_vocals.wav`, then loudnorm.

Script: `~/Downloads/voiceclone.py` — `clone` / `swap <track> <out>`

Honest limits: speech-to-speech inherits pitch from the source vocal, so singing stays in tune;
what changes is timbre, and it can read as uneven. demucs leaves artifacts where the vocal sat.
Jonathan's verdict on attempt one: "just ok." Untried levers: `htdemucs_ft` (slower, cleaner),
blending cloned vocal under the original ~70/30, stability/similarity params on the STS call,
real pitch correction in Logic (installed on the Mac).

---

## 7. Songwriting rules that actually worked

Jonathan's ear is good and specific. These are his corrections, generalized:

- **Monorhyme the chorus.** All four lines on the same sound. AABB read as "doesn't rhyme" to him
  more than once. Same trick works for a rap verse — eight lines, one rhyme.
- **Title in the first line and the last line** of the chorus.
- **Adjacent lines must rhyme.** ABAB verses got flagged as dead spots. Reorder to AABB.
- **Watch section seams** — don't switch rhyme families abruptly between the end of one section
  and the start of the next; it reads as unrhymed.
- **Real detail beats research.** The best song on the record (Waitin' On The Sun) came from
  Jonathan texting what he saw from a dock at sunrise: pluff mud snapping, glass water, an eagle
  and an otter. Research gave correct facts; the dock gave a song.
- **Local shibboleths land.** Legare = la-GREE, Huger = YOU-gee, Vanderhorst = Van-dross,
  Hasell = HAY-zul. Pass these to the model as explicit pronunciation instructions.
- **Get factual details right** — an eagle soars, it doesn't perch on a piling. He caught that.
- **Lyrics first, then generate.** Locking words avoids losing melodies to revision.
- **Roll 3 takes of locked lyrics and let him pick.** ~2 min a batch, and the only defense
  against non-reproducibility.

---

## 8. Album facts

16 tracks, 33:02: Charleston Brunch / Happy Hour / Downtown Streets / Dorchester Road /
Shrimp And Grits (And BBQ Too) / Easy Life / Mind The Locals / Capers Island / Boat Party /
Intracoastal / Mermaids And Dolphins / Caroline / Fishing With My Love / Redfish /
Waitin' On The Sun / Shem Creek Dreaming.

Island run sits together mid-record; the last three are all water at rest.

References are real and checkable: Husk and Poogan's both on Queen, 82 Queen's she-crab, SNOB at
192 East Bay, FIG on Meeting, Hall's on King, Henry's on the Market, Four Corners of Law,
Philadelphia Alley (formerly Cow Alley), Chalmers cobblestone, Queen St was Dock St, Calhoun was
Boundary St, South of Broad = "S.O.B.", Rec Room and Burns Alley on Upper King, Marion Square.
Boneyard Beach on Capers, Dewees, Kiawah, Folly, Sullivan's, Shem Creek shrimp boats, Red's,
Shrimp Boat Lane, Copahee Sound, spartina, pluff mud, strand feeding (dolphins — nearly unique to
the SC Lowcountry), Marsh Hen Mill, Jimmy Red corn, Mason the 15-ft sweet tea jug in Summerville,
Colonial Dorchester's oyster-shell tabby walls.

Personal: Shelly (wife, from Vicksburg MS), Ziggy (dog), son and daughter, 20 years married.
"Caroline" is a wistful one-perfect-day memory song, deliberately written as someone never seen
again — not an affair. Don't imply otherwise if asked.

---

## 9. Files and locations

- Album masters: `~/Music/Lowcountry-16/` (tagged MP3s, cover.png, lyrics.txt)
- Earlier 10-track cut: `~/Music/Lowcountry/`
- Shelly-specific songs, not on the album: `~/Downloads/shelly-*.mp3`
- Site repo checkout: `/tmp/lc-repo` — **/tmp gets wiped between sessions.** Re-clone with
  `gh repo clone Orionation/lowcountry`. A push once silently failed because `.git` had vanished.
- Generation scripts: `/tmp/songs/*.py` — also ephemeral, worth moving somewhere permanent
- Voice clone script: `~/Downloads/voiceclone.py`
- Full album video + `youtube-description.txt` (timestamps auto-convert to YouTube chapters):
  `~/Music/Lowcountry/`

Publishing: edit files in a clone, `git push`, Pages redeploys in ~60s. Don't forget
`tracks.json` — the player reads it, and a stale one silently shows the old tracklist.

---

## 10. Rights and money — say this carefully

- ElevenLabs grants broad commercial use of generated music on paid plans. Film/TV/large studio
  game rights need Enterprise.
- In the US, **purely AI-generated output isn't copyrightable.** Human authorship anchors a claim —
  here that's Jonathan's lyrical direction and his recorded voice. Strategy follows: the more of
  him in it, the more there is to own.
- Not legal advice. A music attorney is worth an hour before anything commercial.
- Distribution: DistroKid (~$23/yr) or Bandcamp. Both want better than 192kbps — re-render at 320
  or WAV first.

---

## 11. Open items

- Waitin' On The Sun on the album is the **v1 melody** Jonathan preferred; it still says
  "eagle on the pilin'" and lacks the heron/egret/osprey/pelican verse. Three corrected-lyric
  takes exist (seeds 7734, 20988, 41562), never chosen.
- Redfish (Ol' Copper) — he said the story version reads too close to "Ole Red." Two alternative
  angles were pitched, not yet written.
- Guest rapper only appears on tracks 1, 2 and the Shelly songs; the 8 batch-generated tracks are
  single generations without the feature.
- Vocal consistency across tracks hasn't been checked end to end.
- Artist name is a placeholder.
- litterbox.catbox.moe (temp links all session) was returning 500s; those links expire in 72h
  regardless. GitHub Pages is the durable home.
