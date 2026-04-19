# Phase 0 Design: Manual SadTalker Validation

**Date:** 2026-04-19
**Status:** approved

## Goal

Prove the end-to-end ComfyUI talking-head pipeline works on the target hardware before writing any code. Produce one 10-second talking-head MP4 good enough to show a colleague.

## Context and decisions

- **JoyVASA** (original Phase 0 target) has no ComfyUI node — only a standalone Python repo. It is being evaluated separately outside this project.
- **SadTalker** is the pivot: `ComfyUI-SadTalker` (haomole) is available in ComfyUI-Manager, well-maintained, and confirmed working on RTX hardware.
- Hardware: Puente box, RTX 3090 (24 GB VRAM), ComfyUI running at `http://192.168.20.120:8188/`.
- Voice: Voicebox running at `https://voice.locopuente.org`.

## Stack

| Slot | Backend | Source |
|------|---------|--------|
| Voice | Voicebox (voice clone) | `https://voice.locopuente.org` |
| Video | SadTalker | `ComfyUI-SadTalker` via ComfyUI-Manager |
| Image | passthrough (portrait JPEG) | local file |
| Sync | none | — |

## Steps

1. **Install `ComfyUI-SadTalker`** via ComfyUI-Manager → Install Custom Nodes → search "SadTalker" → install → restart ComfyUI.
2. **Prepare test portrait** — clear frontal face image, ~512×512 px JPEG.
3. **Generate test audio** — write a 2–3 sentence script, generate WAV in Voicebox, download it.
4. **Build workflow** — load the SadTalker workflow in ComfyUI, wire portrait and WAV inputs, configure output path.
5. **Run and monitor** — submit the workflow; watch `nvidia-smi dmon` in a separate terminal for VRAM peak.
6. **Collect output** — download the MP4, assess quality.

## What to document

Record all of the following for use in the Phase 1 workflow template:

- SadTalker node version / git commit hash
- Model weights loaded and their source (auto-downloaded vs manual)
- Exact node input parameters used (enhancer, preprocess, still mode, etc.)
- Workflow JSON exported from ComfyUI (save as `docs/workflows/sadtalker-zero-shot.json`)
- Install time (node + model weights)
- Inference time for a 10-second clip
- VRAM peak (GB)
- Subjective quality: lip sync accuracy, naturalness, artefacts
- Any gotchas or manual steps required

## Exit criteria

A 10-second talking-head MP4 on disk that:
- Has plausible lip sync to the audio
- Doesn't exhibit severe artefacts (face collapse, jitter, black frames)
- You would show it to a colleague as a proof-of-concept

## Next step

If exit criteria are met → Phase 1 (CLI skeleton).
If quality is unacceptable → evaluate Hallo3 or DreamTalk using the same test portrait and audio.
