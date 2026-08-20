# VOX-Style Documentary Collage Video Workflow (v2)

Automated production of VOX-style documentary paper-collage animation videos using two browser tools (v2: Artistly removed — images are generated inside VideoExpress):

| Tool | Role |
|---|---|
| **CloneVoice** (`app.clonevoice.ai`) | Narration audio (text-to-speech, "Tyler Brooks" voice by default) |
| **VideoExpress** (`app.videoexpress.ai`) | Per-beat image generation ("Create Image" in the Create Video From Prompt modal), image-to-video clips, timeline assembly, export |

The full machine-readable contract — every URL, DOM selector, API endpoint, checkbox value, corner case, and the resume protocol — lives in **`vox_workflow.json`** (v2.0.0). This README is the human overview.

---

## What the workflow produces

One finished mp4: a continuous "Fern-style" narrated documentary over hand-cut-newsprint collage scenes that assemble themselves stop-motion style, with the video ending exactly on the narration's last word.

## Pipeline at a glance

```
Phase 0  Auth gate            - verify CloneVoice and VideoExpress are logged in (blocker if not)
Phase 1  User inputs          - FIRST: own script or generate? Own script = used verbatim, no
                                suggestions, duration derived (words/150); Generate = 10 genres ->
                                pick one -> 5 ideas -> pick one -> run begins; + duration (1-5 min).
                                BOTH branches: ratio (Landscape 16:9 / Vertical 9:16)
Phase 2  Script and beats     - (generate branch only) narration script (minutes x 150 words),
                                beat table, image prompts
Phase 3  Narration            - CloneVoice Create Audio -> Tyler Brooks voice -> Create New Audio
                                -> Preview Segments (DRAFT!) -> click "Generate Audio" -> Completed
Prompt book + gate           - full storyboard authored per shot (title, TIME window, voiceover cue,
                                text-to-image prompt with exact labels, timestamped image-to-video
                                prompt) and self-checked against the prompt_gate checklist BEFORE
                                any generation (internal gate - never pauses the run)
Phase 5  Images               - in the SAME VideoExpress modal: image prompt -> image type 'other'
                                -> uncheck auto-enhance -> Create Image -> verify in My AI Images;
                                chosen ratio asserted, QC per image (max 3 takes/beat)
Phase 7  Clips                - same modal, right after each image: fixed video prompt, batches of 5
                                (account cap), per-beat length, acceptance verified by data.mediaId,
                                both enhancers OFF, Video Only ON
Phase 8  Assembly             - clips dragged sequentially (reverse-drop for correct order),
                                narration on the bottom audio track at 0, endpoints matched to 0px
Phase 9  Save + Export        - save (verify via document.title), export High/FullHD/mp4,
                                done ONLY at "Your movie creation is currently number N in the queue"
```

## Duration math (the sync contract)

Everything derives from the user's selected length so voice and picture match:

1. `TARGET_WORDS = minutes x 150` (2.5 words/sec) — the script is written to this.
2. After the narration renders, measure **A** = actual mp3 duration. **A is the single authority** — TTS pace drifts (verified: 72 words → 32.56 s ≈ 2.21 wps).
3. `N = ceil(A / 6)` beats → N images → N clips (batches of 5).
4. Per-clip planned length = `clamp(round(A/N), 3, 10)` seconds; +1 s spread evenly until planned total slightly **exceeds** A. Longer clips are distributed evenly, never clustered.
5. Assemble, then trim the overshoot from the last clip at the narration endpoint (playhead + Cut + delete tail) until `video_end == narration_end` at **0 px**.

Example: 3 minutes → 450 words → A ≈ 180 s → 30 beats/images/clips → 6 batches of 5.

## State management and Resume

The run maintains **`WORKFLOW_STATE.json`** next to the workflow file:

- One atomic checkpoint after every **verified** side effect (job accepted, image completed, brick placed…). Checkpoints record proof (IDs, statuses, durations, px), never clicks.
- Every error is logged with phase, step, exact symptom text, root cause, recovery, and outcome.
- If the run dies, the user says **"Resume"**: the runner reloads the state, re-checks auth, **reconciles the failed step against the live app** (a client-side error often succeeded server-side), marks existing results verified, and retries only the smallest missing action. Completed phases are never re-run.

## Hard rules (learned from live runs)

- **Never** enter credentials or API keys. A login page or a missing-key panel is a *user* action.
- Max **5 concurrent** VideoExpress generations per account (shared across sessions). Over-cap submissions are rejected **silently with no record** — acceptance is proven only by a new My AI Videos record whose `get_media_prompt_data.data.mediaId` equals the submitted image id.
- Timeline drops insert at **position 0** — drop clips in reverse beat order for a sequential result, verify count and order after every drop.
- Both prompt enhancers OFF, Video Only ON, "Share in public gallery" OFF, image type `other` (never `human` for collage).
- CloneVoice "Create New Audio" only makes a **draft** — the audio renders when **Generate Audio** is clicked on the Preview Segments page.
- Decline any "make this the account default" popup.
- Verify by API / `document.title` / queue text — never by a toast.
- Avoid text labels inside image prompts (models garble short labels ~50% of takes); add text via VideoExpress Text Animations instead.

## How to use

1. Make sure both tools (CloneVoice and VideoExpress) are logged in inside the browser the AI controls.
2. Connect the CloneVoice integration in the VideoExpress profile (needed to import the narration onto the timeline).
3. Give the AI runner `SYSTEM_PROMPT.md` (and `vox_workflow.json` alongside it).
4. Answer its first question — paste **your own script** (used verbatim, duration derived from word count) or say **generate** (it then asks niche/idea and duration 1–5 min). Both paths ask for the ratio.
5. Let it run. After your Phase 1 answers the run is **fully automatic** — no confirmation gates, no "type yes to continue"; it reports progress briefly while working and stops only for true blockers (logins, payments, CAPTCHA).
6. If a run is interrupted, say **"Resume"**.

## Files

| File | Purpose |
|---|---|
| `vox_workflow.json` | The full executable contract (selectors, APIs, rules, corner cases) |
| `SYSTEM_PROMPT.md` | Drop-in system prompt for any AI agent (Claude, ChatGPT/Codex, etc.) |
| `WORKFLOW_STATE.json` | Created at runtime — live state, checkpoints, error history |
| `README.md` | This file |
