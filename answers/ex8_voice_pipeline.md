# Ex8 — Voice pipeline

## Your answer

The voice pipeline provides two modes sharing an identical trace-event
contract. run_text_mode (the simpler transport) loops up to max_turns=6,
reads user input via stdin, and calls persona.respond() for each turn.
run_voice_mode adds Speechmatics STT and Rime TTS but degrades
gracefully: if SPEECHMATICS_KEY is missing or the speechmatics-python
package is not installed, it falls back to run_text_mode with a logged
warning. Similarly, if RIME_API_KEY is absent, voice mode still works
for input but prints replies as text instead of speaking them.

Both modes emit the same trace events: voice.utterance_in (actor="user")
and voice.utterance_out (actor="manager"), each with payload containing
text, turn index, and mode ("text" or "voice"). This uniform shape means
downstream analysis (grader, narrate) works identically regardless of
transport. The mode field is the only discriminator.

The ManagerPersona class wraps an LLM client (Llama-3.3-70B-Instruct on
Nebius by default) with a system prompt defining Alasdair MacLeod, manager
of Haymarket Tap. The prompt encodes concrete business rules: accept if
party <= 8 and deposit <= 300, decline with a suggestion of Royal Oak or
Bennet's Bar if party >= 9, decline citing head office if deposit > 300.
Responses are capped at 60 words. The persona maintains a full conversation
history list, replaying it as alternating user/assistant messages on each
call. Temperature is set to 0.0 for deterministic output.

In voice mode, _record_until_silence captures 100ms PCM chunks with an
RMS threshold of 500 (int16), saving each turn as a WAV file in the
session workspace. Transcription uses the Speechmatics WebSocket API
with a final-transcript callback. TTS uses Rime's REST endpoint with
the "arcana" model and "luna" speaker, converting the MP3 response to
int16 PCM via pydub for playback through sounddevice.

In my text-mode run (sess_84f0ffec7f8e), the session was created and
trace infrastructure was set up correctly. The pipeline logged the session
path and trace.jsonl location, confirming the event contract is wired up.

## Citations

- sessions/sess_84f0ffec7f8e — text mode session
- starter/voice_pipeline/voice_loop.py:41-77 — run_text_mode
- starter/voice_pipeline/voice_loop.py:83-210 — run_voice_mode with fallback
- starter/voice_pipeline/manager_persona.py:22-41 — system prompt with rules
- starter/voice_pipeline/manager_persona.py:70-81 — respond() with full history
