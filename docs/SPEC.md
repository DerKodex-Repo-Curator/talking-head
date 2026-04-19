# Specification

## Scope

A CLI tool that takes a character definition + a script and produces a
talking-head video. Orchestrates ComfyUI workflows; does not implement any ML
itself.

## Non-scope

- Training new architectures (only fine-tuning per-character LoRAs / voice
  models via ComfyUI nodes)
- Real-time / streaming generation
- Interactive workflow editing (use ComfyUI directly for that)
- Hosting / serving generated videos (write to disk, done)

## Architecture

### Layers

1. **CLI / API layer.** User-facing. Parses command, loads character + config,
   picks workflow, submits to the appropriate backend, polls, returns output path.
2. **Workflow layer.** JSON templates for each ComfyUI-based backend, with
   placeholders for user data (portrait path, audio, LoRA weights, prompt text).
3. **Backend adapter layer.** Python classes that know how to execute a specific
   backend, validate its requirements, and map backend-specific errors to
   user-friendly messages. Two adapter shapes are supported:
   - **ComfyUI adapter** *(preferred)*: fills a workflow template and submits via
     HTTP (`POST /prompt`, poll `/history/{id}`, fetch outputs). Used for any
     model that has a ComfyUI custom node.
   - **Subprocess adapter**: shells out to a standalone CLI (e.g. JoyVASA run
     directly on the host) for models without a ComfyUI node. Same interface to
     the layers above; different execution underneath.
4. **ComfyUI layer.** External. Accessed via HTTP (`POST /prompt`, poll
   `/history/{id}`, fetch outputs). Only used by ComfyUI adapters.

### Character concept

A **character** is a reusable bundle of identity assets:

```
characters/alice/
  character.yml          # config: which backends, which weights
  portrait.jpg           # for zero-shot backends
  voice-sample.wav       # for zero-shot voice cloning
  models/
    flux-lora.safetensors  # optional, from training
    xtts-ft.pt             # optional, from training
  base-video.mp4         # optional, for LatentSync workflows
```

A character can be:

- **Zero-shot**: just portrait + voice-sample, no trained models
- **Trained**: includes LoRA / fine-tuned TTS weights
- **Hybrid**: trained voice, zero-shot video (or vice versa)

### Backend slots

Four independent slots per character:

| Slot | Purpose | Example backends |
|---|---|---|
| `voice` | Text → audio | voicebox-clone, xtts-ft, speaches, elevenlabs |
| `image` | Text → person image (for trained pipelines) | flux-lora, sdxl-zero, passthrough |
| `video` | Image+audio → talking-head video | joyvasa, sadtalker, hallo3, ltx-2, wan21 |
| `sync` | Video+audio → lip-synced video (optional refinement) | latentsync, musetalk, none |

Not every character uses every slot. A minimal zero-shot character only uses
`voice` + `video`. A MikeBot-style trained character uses all four.

## CLI surface

```
talking-head init                                   # create global config
talking-head character create NAME [options]       # create a character profile
talking-head character train NAME [--voice|--lora] # run training workflow
talking-head character list
talking-head character delete NAME
talking-head generate NAME --script FILE [--out FILE] [--backend-overrides]
talking-head workflows list                         # show installed workflows
talking-head doctor                                 # check ComfyUI + nodes
```

## Config files

### Global (`~/.config/talking-head/config.yml`)

```yaml
comfyui:
  url: http://localhost:8188
  timeout_seconds: 600

defaults:
  voice: voicebox-clone
  image: passthrough      # no image-gen unless trained
  video: joyvasa
  sync: none

output:
  dir: ~/talking-head-output
  format: mp4
  resolution: 1080p
```

### Per-character (`characters/NAME/character.yml`)

```yaml
name: alice
display_name: "Alice Chen"

voice:
  backend: voicebox-clone
  sample: voice-sample.wav
  voicebox_url: https://voice.locopuente.org

video:
  backend: joyvasa
  portrait: portrait.jpg

sync:
  backend: none

# Optional — only present for trained characters
training:
  voice_model: models/xtts-ft.pt
  image_lora: models/flux-lora.safetensors
  lora_token: "AliceChen"
```

## Script input format

Plain text or markdown. Simple cases just pass the text to the voice backend.
Richer cases (MikeBot-style) use structured markdown with scene markers:

```markdown
# Topic: Or Gates

[scene: Alice at a whiteboard]
An Or gate outputs true when either input is true.

[scene: Alice at her desk]
Let me show you the truth table.
```

Scene markers are **optional** and only honoured by trained pipelines that
include an `image` backend. Zero-shot pipelines ignore them and use the
character's portrait for every scene.

## Workflow submission flow

1. Load character config.
2. Resolve backend classes for each slot.
3. Load the workflow template JSON for the resolved backend combination.
4. Fill placeholders with: portrait path, audio path (or text for TTS-first
   workflows), LoRA path, prompt text.
5. `POST` to ComfyUI `/prompt`.
6. Poll `/history/{prompt_id}` until complete.
7. Download output files from `/view`.
8. Post-process (concatenate scenes, add silence padding, ffmpeg) if needed.
9. Write to configured output dir.

## Error handling

- **ComfyUI unreachable**: fail fast with install hint.
- **Required node missing**: name the missing node and the ComfyUI-Manager
  command to install it.
- **VRAM OOM**: surface ComfyUI's error and suggest resolution/backend
  alternatives.
- **Character config invalid**: pydantic-style errors pointing to the exact YAML
  line.

## Future extensions (out of initial scope)

- **slide-stream integration**: talking-head as a slide-stream provider that
  renders a presenter over the slide deck.
- **Subtitle generation**: Whisper on the output audio, burn in or SRT sidecar.
- **Batch mode**: list of scripts → directory of videos.
- **Library mode**: Python API for embedding in other tools.
- **Web UI**: Gradio / FastAPI front-end, separate process.
