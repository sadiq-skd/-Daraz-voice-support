# Daraz-style Order Support Voice Bot

An AI voice-support agent that handles **order tracking** and **return requests**,
similar to how a Daraz call center agent would — built as a portfolio project
mapping directly to an *Associate AI Engineer* job description (FastAPI,
WebSockets, Voice AI, prompt engineering, LLM integration, production thinking).

## Architecture

```
Browser (mic) --Web Speech API (STT)--> text
      |
      v  WebSocket (/ws/voice)
FastAPI backend
      |
      +--> Gemini call #1: intent extraction (strict JSON)
      |        -> intent: track_order / return_order / greeting / other
      |        -> order_id, reason extracted from natural speech
      |
      +--> order_db.py: real lookup (mock DB, swap for Daraz API in prod)
      |
      +--> Gemini call #2: natural language reply generation
      |
      v
Browser <--WebSocket-- bot reply text --SpeechSynthesis (TTS)--> spoken reply
```

Why two Gemini calls instead of one? It's a deliberate production pattern:
- Call 1 is constrained to strict JSON → reliable to parse, low hallucination risk.
- Order data is fetched from **our own database**, never invented by the model.
- Call 2 only turns *verified* data into natural speech → the model can't
  hallucinate order status or make up a return that didn't happen.

## Tools used

| Concern            | Tool                                   |
|---------------------|-----------------------------------------|
| Backend / API        | FastAPI                                |
| Real-time transport   | WebSockets                             |
| LLM / conversation    | Google Gemini API (`gemini-2.0-flash`) |
| Speech-to-text        | Browser Web Speech API (free, no key)  |
| Text-to-speech        | Browser SpeechSynthesis API (free)     |
| Order data            | Mock in-memory DB (`order_db.py`)      |

> Voice layer is intentionally kept on free browser APIs so the whole demo
> runs with just a Gemini key. To go production-grade for real phone calls,
> swap the browser mic/TTS for a telephony-integrated voice stack — e.g.
> **Vapi** or **Twilio + ElevenLabs** — the FastAPI/WebSocket/Gemini logic
> underneath stays the same, only the audio I/O layer changes.

## Setup

```bash
cd daraz-voice-support
python -m venv venv && source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt

# create backend/.env
echo "GEMINI_API_KEY=your_key_here" > backend/.env

cd backend
uvicorn main:app --reload
```

Open **http://localhost:8000** in Chrome (Web Speech API works best there).
Click the mic and try things like:
- "Assalam o alaikum, mera order DRZ1002 track karna hai"
- "I want to return order DRZ1003, it's damaged"
- "Mujhe DRZ1001 kab tak milega?"

Test order IDs available in the mock DB: `DRZ1001`, `DRZ1002`, `DRZ1003`.

## Mapping to the job description

- **FastAPI + Python** → entire backend (`main.py`)
- **WebSockets for real-time systems** → `/ws/voice` endpoint, streaming
  turn-by-turn conversation
- **Voice AI & conversational pipelines** → STT → LLM → TTS pipeline
- **Prompt engineering across multiple LLMs** → `prompts.py` is model-agnostic
  by design (swap `gemini_service.py` for an OpenAI/Claude wrapper with the
  same prompt templates)
- **Integrating AI services into scalable applications** → clean separation:
  `order_db.py` (data layer) / `gemini_service.py` (AI layer) / `main.py`
  (orchestration) — easy to swap any layer independently
- **System-level / production thinking** → session state per call, defensive
  JSON parsing, real data never hallucinated, clear path to swap mock DB →
  real Daraz API and browser voice → Vapi/Twilio+ElevenLabs

## Next steps to make it fully production-grade

1. Replace mock `order_db.py` with a real database (Postgres) or Daraz API call.
2. Add authentication (verify caller identity via order ID + phone number).
3. Swap browser voice I/O for a telephony provider (Vapi / Twilio) to handle
   real phone calls instead of browser mic.
4. Add logging/analytics per call (intent accuracy, resolution rate).
5. Add a fallback-to-human-agent path when the model is unsure.
