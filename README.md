# talking-head

A pluggable CLI for generating talking-head videos from a portrait and a script.
Thin orchestrator over [ComfyUI](https://github.com/comfyanonymous/ComfyUI); every
model is a swappable backend.

**Status:** specification draft. No working code yet. See [`docs/SPEC.md`](docs/SPEC.md)
and [`docs/ROADMAP.md`](docs/ROADMAP.md) for the plan.

## The idea in one diagram

```
┌──────────────────────────────────────────────────────────┐
│  talking-head CLI                                        │
│  • Character profile (portrait or LoRA, voice, config)   │
│  • Script (markdown or plain text)                       │
│  • Picks a workflow, fills in values, submits            │
└────────────────────┬─────────────────────────────────────┘
                     │ HTTP → ComfyUI API
┌────────────────────▼─────────────────────────────────────┐
│  ComfyUI (you run this separately — not reinvented here) │
│  Voice:   Voicebox / XTTS fine-tuned / Speaches / ...    │
│  Image:   Flux + per-person LoRA / zero-shot SDXL / ...  │
│  Video:   SadTalker / Hallo3 / LTX-2 / WAN 2.1 / ...     │
│  Sync:    LatentSync / MuseTalk / (none) / ...           │
└──────────────────────────────────────────────────────────┘
```

## Two supported character types

| Type | Setup | Trade-off |
|---|---|---|
| **Zero-shot** | 1 portrait + 10 sec voice sample | Start in minutes. Uncanny valley risk, limited scene variety. |
| **Trained** | 20 photos + 10 min of audio, one-time training (~6–8 hrs) | Higher quality, appears in varied scenes/poses, consistent identity forever after. |

Same CLI, same commands, different workflow submitted under the hood.

## Two example use cases

- **Lecturer content.** Lecturer records a short base video once (or trains a LoRA).
  Every subsequent week, feed a script → get the lecturer explaining this week's
  topic. Companion to [slide-stream](https://github.com/michael-borck/slide-stream).
- **Fictional personas.** LocoEnsayo characters (e.g. the CEO of a simulated
  company) need a consistent visual identity across videos. Train a LoRA on
  AI-generated reference images of the character; generate intro videos, briefings,
  dialogue scenes, all with the same face.

## Design principles

- **ComfyUI is the execution layer.** We orchestrate, we don't reinvent. Every
  backend exists as a ComfyUI node maintained by its own community.
- **Pluggable backends.** Voice, image, video, lip-sync are four separate backend
  slots. Pick one per character.
- **Local-first.** Nothing leaves your machine by default. Cloud backends
  (ElevenLabs, HeyGen) can be adapters, not defaults.
- **Per-character config.** Global defaults, per-character overrides. One YAML
  per character under `characters/`.
- **Graceful fallback.** If a backend isn't configured, fall back to a simpler
  one (e.g. no voice clone → Speaches generic TTS). Same pattern as
  [slide-stream](https://github.com/michael-borck/slide-stream).
- **Honest output.** Following Lewis Stuart's MikeBot disclaimer verbatim: the
  majority of output will need curation. This tool is a workflow accelerator, not
  a "press button receive finished video" appliance.

## Prior art and inspiration

This project stands entirely on the shoulders of other people's work. If it's
useful, the credit belongs to them.

- **[MikeBot](https://github.com/Lewis-Stuart-11/MikeBot)** (Lewis Stuart) — the
  direct architectural inspiration. Demonstrated that thin Python over ComfyUI
  workflows is the right shape for this problem. Workflow JSONs in his repo are
  excellent reference implementations.
- **[slide-stream](https://github.com/michael-borck/slide-stream)** — the
  pluggable-backend CLI pattern this project mirrors.
- **[ComfyUI](https://github.com/comfyanonymous/ComfyUI)** — the execution layer
  everything runs on.

**Talking-head / portrait animation models:**

- **[JoyVASA](https://github.com/jdh-algo/JoyVASA)** — JD Health, MIT, actively
  maintained, claims RTX 4060 8 GB VRAM. Current top pick for zero-shot.
- [JoyHallo](https://github.com/jdh-algo/JoyHallo) — Mandarin-optimised Hallo
  variant from same team, MIT
- [SadTalker](https://github.com/OpenTalker/SadTalker) — zero-shot, lightest,
  Apache-2.0, upstream abandoned but ComfyUI community active
- [Hallo / Hallo2 / Hallo3](https://github.com/fudan-generative-vision/hallo)
- [LivePortrait](https://github.com/KwaiVGI/LivePortrait) — Kuaishou,
  expression/pose transfer
- [DreamTalk](https://github.com/ali-vilab/dreamtalk) — Alibaba
- [MuseTalk](https://github.com/TMElyralab/MuseTalk) — Tencent, real-time lip
  sync
- [Sonic](https://github.com/jixiaozhong/Sonic) — Tencent, audio-driven portrait
- [EchoMimicV2](https://github.com/antgroup/echomimic_v2) — Alibaba, half-body

**Video-synchronisation:**

- [LatentSync](https://github.com/bytedance/LatentSync) — ByteDance,
  SD-based lip re-sync, Apache-2.0

**Video generation (for trained pipelines):**

- [LTX-2](https://github.com/Lightricks/LTX-2) — Lightricks, audio-video
  foundation model
- [WAN 2.1](https://github.com/Wan-Video/Wan2.1) — Alibaba, used in MikeBot
- [HunyuanVideo](https://github.com/Tencent/HunyuanVideo) — Tencent

**Voice:**

- [Voicebox](https://github.com/jamiepine/voicebox) — local voice cloning
  studio, integrated with puente
- [XTTS v2 / Coqui](https://github.com/coqui-ai/TTS) — voice fine-tuning
- [Speaches](https://speaches.ai) — local Whisper STT + Kokoro/Piper TTS

**ComfyUI nodes we'll depend on:**

- [ComfyUI-SadTalker](https://github.com/haomole/Comfyui-SadTalker)
- [ComfyUI_DreamTalk](https://github.com/hay86/ComfyUI_Dreamtalk)
- [JoyHallo-ComfyUI](https://github.com/AIFSH/JoyHallo-ComfyUI)
- [ComfyUI-Manager](https://github.com/Comfy-Org/ComfyUI-Manager)

## Non-goals

- **Training new base models.** We use whatever the community has published.
  Per-character LoRAs are fine (one-off cost); novel architectures are not.
- **Competing with HeyGen.** HeyGen's premium avatars remain noticeably better.
  The point is institutional data sovereignty and zero per-minute cost, not
  matching cloud quality.
- **Live/real-time generation.** Batch generation only. Real-time adds orders of
  magnitude of complexity for use cases we don't have.
- **Being a ComfyUI replacement.** If you want to build workflows interactively,
  use ComfyUI directly. This tool is for repeatable, scripted generation from a
  CLI.

## License

MIT. See [`LICENSE`](LICENSE).
