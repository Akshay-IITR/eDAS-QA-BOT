# eDAS-QA-BOT

```
docassist/
├── main.py                        # FastAPI app entry point
├── requirements.txt
├── .env.example
├── app/
│   ├── config.py                  # Settings via env vars
│   ├── models/schemas.py          # Pydantic request/response models
│   ├── api/
│   │   ├── upload.py              # POST /api/v1/documents/upload
│   │   └── query.py               # POST /api/v1/query
│   └── core/
│       ├── document_processor.py  # Extract → chunk → assign IDs
│       ├── embeddings.py          # OpenAI text-embedding-3-small
│       ├── vector_store.py        # FAISS in-memory store
│       ├── intent_detector.py     # Rule-based + LLM fallback
│       ├── emotion_detector.py    # Lexicon-based + LLM fallback
│       └── response_generator.py # Prompt builder + LLM call
└── voicebot/
    ├── core/
    │   ├── session_store.py       # In-memory session management
    │   ├── intent_router.py       # 3-intent classifier
    │   └── conversation_engine.py # Multi-turn slot filling logic
    └── api/
        └── chat.py                # POST /api/v1/chat, /voice
```

## Setup

```bash
# 1. Clone and enter project
cd docassist

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
cp .env.example .env
# Edit .env: set OPENAI_API_KEY=sk-your-key

# 5. Run server
uvicorn main:app --reload --port 8000
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/documents/upload | Upload & ingest document |
| POST | /api/v1/query | Query with intent + emotion |
| POST | /api/v1/chat | VoiceBot multi-turn chat |
| POST | /api/v1/voice | VoiceBot voice (STT simulated) |
| GET  | /health | Health check |
| GET  | /docs | Swagger UI |

## Running Tests

```bash
# Run all unit tests (no API key needed)
pytest tests/ -v

# Run with coverage
pytest tests/ -v --cov=app --cov=voicebot
```

## Tools & LLMs Used

| Component | Tool |
|-----------|------|
| LLM | OpenAI GPT-4o |
| Classification | GPT-4o-mini (cheaper) |
| Embeddings | text-embedding-3-small |
| Vector Store | FAISS (in-memory) |
| Framework | FastAPI + Pydantic v2 |
| Document parsing | PyPDF2, python-docx, BeautifulSoup |
| Tokenization | tiktoken |

## Assumptions

1. Single-process deployment (FAISS is in-memory; use Pinecone for multi-process).
2. OpenAI API key required for embeddings + LLM. All other logic is rule-based.
3. Vector store is not persisted to disk (add pickle/faiss.write_index for persistence).
4. VoiceBot STT/TTS is mocked; integrate Whisper + ElevenLabs for production.
5. User isolation is enforced at the vector store level (per user_id index).

## Production Scaling (Section D)

- **Multiple users**: Each user_id gets an isolated FAISS index. For true multi-tenancy, use Pinecone namespaces or Weaviate multi-tenancy.
- **Reduce latency**: Cache embeddings for repeated queries (Redis). Run intent+emotion detection in parallel (asyncio.gather). Use gpt-4o-mini for classification.
- **Reduce LLM cost**: Rule-based fast path handles ~70% of intent/emotion without LLM. Cache responses for identical queries. Use streaming for UX without waiting.

## Edge Case Handling (Section E)

| Edge Case | Strategy |
|-----------|----------|
| Wrong chunks retrieved | Score threshold 0.25 filters low-relevance results. LLM instructed to say "not enough info" rather than hallucinate. |
| Incorrect intent | Confidence threshold 0.6. Below threshold → LLM fallback. User can rephrase. |
| Slow response | Async FastAPI. Intent+emotion run in parallel. Streaming response optional. |
| No documents uploaded | Returns fallback_used=True. LLM explicitly told it has no context. |
| Context switch in VoiceBot | Detected via keyword patterns + confidence delta. Resets slots cleanly. |
