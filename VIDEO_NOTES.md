# Video research notes

**Video:** [How I Built an End-to-End Local Voice Agent, and Made It Fast](https://www.youtube.com/watch?v=xbedfuqYQYA)  
**Use:** paraphrased implementation notes for the reproduction guide; this is not a transcript.

## Stack described

- Qwen-family mixture-of-experts local LLM for agent work.
- Whisper for speech recognition.
- Breeze TTS 2 for speech generation.
- Silero VAD for hands-free turn detection.
- A Pythagoras agent harness with browser, terminal, and canvas work panels.
- Target described: fitting the working stack on one GPU with about 12 GB VRAM.

## Latency changes described

- Avoid waiting for the full LLM response: stream text to TTS in short phrases.
- Synthesize upcoming audio while the current phrase is playing, using a queue to reduce gaps.
- Begin transcription on rolling audio windows while the person is still speaking, then wait about a second after speech ends before responding.
- Disable model “thinking” on the first response after the person speaks to reduce first-response delay.
- Quantize Breeze TTS 2 to Q8 through audio.cpp; the video reports 2–3× faster generation and about 4 GB of VRAM freed.
- Keep voice responses short, announce tool work, and play a short acknowledgement during longer tool calls.

## LLM memory and tool throughput changes

- Reduce context length to make room for the voice model.
- Increase llama.cpp micro-batch size after freeing VRAM, and move more expert layers to GPU where headroom allows.
- Persist conversation cache to disk so later turns do not repeat expensive prompt prefill.

## Results reported by the creator

- The video gives a rough improvement from an 8-second turn delay to 2–3 seconds on its setup.
- It notes that long TTS phrases can still empty the playback queue and create a pause.
- Treat all numbers as machine- and revision-specific measurements.

## Transcript availability

On 2026-10-27, YouTube's player reported that subtitles/closed captions were unavailable. The linked transcript extractor returned a processing error, and the browser's YouTube transcript export returned “No transcript is available for this YouTube video.” The transcript is therefore not included; these notes paraphrase an indexed summary and are not verbatim captions.

