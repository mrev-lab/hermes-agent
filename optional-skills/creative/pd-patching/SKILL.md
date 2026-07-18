---
name: pd-patching
description: "Build and control Pure Data (Pd) audio/DSP patches in real time by sending native canvas-manipulation messages over a socket. Zero external deps — stdlib socket only. Use for generative synths, live DSP graphs, MIDI/OSC workflows."
version: 1.0.0
author: mrev-lab
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [puredata, pd, dsp, audio, synthesis, dynamic-patching, netreceive, creative-coding, generative-audio, midi, osc]
    related_skills: [touchdesigner-mcp, songwriting-and-ai-music, comfyui]
---

# Pure Data Dynamic Patching

## When to use

Use when the user wants to **generate or control sound/DSP** in a running Pure Data
instance: build a synth, wire an effects chain, drive an LFO/filter, prototype a
generative audio patch, or set up MIDI/OSC processing — all by emitting text.

This is **Approach A (dynamic patching)**: Pd-vanilla's own canvas-editing
messages (`obj`, `connect`, `clear`, `dsp`) are sent to a `[netreceive]` socket.
No MCP server, no externals, no compiled bridge — just Python's `socket`.

## Architecture

```
Hermes Agent                    Pure Data (running receiver_stub.pd)
+----------------+  TCP :9999   +--------------------------------------+
| pd_client.py   | -----------> | [netreceive 9999]                    |
| pd_patch.py    | "pd-target_  |        |                             |
+----------------+  canvas obj  | [route pd-target_canvas pd]          |
                    100 50 osc~  |    |0        |1          |2          |
                    440"         | [send        [send pd]   [print      |
                                 |  pd-target_             pd-net-in]    |
                                 |  canvas]                (handshake)  |
                                 |    v                                 |
                                 | [pd target_canvas]  <- objects here  |
                                 +--------------------------------------+
```

A command is a **receiver-addressed FUDI message**: the first atom is the
receiver name, the rest is the message. Every open canvas exposes a receiver
`pd-<canvasname>`, and the global engine listens on `pd`.

> **Important — no leading `;`.** A leading semicolon routes to a receiver *only
> from a Pd message box*, **not** over `[netreceive]` (verified empirically: it
> just falls out the raw outlet). So the stub does the dispatch explicitly with
> `[route] → [send]`, and messages are sent **without** a leading `;`, e.g.
> `pd-target_canvas obj 100 100 osc~ 440` creates an `[osc~ 440]` inside the
> `target_canvas` subpatch. `pd_client.py` strips a stray leading `;` for you.

## Files

| File | What |
|------|------|
| `pd_client.py` | Socket bridge. `PureDataClient` class + CLI. TCP (default) or UDP. |
| `pd_patch.py` | `PdPatch` builder (auto layout, stable indices), DSP safety linter, `.pd` export (`to_pd_file`), WAV recorder (`to_dac(record=True)` / `add_recorder`). |
| `pd_examples.py` | Gallery of ready-to-run patches + CLI (`--send`, `--export file.pd`, `--wav out.wav --seconds N`). |
| `templates/receiver_stub.pd` | Base patch: `[netreceive 9999] → [route] → [send]` + `[pd target_canvas]`. Open this in Pd first. |

## Setup

### 0. Install the skill into Hermes

Bundled skills that ship in the repo are seeded automatically. If you are
developing this skill locally (editing it in a checkout), make Hermes see your
copy by placing it under the skills dir of your Hermes home — on macOS/Linux
`~/.hermes`, and the **Desktop app uses the same home**, so this covers CLI,
TUI, and Desktop at once:

```bash
SRC="$HOME/github/hermes-agent/optional-skills/creative/pd-patching"
DST="${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
rm -rf "$DST" && cp -R "$SRC" "$DST"        # a real copy — see the warning below
```

> **Use a real copy, not a symlink.** The CLI/agent follow symlinks, but the web
> **dashboard resolves skills with `rglob`, which does NOT follow directory
> symlinks** — a symlinked skill dir makes the dashboard return
> `404: Skill '…' not found`. Re-copy after edits (a copy does not live-reload).

