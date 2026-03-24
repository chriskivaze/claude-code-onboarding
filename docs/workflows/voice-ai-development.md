# Voice AI Development Workflow

Phases for building Gemini voice features — real-time Live API and TTS synthesis — with LangGraph and Google ADK.

**Skill:** `voice-ai-development`
**Models:** `gemini-live-2.5-flash-native-audio` (Live), `gemini-2.5-pro-tts-preview` / `gemini-2.5-flash-tts-preview` (TTS)
**SDK:** `google-genai` (NEVER `google-generativeai` — deprecated)

---

## Phase 0: Decide Modality and Framework

Before writing a line of code, answer these two questions:

**Modality:**
| Need | Choose |
|------|--------|
| Real-time voice (<600ms, barge-in) | Gemini Live API (WebSocket) |
| Batch/one-shot speech synthesis | Gemini TTS models |
| Both (voice assistant) | Live API for input, TTS for output |

**Framework:**
| Need | Choose |
|------|--------|
| Session memory + Google Cloud deploy | Google ADK |
| Fine-grained state control, LangChain tools | LangGraph |
| Unsure | Start with LangGraph (simpler to debug) |

Load the skill: `voice-ai-development`
Load the decision guide: `reference/voice-adk-integration.md` section 8 (ADK vs LangGraph).

---

## Phase 1: Environment Setup

```bash
# Install SDK
uv add google-genai

# For LangGraph pipeline
uv add langgraph langchain-google-genai

# For ADK pipeline
uv add google-adk

# For audio file I/O (optional — WAV/MP3 export)
uv add pydub  # requires ffmpeg: brew install ffmpeg

# Set API key (get from https://aistudio.google.com)
export GOOGLE_API_KEY=your_key_here

# Verify
python -c "import google.genai; print('SDK OK:', google.genai.__version__)"
```

**Gate:** `GOOGLE_API_KEY` is set and SDK imports without error.

---

## Phase 2: Live API — Real-time Voice Streaming

Reference: `.claude/skills/voice-ai-development/reference/gemini-live-api.md`

Steps:
1. Copy WebSocket connection setup (section 1)
2. Implement bidirectional audio streaming (section 2)
3. Add barge-in handling if needed (section 3)
4. Wrap in FastAPI WebSocket endpoint (section 5)

**Validation:**
```bash
# Start server
uvicorn main:app --reload --port 8000

# Test WebSocket (requires wscat: npm install -g wscat)
wscat -c ws://localhost:8000/ws/voice
# Send: raw PCM bytes
# Send: END (0x454e44) to trigger response
# Expect: PCM audio bytes back
```

**Gate:** WebSocket connects, sends audio, receives audio bytes (not silence, not empty).

---

## Phase 3: TTS — Text-to-Speech Synthesis

Reference: `.claude/skills/voice-ai-development/reference/gemini-tts.md`

Steps:
1. Choose model (`gemini-2.5-pro-tts-preview` or `gemini-2.5-flash-tts-preview`) — see section 1 decision guide
2. Implement single-speaker synthesis (section 2)
3. Add multi-speaker support if needed (section 3)
4. Add language switching if needed (section 4)
5. Wrap in FastAPI endpoint (section 6)

**Validation:**
```bash
curl -X POST http://localhost:8000/tts \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello, this is a test.", "model": "gemini-2.5-flash-tts-preview", "voice": "Aoede"}' \
  --output test.wav

# Verify WAV is playable
file test.wav   # should be RIFF (little-endian) data, WAVE audio
```

**Gate:** WAV file is non-empty and plays correctly in a media player.

---

## Phase 4: LangGraph Voice Pipeline (if using LangGraph)

Reference: `.claude/skills/voice-ai-development/reference/voice-langgraph-integration.md`

Steps:
1. Define `VoiceAgentState` TypedDict (section 1)
2. Implement `voice_input_node` — transcription (section 2)
3. Implement `llm_response_node` — agent logic (section 3)
4. Implement `voice_output_node` — TTS synthesis (section 4)
5. Assemble graph: `voice_input -> llm_response -> voice_output -> END` (section 5)
6. Add FastAPI WebSocket endpoint (section 6)

**Validation:**
```python
import asyncio
from voice_graph import voice_agent, VoiceAgentState

fake_audio = b"\x00\x01" * 8000  # 16KB of fake PCM

async def test():
    state = VoiceAgentState(
        audio_input=fake_audio,
        transcript="",
        response_text="",
        audio_output=b"",
        error=None,
        session_id="test",
        turn_count=0,
    )
    result = await voice_agent.ainvoke(state)
    print("Transcript:", result["transcript"])
    print("Response:", result["response_text"])
    print("Audio bytes:", len(result["audio_output"]))
    assert result["audio_output"], "No audio output"

asyncio.run(test())
```

