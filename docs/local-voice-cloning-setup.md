# Local Voice Cloning Setup for MoneyPrinterTurbo

Reviewed: August 28, 2026

> Status: local adapter with Qwen TTS 0.6B/current profile approved for Heritage
> Banner. Other engines and profiles remain experimental. The adapter is
> available to the campaign pipeline only when explicitly selected and must not be used
> with a person's voice without explicit permission. Tool and model licenses,
> provenance, retention behavior, and current system requirements must be
> rechecked from their official sources before any download or real use.

This is a local-first proof-of-concept plan for a Windows 11 laptop with modest CPU-only hardware, such as a Lenovo ThinkPad T590 with an Intel i5-8365U. The priority is reliable, slow-but-working generation, not GPU throughput.

Do not put private voice samples, generated voice files, API keys, or cloned-voice model caches in git.

## Safest Place In This Project

Use a sidecar workflow first:

- `docs/local-voice-cloning-setup.md`: this guide.
- `tools/local-voice/check-local-voice.ps1`: local readiness and endpoint checks.
- `local_voice/`: private working folders for reference clips, script snippets, and outputs.

MoneyPrinterTurbo has explicit local-provider integrations in:

- `app/services/voice.py`, via an OpenAI-compatible `POST /audio/speech` client.
- `webui/Main.py`, where "Chatterbox TTS" appears as a selectable TTS server.
- `config.example.toml`, under `[chatterbox]`.
- `app/services/voice.py`, via `voicebox:<profile-id>` and Voicebox's
  `POST /generate/stream` endpoint.
- `config.example.toml`, under `[voicebox]`.

The Voicebox path is opt-in and fail-closed. If the local service or selected
profile is unavailable, the render fails instead of substituting an Azure or
other generic voice. Manual WAV export remains the safest diagnostic fallback.

## Recommended First Path

Start with Voicebox.sh as a local GUI/API proof of concept.

Why this is the simplest first path for this machine:

- It ships a Windows desktop build with no project-level Python or CUDA setup required.
- It provides a local REST API at `http://127.0.0.1:17493`.
- It can manage voice profiles and engines through a GUI before any MoneyPrinterTurbo integration work.
- It avoids changing this repo while you learn the quality/speed limits of the laptop.

Expected tradeoff: CPU-only generation may be slow, and some engines may be impractical without GPU acceleration. Treat the first test as a short narration clip, not a batch workflow.

If Voicebox is too slow or cannot use a CPU-friendly engine on this laptop, the next practical path is an OpenAI-compatible Chatterbox server because this MoneyPrinterTurbo checkout already knows how to call that style of endpoint.

## Option A: Voicebox.sh

Official sources:

- Website: https://voicebox.sh/
- Source/releases: https://github.com/jamiepine/voicebox
- Local API docs advertised from the app/site: `http://127.0.0.1:17493`

What it gives you:

- GUI for cloning and managing voice profiles.
- Local API endpoints for generation, profiles, model status, and history.
- Multiple engines behind one desktop workflow.
- Windows MSI download.

Fit for this laptop:

- Best first experiment because setup friction is low.
- Do not assume every model will run acceptably on CPU.
- Choose the smallest/fastest available engine first.
- Keep first test text under 20 seconds of speech.

MoneyPrinterTurbo fit:

- Select a profile with `voice_name = "voicebox:<profile-id>"`.
- Voicebox narration accepts the existing `voice_rate` multiplier from 0.5
  through 2.0. Non-native values use FFmpeg's pitch-preserving `atempo`
  filter; script word count must still be planned for the intended duration.
- On memory-constrained machines, set `unload_after_generation = true`. MPT
  releases the local model after narration so FFmpeg can reuse that memory.
- The adapter streams WAV audio directly from Voicebox and writes it into the
  task audio path. It never uploads the sample or narration to a cloud service.
- A host MPT process defaults to `http://127.0.0.1:17493`. An MPT Docker
  container should use `http://host.docker.internal:17493` in `[voicebox]`.
- Heritage Banner uses `engine = "qwen"`, `model_size = "0.6B"`, and native
  `voice_rate = 1.0` after the product owner approved a private diagnostic for
  pacing and inflection. Keep other engine/profile combinations experimental.

## Option B: Chatterbox / OpenAI-Compatible Server

Official and common sources:

- Resemble AI Chatterbox: https://github.com/resemble-ai/chatterbox
- OpenAI-compatible community API wrapper: https://github.com/travisvn/chatterbox-tts-api

What it gives you:

- Open-source TTS with zero-shot voice cloning.
- Chatterbox model family with smaller Turbo and 500M-class multilingual models.
- API wrappers that expose OpenAI-style `/v1/audio/speech`.

Fit for this laptop:

- More setup risk than Voicebox.
- CPU can work, but expect slow generation and possible dependency friction.
- Community API docs recommend 8 GB+ memory and note CPU mode is slower.
- Do not download models until you intentionally choose the engine and confirm disk/RAM budget.

MoneyPrinterTurbo fit:

