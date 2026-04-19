# Phase 0 Execution Plan: Manual SadTalker Validation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce one 10-second talking-head MP4 via SadTalker in ComfyUI, document all findings, and export the workflow JSON for use in Phase 1.

**Architecture:** ComfyUI-SadTalker node takes a portrait image + WAV audio and outputs an MP4. Voice is generated separately via Voicebox, then fed in as a local file. nvidia-smi monitors VRAM throughout.

**Tech Stack:** ComfyUI (http://192.168.20.120:8188), ComfyUI-SadTalker node (haomole), Voicebox (https://voice.locopuente.org), nvidia-smi, ffprobe.

---

### Task 1: Install ComfyUI-SadTalker

**Files:**
- Creates: `ComfyUI/custom_nodes/Comfyui-SadTalker/` (inside container)

- [ ] **Step 1: Open ComfyUI-Manager**

  Open `http://192.168.20.120:8188/` in your browser. Click **Manager** in the top-right menu bar.

- [ ] **Step 2: Search for and install SadTalker**

  Click **Install Custom Nodes**. In the search box type `SadTalker`. Find the entry by **haomole** (`Comfyui-SadTalker`). Click **Install**.

- [ ] **Step 3: Restart ComfyUI**

  Manager will prompt you to restart. Click **Restart**. Wait ~10 seconds, then reload `http://192.168.20.120:8188/`.

- [ ] **Step 4: Verify node is available**

  Double-click anywhere on the ComfyUI canvas → Add Node → search `SadTalker`. You should see at least one SadTalker node in the list. If not, check Manager → **Installed** tab and confirm status shows ✓.

- [ ] **Step 5: Note install details**

  In Manager → Installed, click the SadTalker entry and record the git commit hash shown. You'll need this for the findings doc.

- [ ] **Step 6: Commit checkpoint**

  ```bash
  cd ~/Projects/talking-head
  git commit --allow-empty -m "chore: Phase 0 checkpoint — SadTalker node installed"
  ```

---

### Task 2: Prepare Test Portrait

**Files:**
- Create: `docs/phase0/test-portrait.jpg` (committed to repo for reproducibility)

- [ ] **Step 1: Source or create a portrait image**

  Requirements: frontal face, good lighting, neutral background, minimum 256×256 px. Ideal: 512×512 px JPEG.

  Options:
  - Use a real photo of yourself
  - Generate one with Stable Diffusion in ComfyUI (any checkpoint, prompt: `portrait photo of a person, frontal, neutral background, studio lighting`)
  - Download a royalty-free portrait from unsplash.com

- [ ] **Step 2: Crop and resize to 512×512**

  On Puente (or your Mac):
  ```bash
  ffmpeg -i input.jpg -vf "crop=min(iw\,ih):min(iw\,ih),scale=512:512" \
    portrait-512.jpg
  ```

- [ ] **Step 3: Copy to project and commit**

  ```bash
  mkdir -p ~/Projects/talking-head/docs/phase0
  cp portrait-512.jpg ~/Projects/talking-head/docs/phase0/test-portrait.jpg
  cd ~/Projects/talking-head
  git add docs/phase0/test-portrait.jpg
  git commit -m "test: add Phase 0 test portrait"
  ```

---

### Task 3: Generate Test Audio via Voicebox

**Files:**
- Create: `docs/phase0/test-audio.wav`

- [ ] **Step 1: Write a short test script**

  2–3 sentences, ~10 seconds when read at a natural pace. Example:

  > "Hello, this is a test of the talking-head pipeline. We're validating that SadTalker can animate a portrait with cloned voice audio. If this looks acceptable, we'll move on to building the CLI."

- [ ] **Step 2: Generate audio in Voicebox**

  Open `https://voice.locopuente.org`. Select or create a voice (your own voice clone or any available voice). Paste the script. Generate. Download the output as WAV.

  If Voicebox exports MP3, convert to WAV:
  ```bash
  ffmpeg -i output.mp3 test-audio.wav
  ```

- [ ] **Step 3: Verify the WAV**

  ```bash
  ffprobe test-audio.wav
  ```
  Expected output includes: `Audio: pcm_s16le` or similar PCM encoding, sample rate 16000–44100 Hz, duration ~8–12 seconds. SadTalker requires mono or stereo PCM WAV — not MP3 or AAC.

- [ ] **Step 4: Copy to project and commit**

  ```bash
  cp test-audio.wav ~/Projects/talking-head/docs/phase0/test-audio.wav
  cd ~/Projects/talking-head
  git add docs/phase0/test-audio.wav
  git commit -m "test: add Phase 0 test audio clip"
  ```

---

### Task 4: Build the SadTalker Workflow in ComfyUI

**Files:**
- Create: `docs/workflows/sadtalker-zero-shot.json` (exported from ComfyUI)

- [ ] **Step 1: Start VRAM monitoring in a separate terminal**

  SSH into Puente in a second terminal window and run:
  ```bash
  nvidia-smi dmon -s mu -d 2
  ```
  This prints memory-used and GPU utilisation every 2 seconds. Leave it running throughout Tasks 4 and 5.

- [ ] **Step 2: Clear the ComfyUI canvas**

  In ComfyUI, press **Ctrl+A** to select all, then **Delete** to clear the default workflow.

- [ ] **Step 3: Add a Load Image node**

  Double-click canvas → Add Node → `Image` → `Load Image`. Click **choose file** and upload `test-portrait.jpg`.

- [ ] **Step 4: Add a Load Audio node**

  Double-click canvas → Add Node → search `Load Audio`. Upload `test-audio.wav`.

  If there is no Load Audio node, use **Load VAE Audio** or check if SadTalker accepts a file path string instead — some versions take a path input rather than an audio node. If so, add a **primitive string** node with the absolute path to the WAV on the Puente filesystem.

- [ ] **Step 5: Add the SadTalker node**

  Double-click canvas → Add Node → search `SadTalker`. Add the main **SadTalker** node. Connect:
  - `Load Image` → `source_image` input on SadTalker
  - `Load Audio` (or path string) → `driven_audio` input on SadTalker

- [ ] **Step 6: Configure SadTalker settings**

  Use these starting values (sensible defaults for a first run):
  | Parameter | Value | Notes |
  |-----------|-------|-------|
  | `preprocess` | `crop` | crops face region |
  | `still_mode` | `false` | allows head movement |
  | `use_enhancer` | `false` | skip GFPGAN on first run |
  | `batch_size` | `1` | |
  | `size` | `256` | start here; 512 uses more VRAM |
  | `pose_style` | `0` | neutral |
  | `expression_scale` | `1.0` | |

- [ ] **Step 7: Add a Save Video / Preview Video node**

  Double-click canvas → Add Node → search `Save Video` or `Video Combine`. Connect SadTalker's output to it. Set output filename prefix to `phase0-test`.

- [ ] **Step 8: Export the workflow**

  Click **Save (API format)** in the ComfyUI menu (or `Workflow` → `Export (API)`). Save the JSON as `sadtalker-zero-shot.json`.

  Copy to the project:
  ```bash
  cp ~/Downloads/sadtalker-zero-shot.json \
    ~/Projects/talking-head/docs/workflows/sadtalker-zero-shot.json
  cd ~/Projects/talking-head
  git add docs/workflows/sadtalker-zero-shot.json
  git commit -m "feat: add SadTalker zero-shot ComfyUI workflow JSON"
  ```

---

### Task 5: Run the Workflow and Record Findings

**Files:**
- Create: `docs/phase0/findings.md`

- [ ] **Step 1: Note the start time**

  ```bash
  date
  ```

- [ ] **Step 2: Queue the workflow**

  Click **Queue Prompt** in ComfyUI. Watch the progress bar. Note any model downloads that happen on first run (SadTalker will auto-download weights ~300 MB).

- [ ] **Step 3: Record VRAM peak**

  Watch the `nvidia-smi dmon` output. Note the highest memory-used value (MiB) you see during inference.

- [ ] **Step 4: Note inference time**

  When the workflow completes, note the elapsed time from Step 1. ComfyUI also shows execution time per node in the node headers after a run.

- [ ] **Step 5: Locate and review the output**

  Output MP4 will be in `ComfyUI/output/` inside the container, prefixed `phase0-test`. Copy it out:
  ```bash
  docker cp <container-name>:/root/ComfyUI/output/phase0-test_00001.mp4 \
    ~/Projects/talking-head/docs/phase0/phase0-output.mp4
  ```
  Play it and assess quality.

- [ ] **Step 6: Fill in the findings document**

  Create `docs/phase0/findings.md` with the template below, filling in all fields:

  ```markdown
  # Phase 0 Findings — SadTalker

  **Date:** YYYY-MM-DD
  **Hardware:** Puente, RTX 3090 (24 GB VRAM)

  ## Node version
  - ComfyUI-SadTalker commit: <hash from Task 1 Step 5>

  ## Model weights
  - Weights downloaded automatically: yes / no
  - Weight files and sizes:
    - `<filename>`: <size MB>

  ## Settings used
  - preprocess: crop
  - still_mode: false
  - use_enhancer: false
  - size: 256
  - (list any other non-default settings)

  ## Performance
  - Install time (node + weights): <N> minutes
  - Inference time: <N> seconds for a <N>-second clip
  - VRAM peak: <N> MiB / <N> GB

  ## Quality assessment
  - Lip sync accuracy: [poor / acceptable / good / excellent]
  - Head movement naturalness: [poor / acceptable / good / excellent]
  - Artefacts observed: <describe any, or "none">
  - Overall: [acceptable for Phase 1 / needs further tuning / not acceptable]

  ## Input format notes
  - Portrait: 512×512 JPEG ✓ / issues: <describe>
  - Audio: WAV PCM ✓ / issues: <describe>

  ## Gotchas / surprises
  - <anything that wasn't obvious or required a workaround>

  ## Verdict
  - [ ] Acceptable → proceed to Phase 1
  - [ ] Needs tuning → re-run with adjusted settings (describe below)
  - [ ] Not acceptable → evaluate next backend (Hallo3 or DreamTalk)
  ```

- [ ] **Step 7: Commit findings and output**

  ```bash
  cd ~/Projects/talking-head
  git add docs/phase0/findings.md docs/phase0/phase0-output.mp4
  git commit -m "docs: Phase 0 findings — SadTalker validation run"
  ```

---

### Task 6: Optional — Tune Settings if Quality is Poor

Only do this task if the verdict in findings.md is "needs tuning".

- [ ] **Step 1: Try with enhancer enabled**

  In the SadTalker node set `use_enhancer: true`. Re-queue. Note: GFPGAN enhancer adds ~30–60 seconds and ~2 GB VRAM. Record new inference time and VRAM peak.

- [ ] **Step 2: Try size 512**

  Set `size: 512`. Re-queue. Better quality, higher VRAM.

- [ ] **Step 3: Try still_mode true**

  Set `still_mode: true`. Reduces head movement — sometimes looks more natural for a static portrait.

- [ ] **Step 4: Update findings.md**

  Add a **Tuning runs** section to `docs/phase0/findings.md` recording each setting combination and the resulting quality change.

- [ ] **Step 5: Commit**

  ```bash
  git add docs/phase0/findings.md
  git commit -m "docs: Phase 0 tuning runs — updated findings"
  ```

---

### Task 7: Wrap-up

- [ ] **Step 1: Confirm exit criteria met**

  The output MP4 must:
  - Show the portrait face animated and speaking
  - Have plausible lip sync to the audio
  - Have no severe artefacts (face collapse, black frames, audio desync > 0.5 s)
  - Be something you'd show a colleague as a proof-of-concept

- [ ] **Step 2: Update ROADMAP.md checkboxes**

  Mark completed Phase 0 items in `docs/ROADMAP.md`:
  ```markdown
  - [x] Install ComfyUI-Manager
  - [x] Install the JoyVASA ComfyUI node (primary candidate; claims 8 GB VRAM)
  ```
  Note: the roadmap says JoyVASA but we used SadTalker — update the checkbox text:
  ```markdown
  - [x] Install video backend node (used SadTalker; JoyVASA has no ComfyUI node)
  - [x] Pick a test portrait + a short audio clip (generated in Voicebox)
  - [x] Run video backend manually in ComfyUI — produce one talking-head clip
  - [x] Document: install time, inference time, VRAM peak, subjective quality
  ```

- [ ] **Step 3: Final commit**

  ```bash
  cd ~/Projects/talking-head
  git add docs/ROADMAP.md
  git commit -m "docs: mark Phase 0 complete in roadmap"
  ```

- [ ] **Step 4: Confirm Phase 1 readiness**

  Before moving to Phase 1, verify you have all of:
  - [ ] `docs/phase0/test-portrait.jpg`
  - [ ] `docs/phase0/test-audio.wav`
  - [ ] `docs/phase0/phase0-output.mp4`
  - [ ] `docs/phase0/findings.md` with verdict = "acceptable"
  - [ ] `docs/workflows/sadtalker-zero-shot.json`

  The workflow JSON is the key input to Phase 1 — it becomes the template we parametrise.
