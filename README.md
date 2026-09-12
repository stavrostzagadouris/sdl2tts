# sdl2tts

Push-to-talk voice input for Wayland. Hold a key, speak, release — the transcript is typed
into whatever field has focus.

Two pieces, about 430 lines total:

| | |
|---|---|
| `talkwave.c` | The recorder. libpulse capture on a dedicated thread, plus a small SDL2 level-meter bar pinned bottom-centre. Writes a finalized WAV on SIGTERM. Knows nothing about transcription. |
| `talk` | The orchestration. `start` / `stop` / `toggle`, PID state, a noise gate, the POST to your STT server, and `wtype` to inject the text. This is what your keybind calls. |

Speech-to-text is delegated to any server exposing an OpenAI-compatible
`/v1/audio/transcriptions` endpoint. There is no bundled model and no cloud dependency —
point it at a box on your LAN or tailnet running whatever you like (this was built against
[parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) on an M1 Mac).

## Requirements

- Wayland compositor (developed on Hyprland)
- PulseAudio or PipeWire's pulse shim
- `sdl2` (or `sdl2-compat`), `libpulse`, `wtype`, `curl`, `python3`
- `libnotify` for failure notifications; `tailscale` optional, only for failure diagnosis

On Arch:

```
sudo pacman -S --needed sdl2-compat libpulse wtype curl python libnotify
```

## Build

```
gcc -O2 -o talkwave talkwave.c \
    $(pkg-config --cflags --libs sdl2) \
    $(pkg-config --cflags --libs libpulse-simple)
install -Dm755 talkwave talk -t ~/.local/bin/
```

Note `libpulse-simple` is a **separate** pkg-config module from `libpulse`; `pa_simple_*`
will not link without it.

## Configure

```sh
export TALK_STT_URL=http://10.0.0.5:5000/v1/audio/transcriptions
export TALK_STT_PEER=my-stt-host   # optional, Tailscale peer name for diagnostics
export TALK_GATE=0.02              # optional, noise-gate RMS floor
```

Defaults to `http://127.0.0.1:5000/v1/audio/transcriptions`.

Your compositor probably does **not** put `~/.local/bin` on the PATH of the processes it
spawns, so export these somewhere the session actually reads, or use absolute paths in the
bind.

## Bind it

Hold-to-talk is a press bind and a release bind on the **same** key spec. Hyprland, flat
config:

```
bind  = SUPER, SPACE, exec, /home/you/.local/bin/talk start
bindr = SUPER, SPACE, exec, /home/you/.local/bin/talk stop

windowrulev2 = float, class:^(talkwave)$
windowrulev2 = pin,   class:^(talkwave)$
```

Use an absolute path: the compositor spawns children with a PATH that need not include
`~/.local/bin`.

SDL cannot place a borderless toplevel itself — Hyprland centres it and ignores the
requested x/y — so if you want the bar somewhere specific, move it with a rule. For a
1920-wide monitor, `(1920-280)/2 = 820` and 24px of clearance under the 48px bar:

```
windowrulev2 = move 820 100%-72, class:^(talkwave)$
```

`talk toggle` works too, if you prefer press-once-to-start, press-again-to-stop. Holding is
the better default: a toggle lets recorder state drift out of sync, and holding makes
"am I recording?" self-evident.

## Debugging

The compositor discards stderr, so everything goes to `~/.cache/talk.log`:

```
tail -f ~/.cache/talk.log
```

Each stop logs the measured peak RMS and loud-window count, so you can see whether the
noise gate ate your clip before blaming the server.

## Notes from building it

Every one of these failed *silently*, which is why they're worth writing down.

**PulseAudio's default record `fragsize` can be two seconds.** 64000 bytes at 16 kHz mono.
That delays first delivery and discards the in-flight partial fragment at stop — a 5-second
press yielded 4.0 seconds with the first word clipped. `talkwave` passes an explicit
`pa_buffer_attr` with a 40 ms `fragsize`. This was the actual cause; render-starvation and
ALSA-suspend were both wrong theories.

**Audio must never wait on pixels.** An earlier single-loop version interleaved
`pa_simple_read()` with rendering, so frame pacing throttled how fast audio drained. The
backlog queued server-side and was thrown away at SIGTERM, losing 45–70% of the recording.
Capture gets its own thread; the UI only peeks at a peak value under a mutex.

**Don't gate the first `SDL_RenderPresent` behind a blocking audio read** — the window never
maps, and there is no error.

**Writing a WAV header by seeking back over a stdio write stream garbles it.** Buffer the
PCM and write header-then-data once, at the end.

**`parec` may be a symlink to `pacat`** and never emit captured PCM to stdout. Don't build
on a `parec | ...` pipe; capture in-process.

**Mic input volume matters more than you'd think.** A laptop mic at 100% can sit ~40 dB over
its base gain and hard-clip everything — even room noise pins to full scale, which makes a
level gate impossible and makes the STT model invent words out of the mush. Check with
`pactl list sources | grep -m1 'Volume: front-left'`; around 40% was right here.

**A level gate cannot distinguish your voice from a TV.** Both are speech. The gate only
rejects near-silence — it needs 3+ consecutive 20 ms windows above `TALK_GATE` or it discards
the clip and notifies rather than spending a request. It fails *open*: a measurement error
must never silently drop a real recording.

**`wait` doesn't work across shells.** `talk stop` can't `wait` on the recorder, because
`talk start` was a different process — the recorder isn't its child, so `wait` returns
immediately and races `curl` against a WAV that's still being written. Poll `kill -0`
instead.
