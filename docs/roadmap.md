# Version roadmap

[← Project overview](../README.md) · [Backend setup](../backend/README.md)

### V1.0 — Conversational RAG Chat ✅

Stable chat-based Talking CV with:

- Laravel chat interface
- FastAPI AI backend
- RAG-based retrieval from the professional knowledge base
- LLM-generated grounded responses
- Multi-turn conversation history
- Context-dependent follow-up questions
- Suggested starter questions
- New conversation / conversation reset
- Input validation
- Graceful FastAPI failure handling
- Laravel and FastAPI automated tests
- Docker-based deployment
- Azure Container Apps deployment
- GitHub Actions CI/CD

### V2.0 — Agentic Talking CV ✅

Extending the conversational system with agentic capabilities:

- LangGraph-based agent orchestration
- LLM-driven dynamic tool selection
- Intent-based answer-model routing through a shared LiteLLM gateway, with environment-configured Azure and OpenAI models
- Live route and provider/model labels in the architecture chart, plus per-call LangSmith routing metadata
- Professional knowledge-base retrieval exposed as an agent tool
- GitHub integration through MCP
- README-only GitHub file access enforced through wrapper tools, with repository discovery and explicit access-limit guidance
- Multi-tool execution and answer synthesis
- Real-time agent activity streaming through FastAPI and Laravel
- Recruiter-visible tool activity such as CV search and GitHub retrieval

### Persistent vector retrieval ✅

- Qdrant Cloud storage with API-key authentication; optional Dockerised Qdrant with a persistent named volume
- Separate PDF ingestion command and query-time retrieval
- Azure OpenAI embeddings and cosine similarity search
- Source metadata, document version hashes, and stable chunk IDs
- Cloud ingestion and retrieval from the local application verified; Azure Container Apps deployment pending

### V3.0 — Voice Interaction 🚧

The next stage will extend the agentic conversational pipeline with voice support:

- Microphone input in the Laravel interface
- Speech-to-text (STT)
- Voice questions with conversation history
- Agentic tool selection through the existing LangGraph pipeline
- Text-to-speech (TTS)
- Synchronized text and audio responses
- Graceful fallback to text

```text
Recruiter
   │
   │ Speech
   ▼
Laravel UI
   │
   ▼
FastAPI
   │
   ├──► STT
   │       │
   │       ▼
   │   Transcribed Question
   │
   ▼
LangGraph Agent
   │
   ├──► CV / RAG Tool
   │
   ├──► MCP Tools
   │
   └──► LLM
           │
           ▼
      Final Response
           │
           ▼
          TTS
           │
           ▼
      Audio Response
           │
           ▼
      Laravel UI
       │       │
       ▼       ▼
      Text    Audio
           │
           ▼
       Recruiter
```
