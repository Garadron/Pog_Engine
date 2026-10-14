# Unreleased

## Bug fixes

- Minutes of transcript were missing, replaced by one sentence repeated over and over (up to 700 captions in a row), with real speech underneath that Whisper never decoded. Cause: whisper.cpp feeds the previous window's text back into the next window by default, and on streamer audio (sparse speech over silence) the decoder latches onto its own last caption. Whisper now decodes every 30 s window independently (`-mc 0`, new TRANSCRIPTION_MAX_CONTEXT setting); as a bonus, punctuation and casing come back too.
- The loop detector marked looped chunks clean. A fast harmless run could hide a longer slow loop in the same chunk, and the alternating "A / B / A / B" check was skipped whenever any run of 6+ existed. Every shape is checked now and the widest wins, plus a new diversity gate (16 captions with at most 3 distinct texts in 180 s). New TRANSCRIPTION_LOOP_DIVERSITY_* settings.
- Captions Whisper invented over dead air or faint room noise are dropped before the loop check: any text over digital silence, "Thank you."-style filler under a low audio level (new TRANSCRIPTION_GENERIC_PHRASE_RMS setting, judged on the caption's own span so a 30 s filler block is not saved by the speech that starts right after it), and the "Subtitles by the Amara.org community" credit line at any level. Five test VODs had 212 such filler captions that used to reach the LLM. The drops are listed in transcription_warnings.json.

## Changed

- Step 2 decodes every chunk once. The retry layer that re-transcribed looped spans in halves is gone, along with TRANSCRIPTION_RETRY_MIN_MINUTES and TRANSCRIPTION_RETRY_BUDGET_FACTOR: with `-mc 0` it never fired on 43 chunks (about 20 hours of VOD), and a retry used the same audio and flags. A loop that still shows up is flagged in transcription_warnings.json and collapsed in Step 3. Old retry sub-chunk files and chunk_retries.json are cleaned up on the first run.
- Step 3 collapses decoder loops with the same detector Step 2 uses, so the two steps cannot disagree about what a loop is.
- Existing VODs re-decode in Step 2 once after updating (chunk audio is reused; roughly 10 minutes per VOD on a recent NVIDIA GPU). Re-run steps 3-5 afterwards.

# v2.0.0

## COMPLETE GUI REDESIGN: Run All window rebuilt from scratch

The old step list is gone. The Run All window is a pipeline map now: Evangelion-inspired layout, every step and mini-step is detailed, wiring that lights up as steps hand off, glow in each cell's state color. Twitch VODs get their own 1a-1g cells so you can actually see Demucs chunking, separation, concat, and rendering instead of one stuck progress bar. Hovering a cell tells you what that step does. It opens maximized with a gallery panel, falls back to a screen saver panel on smaller windows.

## New implemented features

- VOD folders are organized by step now (step1_extract_mic_audio/, step2_transcribe_audio/, etc.) instead of everything piled at the root. Old folders get moved automatically when you open them.
- You can use llama.cpp instead of Ollama if you want. There's an LLM_BACKEND setting for it, and the pipeline handles swapping the discovery/judge models itself. (UNTESTED as I prefer Ollama for ease of use)
- There's an updater now. Update_PogEngine.bat pulls the latest release, no more copying files by hand.
- Step 2 reworked to catch Whisper repeat-loops on its own. If a chunk collapses into repeating the same sentence, it re-transcribes that span in smaller halves with fresh context. Results go in chunk_retries.json / transcription_warnings.json.
- The pipeline warns you before loading a model if you're short on RAM + VRAM, and tells you roughly what to close. The config GUI shows the same warning when you pick a model.
- AMD GPU support 9070XT confirmed working: ROCm torch install, Vulkan for the LLM stages, CPU whisper build where there's no NVIDIA card.

## Bug fixes

- Discovery was returning 0 candidates on qwen3.5. The model burned its whole token budget thinking and sent back an empty response. Everything uses /api/chat with think:false now.
- Judge crashed on big VODs because the final round sent all survivors in one call and blew past the context window. It does extra halving rounds now so no call is oversized.
- Markers landing at the wrong time in quiet VODs. Transcript blocks only closed on word count, so one block could stretch over a minute of near-silence and get stamped with the wrong time. Blocks close at 30 seconds max now.
- Whisper VAD was smearing timestamps across silence, so VAD is gone completely. Wall-clock decode only. Old VAD-era chunk SRTs get cleared automatically.
- Wall-clock decode had the opposite problem: silence turned into phantom words ("Thank you" chains). No-speech threshold tightened to 0.5, best-of dropped to 1 (it was wasting GPU on identical decodes), retry budget doubled to 2x, and exhausted-budget spans decode once instead of leaving holes in the transcript.
- ffmpeg sometimes failed to download from gyan.dev depending on region/ISP. The installer tries GitHub mirrors next.
- The installer used to accept truncated or wrong downloads as fine. It checks size and SHA-256 against the origin now and re-downloads on mismatch.
- The launcher kept re-downloading Python 3.12.10 even when it was already installed. It checks for an existing 3.12 first (py launcher, usual install paths).
- VOCAL_ISOLATION_DEVICE went missing from pipeline_config in a refactor and broke Demucs on single-track VODs. One-liner restored.
- Assorted step 3/4 fallout from the folder reorg: --out-dir handling, stale variable crash, stitched SRT landing in the wrong folder.
- Whisper choked on very long VOD paths. Chunk paths go through in verbatim form now.

## Changed

- Python 3.12 is required. The launcher installs 3.12.10 for you if you're on anything else below 3.12, if >3.12, it will work just fine.
- Step 1 fully prepares every audio chunk before Step 2 starts Whisper. No overlap between the two anymore, and chunks get reused on reruns.
- The Twitch path is explicit now: Wave64 extract, Demucs chunks, separation, concat, render, Whisper chunks, in that order.
- Discovery and judge prompts ask for transcript evidence instead of vibes. Discovery dedups on timestamp, judge sees the base score plus the emotion/hype boosts separately.
- 6_RunAllSteps.bat became Run_Pog_Engine.bat, and _mic_combined.w64 became _mic_demucs_combined.w64 since it was never the final mic track. Old names still work.

## Removed

- Silero VAD model entirely. Not downloaded, not checked, not patched, flag deleted.
- Some dead imports and helpers that weren't called by anything.
