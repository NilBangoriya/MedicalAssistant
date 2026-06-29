# 🩺 MediBot — AI Medical Document Assistant

A Retrieval-Augmented Generation (RAG) chatbot that lets users upload medical PDFs and ask natural-language questions about them. The system retrieves the most relevant passages from the uploaded documents and uses an LLM to generate grounded, context-only answers — it explicitly refuses to answer when the documents don't contain the information, rather than guessing.

**Live demo:** https://medicalassistancee.streamlit.app · **API:** https://medicalassistant-94tv.onrender.com

https://medicalassistant-94tv.onrender.com
---

## Overview

MediBot is a full-stack RAG application split into two independently deployable services:

- **`server/`** — a FastAPI backend that handles PDF ingestion, embedding, vector storage, and question answering.
- **`client/`** — a Streamlit chat UI that uploads documents and talks to the backend over REST.

The core pipeline: a user uploads a PDF → it's chunked and embedded → the embeddings are stored in Pinecone → when a question is asked, the most relevant chunks are retrieved → an LLM (Llama 3.1 via Groq) generates an answer constrained to that retrieved context.

## Architecture

```
┌─────────────┐        HTTP (multipart/form)         ┌──────--────────────┐
│  Streamlit  │ ───────────────────────────────────► │   FastAPI server   │
│   Client    │ ◄─────────────────────────────────── │                    │
└─────────────┘            JSON response             └────────┬───────────┘
                                                                │
                                       ┌────────────────────────┼─────────────────────────┐
                                       ▼                        ▼                         ▼
                              PyPDFLoader + Text          Google Gemini              Groq LLM
                              Splitter (chunking)         Embeddings                 (llama-3.1-8b-instant)
                                       │                        │                         ▲
                                       └────────────► Pinecone (vector store) ─────────────┘
                                                       similarity search (top-k)
```

## Tech Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI, Uvicorn |
| LLM orchestration | LangChain (core, community, classic) |
| LLM provider | Groq API — `llama-3.1-8b-instant` |
| Embeddings | Google Generative AI — `gemini-embedding-001` |
| Vector database | Pinecone (serverless) |
| PDF parsing | PyPDF / PyPDFLoader |
| Frontend | Streamlit |
| HTTP client | Requests |
| Config / secrets | python-dotenv |
| Logging | Python `logging` |
| CI/CD | GitHub Actions |
| Deployment | Render |

## Features

- 📄 Upload one or more PDF medical documents from the sidebar
- 💬 Chat interface with conversation history kept in session
- 🔍 Context-grounded answers — the model is instructed to say "I'm sorry, but I couldn't find relevant information" rather than hallucinate
- 📥 Downloadable chat transcript
- 🌐 Decoupled client/server architecture, deployable independently

## Project Structure

```
MedicalAssistant/
├── server/                     # FastAPI backend
│   ├── main.py                 # App entrypoint, CORS + router registration
│   ├── logger.py                # Logging setup
│   ├── routes/
│   │   ├── upload_pdfs.py      # POST /upload_pdfs/
│   │   └── ask_question.py     # POST /ask/
│   ├── modules/
│   │   ├── load_vectorstore.py # Chunk, embed, upsert to Pinecone
│   │   ├── llm.py               # Prompt template + RetrievalQA chain
│   │   ├── query_handlers.py    # Runs the chain, shapes the response
│   │   └── pdf_handlers.py      # File save helpers
│   ├── middlewares/
│   │   └── exception_handlers.py
│   ├── Dockerfile
│   └── requirements.txt
├── client/                     # Streamlit frontend
│   ├── app.py
│   ├── config.py
│   ├── components/
│   │   ├── upload.py
│   │   ├── chatUI.py
│   │   └── history_download.py
│   ├── utils/
│   │   └── api.py
│   ├── Dockerfile
│   └── requirements.txt
├── docker-compose.yml
├── .github/workflows/ci.yml
└── README.md
```

## Getting Started

### Prerequisites

- Python 3.13+
- Docker & Docker Compose (optional, but recommended)
- API keys for: [Groq](https://console.groq.com), [Google AI Studio](https://aistudio.google.com) (Gemini), [Pinecone](https://www.pinecone.io)

### Environment Variables

Create a `.env` file inside `server/`:

```env
GROQ_API_KEY=your_groq_key
GOOGLE_API_KEY=your_google_key
PINECONE_API_KEY=your_pinecone_key
PINECONE_INDEX_NAME=medicalindex-v2
```

### Option A — Run locally with Python

```bash
# Backend
cd server
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# Frontend (in a separate terminal)
cd client
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
streamlit run app.py
```

### Option B — Run with Docker Compose

```bash
docker-compose up --build
```

This builds and starts both services:
- Backend → http://localhost:8000
- Frontend → http://localhost:8501

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/upload_pdfs/` | Multipart file upload — chunks, embeds, and stores PDF(s) in Pinecone |
| `POST` | `/ask/` | Form field `question` — retrieves relevant chunks and returns an LLM-generated answer |

## CI/CD

Every push and pull request to `main` triggers a GitHub Actions workflow that:
1. Installs server and client dependencies
2. Compiles all Python files to catch syntax errors
3. Builds the server and client Docker images

See [`.github/workflows/ci.yml`](.github/workflows/ci.yml).

## Roadmap

- [ ] Unit tests for chunking, retrieval, and the query chain
- [ ] Source citations surfaced in the UI
- [ ] Auth / rate limiting on the API
- [ ] Persistent chat history (currently session-only)

## License

MIT

## Author

**Nil Bangoriya** — AI/ML Engineer
