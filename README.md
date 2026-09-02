# scribe

Local lecture capture. Records a class, transcribes it as it happens, and
writes the notes into an Obsidian vault. Nothing leaves the machine.

Built 2026-09-02, during a lecture, using the prototype to record the lecture
it was being built in.

## What's here

| | |
|---|---|
| `bin/scribe` | the terminal app — Start / Stop / Mark / Export as buttons |
| `bin/lecture` | the headless CLI it grew out of (`lecture`, `lecture stop`) |
| `bin/lecture-live` | the first live-transcript prototype, kept for reference |

## scribe

```
scribe                 open the app
scribe --autostart     begin recording as soon as whisper is loaded
scribe --device NAME   force a specific PulseAudio input
```

Buttons (and keys): **● Start** `s` · **■ Stop** `s` · **★ Mark** `m` ·
**⭳ Export** `e` · Quit `q`.

- Picks the class from a hard-coded schedule (`SCHEDULE` in `bin/scribe`) and
  the mic from `~/lectures/.config.json`.
- One ffmpeg process feeds both the opus archive (~11 MB/hour) and the live
  PCM stream, so the mic is opened once.
- **Suspend is blocked while recording.** A closed lid used to end the capture
  silently — logind takes the machine down, PulseAudio goes with it, ffmpeg
  exits. scribe now holds a `sleep:idle:handle-lid-switch` inhibitor for the
  length of the recording. The lock is held by a `cat` reading a pipe scribe
  owns, so it dies with the app even under SIGKILL — an inhibitor that outlives
  the process would leave the machine unable to sleep at all.
- **If the input dies anyway, it reconnects.** The capture loop notices ffmpeg
  exiting, says so loudly in the UI, and reopens into `…part2.ogg`,
  `…part3.ogg` and so on. Export stitches every segment into one timeline with
  the offsets applied, and lists them all in the note's frontmatter.
- Live transcription cuts on natural pauses (energy VAD with a running noise
  floor), so chunks land between sentences. Roughly 10-20 s behind the speaker.
- **Mark** drops a `★` at the current moment; marks come out in the exported
  note as `> [!important]` callouts at the right timestamp.
- **Export** re-transcribes the *whole* recording at full quality — better than
  the live slices, which only see 20 s of context — and writes
  `<vault>/YYYY-MM-DD COURSE.md`.

## lecture (CLI)

```
lecture                  start recording, course auto-detected
lecture stop             stop, transcribe, write the vault note
lecture transcribe FILE [COURSE]
lecture mic list | <name>
```

## Configuration

Everything personal — your class schedule, where the notes get written, which
mic — lives in `~/.config/scribe/config.json`, not in the source. Copy
`config.example.json` to start:

```bash
mkdir -p ~/.config/scribe
cp config.example.json ~/.config/scribe/config.json
```

| key | what it does |
|---|---|
| `audio_dir` | where recordings land (default `~/lectures`) |
| `vault` | folder the exported notes are written to |
| `device` | PulseAudio source; `scribe`'s mic picker writes it back here |
| `courses` | the class list in the dropdown |
| `schedule` | days/times per class, used to pre-select the right one |
| `grace_before_min` / `grace_after_min` | how far either side of a class still counts as that class |

`days` are `0`=Monday through `6`=Sunday. With no config file at all, both
tools still work — they just call everything "Lecture" and write to
`~/vault/Lectures`. Override the config path with `$SCRIBE_CONFIG`.

## Requirements

- `ffmpeg` with `libopus`, PulseAudio/PipeWire, `pactl`
- `faster-whisper`, `numpy`, `textual`
- For GPU: `nvidia-cublas-cu12` and `nvidia-cudnn-cu12` from pip. ctranslate2
  finds them only via `LD_LIBRARY_PATH`, which the dynamic loader reads at exec
  time — both entry points re-exec themselves once to set it.

Model is `mobiuslabsgmbh/faster-whisper-large-v3-turbo`. On an RTX 5060 Laptop
it runs ~30-40x realtime, so a 75-minute lecture exports in about two minutes.

## Gotchas found the hard way

- Under Textual, `sys.stderr.fileno()` is `-1`. Anything that lazily spawns
  multiprocessing's resource tracker (whisper's load path does) dies with
  `ValueError: bad value(s) in fds_to_keep`. `scribe` starts the tracker up
  front, while stderr is still a real fd.
- `-flush_packets 1` on the archive output: without it ffmpeg flushes the ogg in
  256 KB blocks, so a crash can cost ~90 seconds of audio.

## Still to do before shipping

- No pause/resume, no session browser, no way to export an older recording from
  inside the app.
- A gap in a recording (input lost for 40 s) is stitched out, not marked. The
  exported timeline is shorter than the wall clock and nothing says where.
- Speaker names, slide/screen capture, and a summariser pass are all obvious
  next steps and none of them exist.
- Recording a class needs the instructor's okay. The app doesn't say so anywhere.
- The name collides with a lot of prior art — ElevenLabs' ASR model, scribehow,
  Commure/Athelas, Express Scribe. Fine for a local tool, a problem for anything
  with a logo.