**Gate:** All three nodes execute, state flows end-to-end, audio_output is non-empty.

---

## Phase 5: ADK Voice Pipeline (if using ADK)

Reference: `.claude/skills/voice-ai-development/reference/voice-adk-integration.md`

Steps:
1. Implement FunctionTools: `transcribe_audio`, `synthesize_audio` (section 1)
2. Configure three LlmAgents: TranscriptionAgent, ResponseAgent, SynthesisAgent (section 2)
3. Assemble SequentialAgent pipeline (section 3)
4. Set up ADK Runner + InMemorySessionService (section 4)
5. Add FastAPI endpoints (section 4)

**Validation:**
```bash
# Test HTTP turn endpoint
curl -X POST http://localhost:8000/adk/voice/turn \
  -H "Content-Type: application/json" \
  -d "{\"audio_hex\": \"$(python -c 'print(b\"\\x00\\x01\"*8000, end=\"\")' | xxd -p | tr -d '\n')\"}"
```

**Gate:** Response JSON contains `transcript`, `response_text`, and `audio_hex` (non-empty).

---

## Phase 6: Testing

Reference: `voice-langgraph-integration.md` section 8 (LangGraph tests), `voice-adk-integration.md` section 7 (ADK error table)

Mandatory tests:
- [ ] Happy path: audio in -> transcript -> response -> audio out
- [ ] Empty audio input raises `ValueError` (not silently returns empty)
- [ ] Barge-in: interrupt signal discards buffered audio
- [ ] Rate limit: exponential backoff fires on 429 (mock `ResourceExhausted`)
- [ ] WebSocket disconnect: session cleans up without hanging

```bash
# Run tests
pytest tests/test_voice_agent.py -v

# Test coverage for voice module
pytest tests/test_voice_agent.py --cov=voice --cov-report=term-missing
```

**Gate:** All tests pass. No test uses `google-generativeai` import.

---

## Phase 7: Code Review and Security

Dispatch reviewers after implementation:
```
security-reviewer  — API key handling, no keys in code, audio data sanitization
code-reviewer      — WebSocket lifecycle, error paths, no silent failures
```

Security checklist:
- [ ] `GOOGLE_API_KEY` read from env var only — never hardcoded, never logged
- [ ] Audio bytes are not logged (can contain PII)
- [ ] WebSocket connections have a timeout (no unbounded open connections)
- [ ] Content policy blocks surface as explicit errors — not silent empty audio
- [ ] FastAPI endpoints validate input before passing to API

---

## Phase 8: Deploy

Reference: `adk-deploy-guide` skill (for ADK), `gcp-cloud-run` skill (for LangGraph/FastAPI)

For ADK:
- Use `adk deploy` to Agent Engine for managed session handling
- Or deploy as Cloud Run service with `InMemorySessionService` for stateless single-instance

For LangGraph:
- Deploy as standard FastAPI service on Cloud Run
- WebSocket support: set `--concurrency` to 1 per instance if using session-local state

```bash
# LangGraph — Cloud Run deploy
gcloud run deploy voice-ai-service \
  --source . \
  --port 8000 \
  --set-env-vars GOOGLE_API_KEY=projects/PROJECT/secrets/GOOGLE_API_KEY/versions/latest \
  --allow-unauthenticated

# ADK — Agent Engine deploy (requires adk-deploy-guide skill)
adk deploy agent-engine \
  --project PROJECT_ID \
  --region us-central1 \
  --agent voice_pipeline
```

**Gate:** Health check endpoint responds 200. WebSocket connection test succeeds against deployed URL.

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Using `google-generativeai` SDK | `ModuleNotFoundError` or deprecated API errors | Replace with `google-genai` |
| Polling instead of WebSocket | High latency, missed barge-in | Use WebSocket — see `gemini-live-api.md` section 5 |
| Sending text to Live API | Empty audio response | Live API expects `AudioChunk` or text with `end_of_turn=True` |
| Blocking event loop with sync TTS | FastAPI hangs under load | Wrap sync calls in `run_in_executor` |
| Empty audio on content block | Silent failure, no user feedback | Check `inline_data.data` is non-empty, raise if not |
| No barge-in handling | Robot continues after user speaks | Check `server_content.interrupted` flag |
| Logging audio bytes | PII leak in logs | Never log raw audio — log only byte count |