Verify Hermes sees it:
```bash
hermes skills list | grep pd-patching          # -> pd-patching … enabled
```
Restart the Desktop app (or start a new session) if it doesn't appear yet.

### 1. Install and launch Pd

1. **Install Pd-vanilla** (once):
   - macOS: `brew install --cask pd`
   - Linux: `sudo apt install puredata`
   - Windows: download from https://puredata.info/downloads/pure-data
2. **Launch Pd and open the stub** so the socket is listening. On macOS the
   cask installs a `.app` and does **not** put `pd` on `PATH` — use the binary
   inside the bundle:
   ```bash
   SKILL="${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
   # macOS (cask): binary lives inside the .app bundle
   PD="$(ls -d /Applications/Pd-*.app 2>/dev/null | sort -V | tail -1)/Contents/Resources/bin/pd"
   # Linux: PD=pd
   "$PD" "$SKILL/templates/receiver_stub.pd" &          # with GUI (to hear audio)
   # "$PD" -nogui -stderr "$SKILL/templates/receiver_stub.pd" &   # headless
   ```
3. **Confirm the handshake.** A plain word has no matching receiver, so it falls
   out `[route]`'s last outlet to `[print pd-net-in]` — you'll see
   `pd-net-in: hello` in the Pd window (or on stderr with `-stderr`):
   ```bash
   python3 pd_client.py "hello"
   ```
   If the socket is closed you get `Connection refused` on stderr and a non-zero
   exit — that means Pd/the stub isn't running.

## Workflow

> **[CRITICAL] Run from the skill directory, via the shell/terminal tool.**
> `pd_client.py` / `pd_patch.py` / `pd_examples.py` are imported by name, so the
> commands below only work when the working directory is the skill folder. Do
> NOT run them in an isolated code sandbox that lacks these files — that fails
> with `ModuleNotFoundError: No module named 'pd_patch'`. Always `cd` there
> first (it works for CLI, TUI, and the Desktop dashboard):
>
> ```bash
> cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
> ```
>
> **Prefer file + argument commands over heredocs.** An agent's terminal tool
> often fails to deliver heredoc stdin, so `python3 - <<'PY' … PY` hangs waiting
> on input until the tool times out (~60 s). Use commands that take everything
> as **arguments** — `python3 pd_examples.py …` and `python3 pd_client.py "…"` —
> which never read stdin. Reach for heredoc/`-c` only in a real interactive
> shell.

### Preferred: run a builder script with arguments (no stdin, safety-checked)

Play a gallery patch, or an arbitrary chord of sines, in one stdin-free command:

```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_examples.py sine --send                       # single 440 Hz sine
python3 pd_examples.py sines --freqs 120 254 1201 --send  # mix these three sines
```

`pd_examples.py` builds the `PdPatch`, runs the safety linter, and sends it —
all from arguments, so nothing reads stdin.

To shape a graph in Python, put it in a real `.py` file and run that file (still
no stdin), e.g. `python3 my_patch.py`:

```python
from pd_patch import PdPatch
from pd_client import PureDataClient
p = PdPatch()               # targets pd-target_canvas
p.clear()
saw = p.obj("phasor~", 110)
lop = p.obj("lop~", 1000)          # lowpass
p.connect(saw, 0, lop, 0)
p.to_dac(lop, 0.1)                 # REQUIRED output stage:
                                   # hip~ 5 -> *~ 0.1 -> clip~ -1 1 -> dac~
p.dsp(True)
if not p.is_safe():
    raise SystemExit(p.safety_violations())
PureDataClient().send_batch(p.messages())
```

### Direct: raw messages via CLI

```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_client.py \
  "pd-target_canvas clear" \
  "pd-target_canvas obj 100 50 osc~ 440" \
  "pd-target_canvas obj 100 100 hip~ 5" \
  "pd-target_canvas obj 100 150 *~ 0.1" \
  "pd-target_canvas obj 100 200 clip~ -1 1" \
  "pd-target_canvas obj 100 250 dac~" \
  "pd-target_canvas connect 0 0 1 0" \
  "pd-target_canvas connect 1 0 2 0" \
  "pd-target_canvas connect 2 0 3 0" \
  "pd-target_canvas connect 3 0 4 0" \
  "pd-target_canvas connect 3 0 4 1" \
  "pd dsp 1"
```

