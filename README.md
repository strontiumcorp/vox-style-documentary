# VOX-Style Documentary Collage Video Workflow (v2)

Automated production of VOX-style documentary paper-collage animation videos using two browser tools (v2: Artistly removed — images are generated inside VideoExpress):

| Tool | Role |
|---|---|
| **CloneVoice** (`app.clonevoice.ai`) | Narration audio (text-to-speech, "Tyler Brooks" voice by default) |
| **VideoExpress** (`app.videoexpress.ai`) | Per-beat image generation ("Create Image" in the Create Video From Prompt modal), image-to-video clips, timeline assembly, export |

The full machine-readable contract — every URL, DOM selector, API endpoint, checkbox value, corner case, and the resume protocol — lives in **`vox_workflow.json`** (v3.3.1, also embedded inside `SYSTEM_PROMPT.md`). This README is the human overview.

## ⚠️ IMPORTANT — Supported AI models (runners)

A run is 100+ browser actions over 30–60 minutes. What it demands is **stamina and instruction adherence**, not raw problem-solving — so the runner tier matters more than anything else in this repo.

### ChatGPT / Codex — GPT-5.6

GPT-5.6 has two independent dials: the **capability tier** (Luna → Terra → Sol) and the **reasoning effort** (Light / Medium / High / Extra High). They do not substitute for each other — Luna at High reasoning is still Luna, and Sol at Light can beat Luna at High.

| Tier | Positioning | This workflow |
|---|---|---|
| **Luna** | fastest, cheapest, high-volume | ❌ stalls constantly |
| **Terra** | balanced, everyday work | ❌ stalls constantly |
| **Sol** | flagship: complex reasoning, coding, agents | ✅ **use this — Medium reasoning or higher** |

**✅ Recommended: GPT-5.6 Sol · Medium** (or High for long 4–5 minute runs).

**❌ Luna Light / Terra Light are not supported.** Verified in live testing, they: ask permission mid-run, hand UI clicks back to the user ("please open the modal and reply Resume"), yield the turn after each phase, and let the browser session get cleaned up between turns. The run still *converges* thanks to the checkpoint/Resume protocol — but only with constant nudging.

### Claude

✅ Opus-class models, verified up to **Opus 4.8** (Claude Code / agent mode).

### Why a prompt can't fix a weak tier

Instructions govern behavior *within* a turn; they cannot restart a turn that has already ended. When a light model yields, no rule in this file is running. The only real fixes are a stronger tier, an external auto-continue loop that re-sends "Resume", or deliberately chunked runs (one 5-shot batch per turn — always saved and checkpointed, so nothing is lost).

**Litmus test for any new model:** run a **1-minute video**. A supported runner completes it start-to-export in one continuous run with zero nudges. If it needs nudging at 1 minute, it will need dozens at 5.

---

## 🎬 Result samples

Finished videos produced by this workflow:

