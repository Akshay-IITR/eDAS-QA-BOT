# eDAS-QA-BOT

```
# Folder Structure
docassist/
├── main.py
├── requirements.txt
├── tests/test_all.py
├── .env
├── app/
|   ├── config.py
│   ├── core/
│   │   ├── document_processor.py
│   │   ├── embeddings.py
│   │   ├── vector_store.py
│   │   ├── intent_detector.py
│   │   ├── emotion_detector.py
│   │   └── response_generator.py
│   └── api/
│       ├── upload.py
│       └── query.py
└── voicebot/
    └── core/
        └── intent_router.py
```

## Setup

```bash
# 1. Clone and enter project
cd docassist

# 2. Create virtual environment
conda create --name edas
conda activate edas

# 3. Install dependencies
pip install -r requirements.txt

# 4. Install Ollama & Pull Required Models
Download and install Ollama
```
irm https://ollama.com/install.ps1 | iex
```

Pull models locally:
```
ollama pull llama3.1
ollama pull nomic-embed-text
conda install -c conda-forge langchain-ollama
```

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