## Example gallery (`pd_examples.py`)

Ready-to-run patches that make sound the instant DSP is on — no metro or click
needed. Great first thing to try, and a template for your own. All are
safety-linted (master gain ≤ 0.1). With Pd running the receiver stub:

```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_examples.py --list                # see them all
python3 pd_examples.py wobble_bass --send    # build, safety-check, and play
python3 pd_examples.py fm_bell | python3 pd_client.py -   # or pipe it
```

| Name | What you hear | Technique shown |
|------|---------------|-----------------|
| `sine` | Clean 440 Hz tone | the minimal `osc~ → *~ → dac~` chain |
| `sines --freqs …` | A chord of any sines | N `osc~` chain-summed (e.g. `--freqs 120 254 1201`) |
| `fm_bell` | Metallic, bell-like tone | FM — one `osc~` bends another's frequency |
| `wobble_bass` | Dubstep "wob" bass | `[vcf~]` band-pass swept by an LFO signal |
| `supersaw_drone` | Fat, shimmering drone | three detuned `phasor~` saws mixed |
| `random_blips` | Random blip melody | `[samphold~]` latches noise on a `phasor~` clock (generative, no metro) |
| `resonator` | Plucked/blown string drone | Karplus-Strong `[delwrite~]`/`[delread~]` feedback loop |

From Python you get the `PdPatch` back to tweak or extend:

```python
from pd_examples import wobble_bass
from pd_client import PureDataClient
p = wobble_bass(freq=41, rate=4)   # faster wobble, lower note
PureDataClient().send_batch(p.messages())
```

## Save a shareable artifact

The audio is otherwise ephemeral (it stops when Pd closes). Two ways to keep
something you can share:

**Export a standalone `.pd` patch** — openable in any Pd, no socket needed,
fully offline (Pd need not even be running):

```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_examples.py sines --freqs 120 254 1201 --export chord.pd
# in Python: PdPatch(...).to_pd_file("chord.pd")
```

