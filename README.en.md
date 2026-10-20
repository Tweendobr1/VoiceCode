# VoiceCode

**Hold F8. Speak. Hear the answer.**

[Русский](README.md) · **English**

A voice-first mini-copy of [opencode](https://opencode.ai) for Windows: hold a key,
speak Russian, release — and the answer comes back as speech. No chat window, no
history on screen, just voice and an animated interface in the opencode style.

![VoiceCode interface](docs/preview.png)

## What it does

- **Push-to-talk.** Hold `Space` in the window or `F8` anywhere, talk, release —
  everything else is automatic.
- **Local speech recognition.** `faster-whisper small` in int8 on the CPU, the
  model ships in `models/faster-whisper-small`; recognition needs no internet.
- **Spoken answers.** `edge-tts` (voice `ru-RU-DmitryNeural`) with streaming,
  sentence-by-sentence playback; when Microsoft's endpoint is unreachable it falls
  back to the built-in Windows voice (`Microsoft Irina Desktop`) without hiccups.
- **System commands.** Opens apps and websites, searches the web, takes
  screenshots, controls volume, locks the PC, minimizes and closes windows, tells
  the time and date — all locally, in milliseconds.
- **Everything else goes to opencode.** Questions, code and explanations are sent
  to `opencode-cli run`, streamed back and spoken as they arrive.
- **No text chat.** Only a status line, one line of recognized text and one line
  of answer — they fade away by themselves.

## How it works

```
F8 / Space / button
        │  (microphone stream is always open)
        ▼
faster-whisper small → text
        │
        ├─ local command router → action + short phrase
        │
        └─ opencode-cli run --format json → text stream
                                              │
                                              ▼
                                   edge-tts / SAPI, sentence by sentence
```

Key decisions made for speed:

- the PortAudio input stream stays open, so recording starts instantly;
- the Whisper model is warmed up at startup, the first answer never waits for it;
- the answer text is split into sentences and voiced as it arrives;
- local commands never touch the network;
- the opencode provider is chosen once in `config.json`, with no renegotiation.

Measured on an 8-core CPU: recognition ≈ 0.6–0.7× the phrase duration
(4.4 s of speech → ≈ 2.9 s), opencode ≈ 2.0–2.5 s on `deepseek-v4.1-flash-free`,
speech starts in ≈ 0.2 s (SAPI) or ≈ 0.5–1 s (edge-tts).

## Requirements

- Windows 10/11 (64-bit).
- A microphone and speakers/headphones.
- To build from source — Python 3.10–3.12 (tested on 3.12).

The `VoiceCode-Setup.exe` installer needs no Python and installs nothing extra:
the interpreter, the Whisper model and `opencode-cli.exe` v2.1.1 are already
inside. OpenCode Desktop does not have to be installed.

## Installation

### Installer (recommended)

Run `VoiceCode-Setup.exe`. An animated installer window appears: it unpacks the
app into `%LOCALAPPDATA%\Programs\VoiceCode`, creates desktop and Start Menu
shortcuts, adds an entry to "Apps & features" and then offers to launch VoiceCode
right away. No administrator rights are required — everything is installed for the
current user in about 20 seconds.

On first launch Windows SmartScreen may warn that the publisher is unknown (the
installer is not code-signed). Choose **More info → Run anyway**.

To uninstall, use **Settings → Apps → VoiceCode** or
`%LOCALAPPDATA%\Programs\VoiceCode\uninstall.bat`.

### From source

```bat
install.bat
```

The script creates `.venv`, installs the dependencies and checks opencode, the
model, system voices, microphones and `pygame.mixer`, printing everything it finds.

Manually:

```bat
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe tools\warmup.py
```

If `opencode-cli.exe` is not installed, VoiceCode looks for it next to itself
(`VoiceCode.exe`), then in `PATH` and in
`%LOCALAPPDATA%\Programs\@opencodedesktop\resources\`. Keeping the CLI next to the
app is the most reliable option.

### Building the installer

```bat
.venv\Scripts\python.exe -m pip install pyinstaller
.venv\Scripts\python.exe -m PyInstaller packaging\voicecode.spec --noconfirm
.venv\Scripts\python.exe packaging\build_installer.py
```

The result is a single `dist\VoiceCode-Setup.exe` (≈420 MB). A local 7-Zip
installation is required (`C:\Program Files\7-Zip\7z.exe`); the `7za.exe` used on
the target machine is extracted by the script from the official "7-Zip Extra"
package.

How the installer is put together: `packaging/installer.spec` builds the PySide6
UI (animations, progress, success screen), and `packaging/build_installer.py`
appends `payload.7z` to it with a 16-byte trailer. The UI reads that trailer from
its own file, unpacks the payload with the bundled `7za.exe` and shows real-time
progress. Intermediate builds live in `dist\installer-ui` and
`%TEMP%\voicecode-installer` (`--reuse` reuses an existing `payload.7z`,
`--skip-ui` reuses an existing UI build).

## Running

```bat
run.bat          :: no console, uses pythonw.exe
run-debug.bat    :: console + verbose logging
```

If the Whisper model is missing, `run.bat` offers to run `install.bat` first. The
app minimizes to the tray; `Ctrl+Q` or **Exit** in the tray menu quits it
completely.

To check speech synthesis on an installed copy:

```bat
"%LOCALAPPDATA%\Programs\VoiceCode\VoiceCode.exe" --say "Проверка голоса"
```

The text is spoken by the selected voice (Russian by default) and the process exits
when playback finishes — a quick way to verify that the installed copy has sound.

## Controls

| Action | How |
| --- | --- |
| Talk | hold `Space` in the window, `F8` anywhere, or the "Hold to talk" button |
| Cancel the answer / stop playback | `Esc`, or start a new phrase, or say "стоп" ("stop") |
| Repeat the last answer | `Ctrl+R` or say "повтори" ("repeat") |
| Minimize to tray | the "—" button or say "скройся" ("hide") |
| Quit | `Ctrl+Q` or the tray menu |
| Move the window | drag the card |

`F8` is global and handled by the `keyboard` package. If it does not work in
apps running as administrator, run VoiceCode as administrator too or pick another
key in the config.

## Voice commands

Local commands are handled without calling opencode.

| Say (Russian) | What happens |
| --- | --- |
| «открой телеграм», «запусти калькулятор», «вруби блокнот» — "open Telegram", "launch Calculator", "start Notepad" | launches an app from the Start Menu index |
| «открой ютуб», «открой сайт гитхаб», «открой github.com» — "open YouTube", "open GitHub" | opens the website in the browser |
| «найди в интернете погоду на завтра», «погугли новости» — "search the web for tomorrow's weather" | web search in the browser |
| «который час», «какое сегодня число» — "what time is it", "what's the date" | time and date |
| «сделай скриншот» — "take a screenshot" | saves a screenshot to the Pictures folder |
| «громче», «тише», «выключи звук» — "louder", "quieter", "mute" | Windows volume |
| «сверни окно», «сверни все окна» — "minimize the window", "minimize everything" | minimizes the active window / shows the desktop |
| «закрой окно», «переключи окно» — "close the window", "switch window" | `Alt+F4`, `Alt+Tab` |
| «закрой телеграм» — "close Telegram" | terminates the app's process |
| «заблокируй компьютер» — "lock the computer" | locks Windows |
| «скройся» — "hide" | minimizes to tray |
| «выключись» — "shut down" | quits VoiceCode |
| «повтори», «стоп», «спасибо», «что ты умеешь» — "repeat", "stop", "thanks", "what can you do" | utility phrases |

Anything that is not a command goes to opencode as a normal question: "объясни,
что такое квантовая запутанность" ("explain quantum entanglement"), "напиши
функцию на Python" ("write a Python function"), "какая погода в Москве"
("what's the weather in Moscow").

## Configuration

All settings live in `config.json` next to `main.py`. The file is read at startup,
so edit it while the app is closed.

| Option | Default | Meaning |
| --- | --- | --- |
| `language` | `"ru"` | recognition language; `"auto"` detects it |
| `whisper_model` | `"models/faster-whisper-small"` | path to a local model or a Hugging Face model name (`"small"`, `"base"`, `"medium"`) |
| `whisper_compute` | `"int8"` | CTranslate2 compute type (`int8`, `int8_float32`, `float32`) |
| `whisper_threads` | `0` | CPU thread count; `0` means all cores minus two |
| `tts_engine` | `"edge"` | `"edge"`, `"system"` (offline SAPI only) or `"auto"` |
| `voice` | `"ru-RU-DmitryNeural"` | edge-tts voice; also `ru-RU-SvetlanaNeural`, `ru-RU-DariyaNeural` |
| `tts_rate` | `"+8%"` | edge-tts speech rate |
| `tts_volume` | `"+0%"` | edge-tts volume |
| `opencode_cli` | `""` | path to `opencode-cli.exe` if auto-discovery fails |
| `model` | `"opencode/deepseek-v4.1-flash-free"` | opencode model |
| `agent` | `"build"` | opencode agent |
| `workspace` | `"agent"` | opencode working folder (relative to the project) |
| `auto_approve` | `false` | adds `--auto` so opencode does not ask for confirmations |
| `keep_context` | `true` | include the latest turns in the prompt |
| `max_context_turns` | `6` | how many question/answer pairs to remember |
| `timeout_sec` | `150` | how long to wait for an opencode answer |
| `microphone` | `""` | microphone name; `""` means the default device |
| `sample_rate` | `16000` | recording sample rate |
| `global_hotkey` | `"f8"` | global push-to-talk key |
| `greeting` | `true` | say hello out loud after warm-up |
| `minimize_to_tray` | `true` | closing the window minimizes to tray |
| `search_url` | Yandex | search template, `{query}` is substituted |
| `show_caption` | `true` | show the answer line |
| `start_minimized` | `false` | start minimized to tray |

Example: the fastest profile for a low-end laptop.

```json
{
  "whisper_model": "models/faster-whisper-small",
  "whisper_threads": 4,
  "tts_engine": "system",
  "model": "opencode/deepseek-v4.1-flash-free"
}
```

### Speech recognition model

The bundle includes `faster-whisper-small` in int8 (≈ 240 MB) — a compromise
between Russian accuracy and CPU speed.

To rebuild the model from the original OpenAI checkpoint:

```bat
.venv\Scripts\python.exe -m pip install torch --index-url https://download.pytorch.org/whl/cpu
.venv\Scripts\python.exe -m pip install transformers openai-whisper
.venv\Scripts\python.exe tools\build_model.py
```

The script downloads `small.pt`, converts it to the Hugging Face format, builds a
tokenizer from the `openai-whisper` assets and compresses the model to CTranslate2
`int8`. For `base` (faster but less accurate):
`--size base --output models\faster-whisper-base`, then point `whisper_model` at it.

You can skip storing the model locally altogether: set `"whisper_model": "small"`
and faster-whisper will download it from Hugging Face on first launch (internet
required).

### Voice

By default VoiceCode uses `edge-tts`, which sounds natural but needs access to
`speech.platform.bing.com`. At startup it checks reachability once (2 s) and
switches to SAPI immediately if the endpoint is unavailable, so no time is wasted
on per-phrase timeouts.

Fully offline mode:

```json
{ "tts_engine": "system" }
```

SAPI picks a Russian voice available in Windows; if there is none, install one via
**Settings → Time & Language → Speech**.

### opencode

VoiceCode calls the CLI in stateless mode:

```
opencode-cli run --format json --model <model> --agent <agent> "<prompt>"
```

The opencode free tier does not accept `--session` and `--continue`, so the
conversation context is assembled locally into the prompt (`keep_context`,
`max_context_turns`). Every answer arrives as a stream of JSON events and is voiced
as the text arrives.

The working folder is `agent\`. `agent\AGENTS.md` tells the model to answer in
Russian, briefly, without markdown or code, because the answer is read aloud.

The fastest free models tested: `opencode/deepseek-v4.1-flash-free`
(≈ 2.0–2.5 s) and `opencode/big-pickle` (≈ 3.1 s). `glm-5.3-flash-free` and `mimo`
are noticeably slower, `minimax` takes over 15 s.

## Privacy

- Microphone audio lives only in RAM and is never written to disk.
- Recognition and local commands run on your machine.
- Only the recognized text (plus the latest turns if `keep_context` is on) is sent
  to opencode. Where it goes depends on the provider configured in opencode.
- `logs\voicecode.log` contains recognized phrases and service events; the file is
  rotated and can be deleted at any time.

## Diagnostics

```bat
.venv\Scripts\python.exe tools\warmup.py       :: environment and model check
.venv\Scripts\python.exe tools\route_check.py  :: which phrases go to opencode
```

`warmup.py` checks the CLI, the opencode version, Whisper loading, edge-tts, SAPI
voices, microphones, outputs and `pygame`. `route_check.py` prints the router's
decision for a set of phrases — handy when a command unexpectedly goes to the model.

Log: `logs\voicecode.log`.

### Troubleshooting

| Symptom | What to do |
| --- | --- |
| "opencode not found" | set the path in `opencode_cli`; check `opencode-cli --version` |
| "Failed to load the Whisper model" | build the model (`tools\build_model.py`) or set `"whisper_model": "small"` |
| "Microphone unavailable" | find the device name with `warmup.py` and put it into `microphone` |
| `F8` does not react | close apps that capture the key; try running as administrator; change `global_hotkey` |
| Text appears but there is no sound | check the Windows output device; set `"tts_engine": "system"` |
| edge-tts is silent or reports network errors | expected behind a proxy/firewall — SAPI takes over automatically |
| "OpenCode's free tier can only be used from within OpenCode" | opencode returns this for session flags; VoiceCode does not use them — check the CLI version and your account |
| Answers take too long | lower `max_context_turns`, switch models, keep `int8` and more threads |
| The window does not appear | the app is already in the tray; a second instance is blocked (a `.lock` file in `%TEMP%\voicecode`) |

## Project layout

```
voicecode/
  main.py                 entry point and CLI arguments
  config.json             settings
  install.bat             environment setup
  run.bat / run-debug.bat launch scripts
  agent/AGENTS.md         rules for opencode
  models/faster-whisper-small/  local recognition model
  docs/preview.png        UI screenshot
  tools/
    warmup.py             warm-up and diagnostics
    route_check.py        local router check
    build_model.py        Whisper model builder
    convert_openai_to_hf.py  OpenAI → Hugging Face converter (from transformers)
  voicecode/
    app.py                QApplication, single-instance lock
    config.py, paths.py   settings and paths
    stt.py                recognition thread (faster-whisper)
    opencode.py           CLI discovery, streaming call, JSON parsing
    intents.py            Russian local command router
    apps.py               Start Menu index, websites, launch/close
    audio/recorder.py     always-open microphone stream
    audio/speech.py       edge-tts + SAPI, sentence queue
    ui/window.py          state machine, hotkeys, tray, wiring
    ui/widgets.py         logo, orb, button, status, footer
    ui/theme.py           opencode palette
```

## Limitations

- The interface and commands are designed for Russian and Windows.
- An opencode answer cannot be interrupted server-side: a new phrase stops playback
  and ignores the late answer, but the CLI process lives until its timeout.
- Speech reads plain text only; markdown is stripped, but complex answers are still
  better requested in short form.
- The `small` model occasionally mishears unusual app names — pronounce them a bit
  slower or add the command manually in `voicecode/apps.py`.