- This is already the closest integration match. The project can call:

```toml
[chatterbox]
base_url = "http://127.0.0.1:4123/v1"
api_key = ""
model_id = "chatterbox"
voices = ["default-Female"]
```

Then select "Chatterbox TTS" in the WebUI.

Important limitation: Chatterbox does not return word-level timestamps to MoneyPrinterTurbo's current client. For tighter subtitle sync, use `subtitle_provider = "whisper"` after accepting the CPU/model-size cost, or keep `edge` and manually review subtitles.

## Option C: Direct Qwen/Qwen3-TTS

Official source:

- Qwen3-TTS: https://github.com/QwenLM/Qwen3-TTS

What it gives you:

- Open-source Qwen TTS models with streaming speech generation, voice design, and vivid voice cloning.
- Research-grade model family under Apache-2.0.

Fit for this laptop:

- Not recommended as the first local path on this CPU-only machine.
- More moving parts than Voicebox and likely heavier model/runtime requirements.
- Good later research option if you move generation to a stronger workstation or rented GPU.

MoneyPrinterTurbo fit:

- There is no direct Qwen3-TTS integration in this checkout.
- Best future route would be to wrap Qwen3-TTS behind an OpenAI-compatible `/audio/speech` endpoint, then reuse the existing Chatterbox-style integration pattern.

## Local Folder Layout

Use this layout for private experiments:

```text
local_voice/
  reference/  # Put your private reference voice clips here.
  scripts/    # Put short narration text files here.
  output/     # Generated WAV/MP3 files go here.
```

The `local_voice/.gitignore` file keeps everything private except folder placeholders and the README.

Suggested first files you create manually:

```text
local_voice/reference/my-consented-reference.wav
local_voice/scripts/smoke-test.txt
local_voice/output/smoke-test.wav
```

Reference clip guidance:

- Use only your own voice or a voice you have explicit permission to clone.
- Start with 10 to 30 seconds of clean speech.
- Use WAV if possible, mono or stereo, 16 kHz or higher.
- Avoid music, noise, reverb, and overlapping speakers.

## Readiness Check

Run the local check script from the project root:

```powershell
powershell -ExecutionPolicy Bypass -File .\tools\local-voice\check-local-voice.ps1 -CreateFolders
```

What it does:

- Creates the local private folders when `-CreateFolders` is passed.
- Checks Windows, CPU, RAM, Python, uv, FFmpeg, Docker, and PowerShell.
- Checks whether Voicebox is reachable at `http://127.0.0.1:17493`.
- Checks whether a Chatterbox-compatible API is reachable at `http://127.0.0.1:4123/v1`.
- Does not download models.
- Does not generate audio.
- Does not read or upload voice samples.

## Smoke Test Plan

1. Prepare a reference voice clip.

   Put it in:

   ```text
   local_voice/reference/
   ```

   Keep it private and out of git.

2. Prepare short script text.

   Example:

   ```text
   This is a local narration smoke test for a short-form video workflow. The goal is clear speech, stable pacing, and an audio file that can be imported into MoneyPrinterTurbo.
   ```

   Save it as:

   ```text
   local_voice/scripts/smoke-test.txt
   ```

3. Generate a short output file.

   In Voicebox, create or select the voice profile, paste the script text, generate speech, and export:

   ```text
   local_voice/output/smoke-test.wav
   ```

   If using a Chatterbox API server instead, generate to:

   ```text
   local_voice/output/smoke-test.mp3
   ```

4. MoneyPrinterTurbo adapter smoke test.

   Start Voicebox, copy the consented profile id from its local profile list,
   and set `voice_name` to `voicebox:<profile-id>` for one generation-only
   draft. Keep the existing production voice configured until the comparison
   passes human review.

5. Review before integrating.

   Check:

   - Voice similarity is acceptable and consented.
   - Audio has no hallucinated extra speech.
   - Timing is short enough for the target video.
   - Volume is consistent.
   - Subtitles still make sense after the audio swap.

## Model Download Policy

Do not let setup scripts download large models automatically.

Before downloading any voice model, document:

- Tool and model name.
- Official source, model license, and code license.
- Evidence that the reference speaker consented to this specific use.
- Expected download size.
- Expected disk cache location.
- Whether the tool retains history, profiles, or source audio and how to delete it.
- Whether CPU mode is supported.
- Whether the model is required for the first smoke test.

For this laptop, prefer a small engine first and test one short line before downloading larger multilingual or high-quality models.

## Decision Summary

Recommended first path: Voicebox GUI/API through the explicit local adapter.

Fallback path: Chatterbox API server, because MoneyPrinterTurbo already has a compatible Chatterbox client.

Defer: direct Qwen3-TTS until you have a stronger machine or a server wrapper that exposes an OpenAI-compatible speech endpoint.

Activation still requires a consented sample, a short generated comparison,
and human scoring for naturalness, pacing, pronunciation, warmth, and brand fit.
Do not put the profile id, sample, or output audio in source control or evidence.