**Record a WAV** of the exact audio you hear — needs Pd running **with audio on**
(a WAV can't be rendered under `-noaudio`, which has no DSP clock):

```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_examples.py wobble_bass --wav out.wav --seconds 8   # auto-stops
# without --seconds it records until you stop it:
python3 pd_client.py "pd-rec stop"
```

Recording taps the **post-limiter** signal, so the WAV is the same safe output
that reaches `[dac~]`. Under the hood: `to_dac(record=True)` (or `add_recorder()`
on an already-built patch) adds `[writesf~ 2]` fed by `[r pd-rec]`; the receiver
stub routes `pd-rec open <abs.wav>` / `pd-rec start` / `pd-rec stop`. The WAV's
sample rate follows Pd's audio settings.

## Usage from a Hermes session

How the agent turns a natural-language request into sound, **via the shell tool**.
The pattern is always: **cd to the skill dir → ensure the stub is up → build a
patch → `is_safe()` → `send_batch`.** Every snippet starts by entering the skill
directory so the imports resolve (see the [CRITICAL] note under Workflow):

```bash
SKILL="${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
cd "$SKILL"
```

**"Play a wobbly bassline in Pd"** — a canned example fits:
```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
nc -z 127.0.0.1 9999 || open -a "Pd-0.56-5" templates/receiver_stub.pd
python3 pd_examples.py wobble_bass --send
```

**"Play 120, 254 and 1201 Hz together"** — arbitrary chord of sines, one command:
```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_examples.py sines --freqs 120 254 1201 --send
```

**"Make a metallic bell at 300 Hz"** — a gallery builder with tweaked params.
Raw dynamic-patching messages also work as pure arguments (no stdin):
```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_client.py \
  "pd-target_canvas clear" \
  "pd-target_canvas obj 100 50 osc~ 513" \
  "pd-target_canvas obj 100 100 *~ 300" \
  "pd-target_canvas obj 100 150 +~ 300" \
  "pd-target_canvas obj 100 200 osc~" \
  "pd-target_canvas obj 100 250 hip~ 5" \
  "pd-target_canvas obj 100 300 *~ 0.1" \
  "pd-target_canvas obj 100 350 clip~ -1 1" \
  "pd-target_canvas obj 100 400 dac~" \
  "pd-target_canvas connect 0 0 1 0" \
  "pd-target_canvas connect 1 0 2 0" \
  "pd-target_canvas connect 2 0 3 0" \
  "pd-target_canvas connect 3 0 4 0" \
  "pd-target_canvas connect 4 0 5 0" \
  "pd-target_canvas connect 5 0 6 0" \
  "pd-target_canvas connect 6 0 7 0" \
  "pd-target_canvas connect 6 0 7 1" \
  "pd dsp 1"
```

**"That's too loud" / "stop it"**:
```bash
cd "${HERMES_HOME:-$HOME/.hermes}/skills/creative/pd-patching"
python3 pd_client.py "pd dsp 0"                 # stop DSP
# or rebuild with a lower master [*~] (<= 0.1) and resend
```

**Agent guidance**
- **Run via the shell tool, from the skill directory** (`cd` first — see the
  [CRITICAL] note under Workflow). Do not use an isolated code interpreter that
  lacks `pd_patch.py` — it raises `ModuleNotFoundError`.
- **Use argument-based commands, never heredoc stdin.** `python3 pd_examples.py
  sines --freqs …` / `python3 pd_client.py "…"` take everything as arguments; a
  `python3 - <<'PY'` heredoc hangs the terminal tool on stdin until it times out.
- **Discovery:** the `description`/`tags` load this skill for "make a synth / DSP / generative audio in Pd" style requests.
- **Choose the cheapest path:** canned sound → `pd_examples`; a tweak → call a gallery builder with params; novel routing → build with `PdPatch`.
- **Always `is_safe()` before sending.** Never hand-wire a signal object into `[dac~]`; insert a `[*~]` ≤ 0.1. If a request implies danger ("blast it", direct-to-dac), auto-insert the gain stage instead of complying literally.
- **Prerequisite:** Pd must be running the receiver stub. If `nc -z 127.0.0.1 9999` fails, launch it; a `Connection refused` from `pd_client.py` means the same.

## Model notes (results vary by model)

How far a model can drive this skill depends on the model, because there are two
layers:

- **Canned + argument commands** — `python3 pd_examples.py <name> --send`,
  `python3 pd_examples.py sines --freqs …`, and raw `python3 pd_client.py "…"`.
  These are single, stdin-free commands that need no graph design, so **even a
  small local model runs them reliably** (a local Gemma ran `sines --freqs` in
  under a second).
- **Composing a novel patch** with `PdPatch` (arbitrary objects + wiring) needs
  a **capable model**. In head-to-head testing on the same prompt, a frontier
  model produced a correct ~20-object patch (one minor Pd-inlet gotcha, then
  clean); a small local model produced non-runnable code — wrong imports,
  invented objects (`scale~`), no `connect` calls, no output stage. The safety
  linter blocks *unsafe* output but cannot repair an *incorrect* graph.

Guidance: on a weak/local model, prefer the gallery and `sines --freqs`. For
open-ended "design a … synth" requests, use a strong model — that is where the
generative, agent-composes-the-DSP-graph capability actually pays off.

## Message reference (Pd dynamic-patching syntax)

No leading `;`. The client appends the trailing `;` terminator itself.

| Action | Message |
|--------|---------|
| Clear canvas | `pd-target_canvas clear` |
| Create object | `pd-target_canvas obj {x} {y} {name} {args...}` |
| Create message box | `pd-target_canvas msg {x} {y} {text}` |
| Wire | `pd-target_canvas connect {srcIdx} {outlet} {dstIdx} {inlet}` |
| DSP on/off | `pd dsp 1` / `pd dsp 0` |

To target a different canvas, add its receiver name (`pd-<name>`) to the
`[route]` object in `receiver_stub.pd` and address messages to it.

- Object **indices are 0-based** and assigned in creation order (after each
  `clear`). `pd_patch.py` tracks them for you.
- Coordinates: (0,0) is top-left. Rows ~50px apart, columns ~140px apart.

## [CRITICAL] Audio DSP safety rules

The output must reach `[dac~]` through the **protective master chain**:

```
source → [hip~ 5] → [*~ 0.1] → [clip~ -1 1] → [dac~]
```

`PdPatch.to_dac(source, gain)` builds exactly this, and
`PdPatch.safety_violations()` **enforces** all three guards (check `is_safe()`
before you `send`):

1. **Limiter at the DAC** — `[dac~]` must be fed by `[clip~ -1 1]`, a brick-wall
   limiter, so transient overshoots can't hit the DAC as full-scale bursts.
   Never wire a raw signal object (`osc~`/`vcf~`/`*~` …) straight to `[dac~]`.
2. **Bounded master gain** — the `[*~]` feeding that `[clip~]` must be ≤ 0.1
   (`MAX_MASTER_GAIN`). Protects ears/gear from feedback and loud mixes.
3. **DC / subsonic block** — a `[hip~ 5]` must sit upstream of `[dac~]`; DC
   offset and subsonics waste headroom and over-excurse woofers.

Also ramp parameter changes with `[line~]` / smooth with `[lop~]` to avoid
clicks and pops. **Always end a patch with `to_dac(...)`** rather than wiring
`[dac~]` by hand.

## Pitfalls

- **Adding/multiplying two *signals*? Use the no-argument form.** `[+~]`,
  `[*~]`, `[-~]` with a creation argument (`[+~ 110]`) turn their **right inlet
  into a control (float) inlet** — a signal wired there is rejected with
  `error: audio signal outlet connected to nonsignal inlet (ignored)` when DSP
  starts, and the value silently doesn't apply. To sum two signals use `[+~]`
  with no arg (both inlets are signals); keep `[+~ base]` only for adding a fixed
  scalar to a signal on the **left** inlet. (`[vcf~]`'s centre-freq inlet, by
  contrast, *is* a signal inlet.)
- **`osc~` frequency by signal goes in the left inlet (inlet 0)**; the right
  inlet is phase. `[vcf~]`: inlet 0 audio, inlet 1 centre freq (signal), inlet 2
  Q (float).
- **The safety linter checks output *safety*, not signal-flow *correctness*.**
  `is_safe()` guarantees the `hip~/[*~ ≤0.1]/clip~` output chain — it will not
  catch a signal wired to a control inlet or a mis-indexed `connect`. Watch Pd's
  console/stderr for `nonsignal` / `couldn't create` after `pd dsp 1`.

## Verification

1. **Before sending:** lint the graph in-process — `PdPatch.is_safe()` /
   `safety_violations()` catch a missing `[clip~]` limiter, a master gain above
   `0.1`, or a missing `[hip~]` DC blocker. `build_simple_synth()` is a
   known-safe reference (ends in `to_dac`).
2. **Socket handshake:** `python3 pd_client.py "hello"` — a plain word has no
   matching receiver, so it falls out `[route]`'s last outlet to
   `[print pd-net-in]` (visible in the Pd window / stderr). No output plus a
   `Connection refused` on stderr means Pd or the stub isn't running.
3. **Canvas edits landed:** after sending a build, an intentionally invalid
   object name (e.g. `pd-target_canvas obj 10 10 nope_xyz`) makes Pd log
   `nope_xyz ... couldn't create` — proof the message reached the canvas editor.
   Valid objects create silently.

## Security notes

- Communication is **localhost only** by default (`127.0.0.1:9999`). Pd's
  `[netreceive]` has no authentication — any local process can send canvas
  commands, so don't expose the port beyond loopback.
- Dynamic patching can instantiate any Pd object, including ones that read/write
  files or run shell (`[shell]`, `[pdlua]`); only send patches you trust.
