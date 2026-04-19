# Roadmap

Phases are sized so each produces something testable. Don't move to the next
phase until the current one generates an actual video you'd show someone.

## Phase 0 — Manual ComfyUI run-through

**Goal:** prove the end-to-end flow works for one character on the target
hardware, before writing any code.

- [ ] Install ComfyUI-Manager
- [ ] Install the JoyVASA ComfyUI node (primary candidate; claims 8 GB VRAM)
- [ ] Pick a test portrait + a short audio clip (generated in Voicebox)
- [ ] Run JoyVASA manually in ComfyUI — produce one talking-head clip
- [ ] Document: install time, inference time, VRAM peak, subjective quality
- [ ] If JoyVASA output is acceptable → proceed to Phase 1
- [ ] If not → repeat with SadTalker, Hallo3, DreamTalk in that order

**Exit criteria:** a 10-second talking-head MP4 on disk that you'd be willing to
show to a colleague.

## Phase 1 — CLI skeleton + first backend

**Goal:** script the Phase 0 workflow so it's repeatable from the command line.

- [ ] Python package scaffolding (pyproject, CLI entry point, config loader)
- [ ] ComfyUI HTTP client (submit prompt, poll, download output)
- [ ] `character create` command (writes YAML, copies portrait + voice sample)
- [ ] `generate` command for a single backend combination (the one that worked
      in Phase 0 — likely `voicebox-clone` + `joyvasa`)
- [ ] Workflow template adapted from manual Phase 0 run, with placeholders
- [ ] `doctor` command: checks ComfyUI reachable + required nodes installed

**Exit criteria:** `talking-head generate alice --script test.md` reproduces a
Phase 0-quality video from the CLI.

## Phase 2 — Second backend, pluggable selection

**Goal:** prove the backend-selection mechanism works; identify shared vs
backend-specific code.

- [ ] Add a second video backend (likely SadTalker or Hallo3) as a second
      workflow template + adapter
- [ ] CLI flag to override backend per-run (`--video sadtalker`)
- [ ] Per-character default backend in YAML
- [ ] Compare outputs between backends on the same input; note quality differences

**Exit criteria:** same character, same script, two different backends produce
two videos from the CLI. Users can pick.

## Phase 3 — Voice backends decoupled

**Goal:** voice is its own slot, not bundled with video.

- [ ] Voicebox adapter (call external Voicebox API, fetch audio, pass to ComfyUI)
- [ ] Speaches adapter (fallback, generic TTS, no cloning)
- [ ] XTTS adapter (zero-shot path first, leave fine-tuning for later)
- [ ] Workflow templates updated to accept audio from voice backend

**Exit criteria:** one character with Voicebox voice, another with Speaches,
both work; voice can be swapped per-character.

## Phase 4 — Optional LatentSync refinement

**Goal:** add the video→lip-sync refinement stage as an optional post-processor.

- [ ] LatentSync ComfyUI node evaluated
- [ ] `sync: latentsync` backend slot wired into workflow generation
- [ ] Base-video caching per-character so Phase 4 can reuse a Phase 1-generated
      base instead of regenerating every time
- [ ] CLI short-circuit: if a character has a cached base video, skip the
      `video` stage and run only `voice` + `sync`

**Exit criteria:** the two-stage pipeline (base video once, then cheap
dialogue-only regenerations) works end-to-end.

## Phase 5 — Training support

**Goal:** move beyond zero-shot for characters that need it.

- [ ] `character train --voice` command (XTTS fine-tuning via Coqui/ComfyUI)
- [ ] `character train --lora` command (Flux LoRA training via Kohya_SS in
      ComfyUI, following MikeBot's approach)
- [ ] Training progress surfaced in CLI (ComfyUI progress stream → stdout)
- [ ] Trained weights auto-registered in character config on success

**Exit criteria:** one character trained end-to-end, one character left
zero-shot, both generate via the same CLI.

## Phase 6 — MikeBot-shaped pipeline (varied scenes)

**Goal:** support the full trained pipeline with multiple scenes per video.

- [ ] Structured script parser (scene markers from the SPEC)
- [ ] `image` backend slot: Flux+LoRA or SDXL for per-scene images
- [ ] Multi-segment generation (one video per scene, concatenated)
- [ ] ffmpeg post-processing for transitions + audio alignment
- [ ] Llama 3 script-generation helper (optional: turn a topic into a structured
      script using a prompt adapted from MikeBot)

**Exit criteria:** one character appears in ≥3 different scenes in a single
video, each with appropriate imagery, narrated end-to-end.

## Phase 7 — Polish

- [ ] Subtitle sidecar (SRT) via Whisper on generated audio
- [ ] Batch mode (directory of scripts → directory of videos)
- [ ] Python API for library use (slide-stream integration, etc.)
- [ ] Documentation: tutorials for each phase's capability level

## Explicit non-roadmap items

These are deliberately excluded unless a concrete use case appears:

- Web UI (use ComfyUI if you want a UI)
- Real-time / streaming output
- Multi-speaker conversation videos (one character per generation; use multiple
  generations + edit for dialogue)
- Hosting / sharing / auth (write to disk, done)
