# voicemode-local-tutor

Run a fully local, voice-based AI assistant on Linux using [Claude Code](https://claude.com/claude-code) and [VoiceMode](https://github.com/mbailey/voicemode) — speech-to-text and text-to-speech both run on your own GPU, no cloud STT/TTS calls, no API keys for voice.

This is a step-by-step record of the install that got this working on a laptop with an NVIDIA GPU and 8GB of RAM, so someone with similar hardware can reproduce it without the trial and error.

## What this gives you

- **STT (speech-to-text):** [whisper.cpp](https://github.com/ggml-org/whisper.cpp) with the `small` model, GPU-accelerated via **Vulkan** — works on NVIDIA, AMD, or Intel GPUs without installing the full CUDA Toolkit.
- **TTS (text-to-speech):** [Kokoro-FastAPI](https://github.com/remsky/Kokoro-FastAPI), GPU-accelerated via PyTorch/CUDA — dozens of voices across 9 languages.
- **The "brain":** Claude Code, talking to both services through the [VoiceMode](https://github.com/mbailey/voicemode) MCP server.

Everything you say and everything the assistant says to you stays on your machine. Only your text prompts go to Anthropic's API, same as any normal Claude Code session.

## Requirements

| Component | Tested on |
|---|---|
| OS | Ubuntu 26.04 LTS (should work on any recent Debian-based distro) |
| GPU | NVIDIA GeForce RTX 2050, 4GB VRAM |
| RAM | 6.5GB (works, but tight — expect some swap usage during install) |
| Disk | ~9GB free (Python/CUDA cache + models) |
| Audio | Working microphone and output (PipeWire/PulseAudio) |

You do **not** need the system-wide CUDA Toolkit. Whisper uses Vulkan; Kokoro pulls its own CUDA runtime via pip. You do need the NVIDIA driver installed (`nvidia-smi` should work).

If you don't have an NVIDIA GPU, whisper.cpp's Vulkan backend still works on AMD/Intel GPUs. Kokoro will need its CPU fallback (`start-cpu.sh`) instead of the GPU path described below.

## 1. System dependencies

```bash
sudo apt install -y glslc libvulkan-dev spirv-headers libasound2-dev \
  portaudio19-dev libportaudio2 libsdl2-dev ffmpeg vulkan-tools \
  git cmake gcc g++ make
```

| Package | Why |
|---|---|
| `glslc` | Compiles Vulkan shaders (needed to build whisper.cpp with `GGML_VULKAN=1`) |
| `libvulkan-dev`, `spirv-headers` | Vulkan development headers |
| `libasound2-dev` | Required by `simpleaudio` (a VoiceMode dependency) |
| `portaudio19-dev`, `libportaudio2` | Required by `sounddevice` (audio capture) |
| `libsdl2-dev` | Required by whisper.cpp's installer |
| `ffmpeg` | Audio conversion inside `whisper-server` |
| `vulkan-tools` | Optional — lets you confirm GPU detection with `vulkaninfo --summary` |

## 2. Install uv

```bash
pip3 install --user --break-system-packages uv
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

(The official `astral.sh` installer failed over a flaky connection during testing — `pip` is a reliable fallback.) Add the `export PATH` line to your `~/.bashrc` so it persists.

## 3. Install Claude Code

Follow the instructions at [claude.com/claude-code](https://claude.com/claude-code) if you don't already have it, then register VoiceMode as an MCP server:

```bash
claude mcp add --scope user voicemode -- uvx --refresh voice-mode
```

## 4. Build whisper.cpp with Vulkan

```bash
git clone https://github.com/ggml-org/whisper.cpp.git ~/.voicemode/services/whisper
cd ~/.voicemode/services/whisper
git checkout v1.9.5

cmake -B build -DGGML_VULKAN=1 -DCMAKE_BUILD_TYPE=Release
cmake --build build -j2 --config Release
```

`-j2` keeps the build from exhausting RAM on low-memory machines; raise it if you have more cores/RAM to spare. The binary lands at `build/bin/whisper-server`.

**Why pre-build it this way:** if VoiceMode's own installer builds whisper from scratch, it defaults to requiring the full CUDA Toolkit on Linux and has no Vulkan option. If the folder already has a build checked out at the same tag, VoiceMode's installer detects it and skips straight to service setup.

### Download the "small" model

```bash
bash ./models/download-ggml-model.sh small
```

If `models/ggml-small.bin` ends up smaller than 466MB, or whisper-server fails with `not all tensors loaded`, the download was corrupted. Delete and retry:

```bash
rm -f models/ggml-small.bin
curl -L -o models/ggml-small.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin
```

### Register and start the service

```bash
uvx voice-mode service install whisper
uvx voice-mode service start whisper
```

(The installer also fetches a `base` model by default — you can ignore or delete it, `small` is what actually gets used once configured below.)

## 5. Install Kokoro-FastAPI

```bash
uvx voice-mode service install kokoro
uvx voice-mode service start kokoro
```

**First run downloads PyTorch + CUDA runtime libraries (~3GB total)**, then the Kokoro model itself (~330MB). This can take anywhere from 30 minutes to a few hours depending on your connection — it's safe to interrupt and resume later, completed packages stay cached and won't re-download.

Once ready, it responds on port 8880:

```bash
curl http://127.0.0.1:8880/health
# {"status":"healthy"}
```

## 6. Configuration

Edit `~/.voicemode/voicemode.env`:

```bash
VOICEMODE_WHISPER_MODEL=small
VOICEMODE_WHISPER_LANGUAGE=auto        # or a fixed code like en, es, fr...
VOICEMODE_VOICES=af_sky                # pick any installed Kokoro voice
VOICEMODE_VAD_AGGRESSIVENESS=1         # 0-3; lower tolerates pauses better
VOICEMODE_SILENCE_THRESHOLD_MS=2500    # silence required before it stops listening
VOICEMODE_INITIAL_SILENCE_GRACE_PERIOD=2.0
```

Notes:
- With `auto`, Whisper occasionally misdetects the spoken language (especially with background noise). Pinning a language avoids that, at the cost of not being able to freely switch languages mid-conversation.
- Kokoro ships dozens of voices across 9 languages (English US/UK, Spanish, French, Hindi, Italian, Japanese, Portuguese, Mandarin). List them with:
  ```bash
  ls ~/.voicemode/services/kokoro/api/src/voices/v1_0/*.pt
  ```
  Voice names follow the pattern `<lang><gender>_<name>` — e.g. `bf_emma` is British English, female.
- **Changing this file does not take effect live.** Restart Claude Code completely — `/mcp` → Reconnect is not enough, since VoiceMode's config is read once at process start.

## 7. Using it

1. Open a terminal, run `claude` inside any project folder (no IDE required).
2. Confirm VoiceMode is connected with `/mcp`.
3. Start talking:
   ```
   /mcp__voicemode__converse
   ```
4. **Use headphones.** If the microphone picks up the speaker output, Whisper transcribes the assistant's own voice and the conversation breaks down.

## 8. Maintenance commands

```bash
export PATH="$HOME/.local/bin:$PATH"

uvx voice-mode service status whisper
uvx voice-mode service status kokoro

uvx voice-mode service stop kokoro       # frees ~650MB RAM when not in use
uvx voice-mode service restart whisper

journalctl --user -u voicemode-whisper -n 50 --no-pager
journalctl --user -u voicemode-kokoro -n 50 --no-pager
```

Both services run as user-level systemd units (`~/.config/systemd/user/voicemode-*.service`). If you want them to survive logout, enable linger:

```bash
loginctl enable-linger "$USER"
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ffmpeg is not found` in whisper's log | Missing ffmpeg | `sudo apt install -y ffmpeg` |
| `not all tensors loaded from model file` | Model downloaded incompletely | Re-download `ggml-small.bin` (see step 4) |
| `PortAudio library not found` | Missing libportaudio | `sudo apt install -y portaudio19-dev libportaudio2` |
| `sudo: a terminal is required` during `service install` | The installer tries to call sudo non-interactively | Install the packages manually first (step 1) |
| Kokoro says "started" but never listens on port 8880 | Still downloading dependencies | Wait, watch `journalctl --user -u voicemode-kokoro -f` |
| Whisper transcribes nonsense or the wrong language | Auto-detection confused by noise/accent | Pin `VOICEMODE_WHISPER_LANGUAGE`, use headphones |
| Whisper outputs something like "thank you for watching" from silence | Known Whisper hallucination on background noise | Check mic gain/volume |
| Config changes don't seem to apply | The MCP process already loaded the old config | Fully restart Claude Code (not just `/mcp` reconnect) |
| Download speed wildly inconsistent | Other devices saturating the network | Pause other heavy network users and retry |

## License

MIT