- [Result 1](https://drive.google.com/file/d/1-iMq3C7cch0x8_WySU8jvbQSifnhlGkD/view?usp=sharing)
- [Result 2](https://drive.google.com/file/d/190Ct-MAxGk56VzqnZJal9Hpuu_y8QaNZ/view?usp=sharing)

## What the workflow produces

One finished mp4: a continuous "Fern-style" narrated documentary over hand-cut-newsprint collage scenes that assemble themselves stop-motion style, with the video ending exactly on the narration's last word.

## Pipeline at a glance

```
Phase 0  Auth gate            - verify CloneVoice and VideoExpress are logged in (blocker if not)
Phase 1  User inputs          - ONE message asks all three: own script or generate, ratio
                                (Landscape 16:9 / Vertical 9:16), duration (1-5 min; derived from
                                word count for an own script). Generate = ONE more message with 10
                                genres; the agent then picks the story itself (reply IDEAS for 5
                                options). Then the run begins - no further questions
Phase 2  Script and beats     - (generate branch only) narration script (minutes x 150 words),
                                beat table, image prompts
Phase 3  Narration            - CloneVoice Create Audio -> Tyler Brooks voice -> Create New Audio
                                -> Preview Segments (DRAFT!) -> click "Generate Audio" -> Completed
Prompt book + gate           - full storyboard authored per shot (title, TIME window, voiceover cue,
                                text-to-image prompt with exact labels, timestamped image-to-video
                                prompt) and self-checked against the prompt_gate checklist BEFORE
                                any generation (internal gate - never pauses the run)
Phase 5  Images               - in the SAME VideoExpress modal (TAB A, configured once; TAB B
                                monitors): image prompt -> image type 'other' -> uncheck
                                auto-enhance -> Create Image; FAST-QC: accept the first take,
                                retake max 1 only on an obvious error
Phase 7  Clips                - same modal, right after each image: that shot's timestamped
                                image-to-video prompt, rolling 5-slot batching (submit 5, check
                                My AI Videos in TAB B, backfill freed slots), acceptance verified
                                by data.mediaId, both enhancers OFF, Video Only ON
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
- Labels in images follow the prompt-book law: exact label text named in the scene AND in the closer's "no text beyond …" clause (short labels garble ~50% of takes — first take ships with a noted exception unless obviously broken).
- Two-tab pattern: TAB A holds the configured generation modal permanently; TAB B does all Media Library monitoring — never close/reconfigure the modal between shots.

## How to use

### One-time setup

1. Sign in to **CloneVoice** (`app.clonevoice.ai`) and **VideoExpress** (`app.videoexpress.ai`) in the browser the AI controls.
2. Connect the **CloneVoice integration** in the VideoExpress profile (needed to import the narration onto the timeline).
3. Use a **supported model** (see the IMPORTANT section above).

### Start a run

4. Paste **`SYSTEM_PROMPT.md`** into the AI (one file — the full contract is embedded inside it; no attachments needed).
5. The AI first verifies both logins (auth gate). If either app is logged out, it will ask you to sign in, then continue.

### Answer its questions (the only questions in the whole run)

6. **Q1 — "Do you have your own narration script, or should I generate one from an idea?"**
   - Reply **"my script"** and paste your script → it is used **verbatim** (never rewritten), duration is derived automatically from the word count (words ÷ 150; 750-word / 5-minute cap). Skip to Q4.
   - Reply **"generate"** → continue to Q2.
7. **Q2 — Genre.** It suggests **10 genres** (crime & documentary, history, money & power, disasters & survival, mysteries & the unexplained, technology, sports, science & nature, war & espionage, aviation & exploration — or type your own). Reply with a number or your own genre.
8. **Q3 — Idea.** It pitches **5 concrete ideas** in your genre (each with a real date/name/number/place hook). Reply with a number, or describe your own topic.
9. **Q4 — Ratio and duration.** **"Landscape (16:9) or Vertical (9:16)?"** (always asked, never guessed) and — generate branch only — **"How many minutes, 1–5?"**

### Then it runs by itself

10. From the final answer onward the run is **fully automatic** — no "type yes to continue", no approvals. In order, it will: write & show the script (FYI only) → create the narration in CloneVoice (Tyler Brooks voice, incl. the mandatory "Generate Audio" click) → measure the real audio length and plan N beats → author the full per-shot prompt book and self-check it (internal gate) → generate every image + clip inside the VideoExpress modal (two tabs: one generating, one monitoring; rolling 5-slot batching) → assemble the timeline, place the narration on the bottom track, match endpoints to 0 px → save → export (High / FullHD / mp4).
11. It reports brief progress while working and stops **only** for true blockers: a login page, CAPTCHA, payment/credits, or an unrecoverable app error.
12. Done = it shows the export queue confirmation ("Your movie creation is currently number N in the queue") plus a final report with every ID and proof. The finished mp4 appears in VideoExpress → **My Videos** a few minutes later.

### If something breaks

13. Say **"Resume"**. The AI reloads `WORKFLOW_STATE.json`, re-checks logins, reconciles the failed step against the live apps (never duplicating work already done), and continues from exactly where it stopped.

## Files

| File | Purpose |
|---|---|
| `vox_workflow.json` | The full executable contract (selectors, APIs, rules, corner cases) — the editing source |
| `SYSTEM_PROMPT.md` | STANDALONE drop-in prompt for any AI agent (Claude, ChatGPT/Codex) — the full JSON contract is embedded inside it; paste this ONE file, nothing else needed |
| `WORKFLOW_STATE.json` | Created at runtime — live state, checkpoints, error history |
| `README.md` | This file |
