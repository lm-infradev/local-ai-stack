# Local AI Stack — Ollama + Open-WebUI + Tika RAG Pipeline

A fully private, self-hosted AI stack running locally on Apple Silicon (M1/M2/M3), featuring document ingestion, OCR extraction, vector embeddings, and retrieval-augmented generation (RAG). No data leaves your machine.

## Overview

This repository documents the setup, configuration, and troubleshooting of a production-ready local AI pipeline built on open-source components. The stack enables you to chat with your own documents privately — no cloud APIs, no subscription fees, no data sharing.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        macOS Host (M1)                       │
│                                                             │
│   ┌─────────────┐        ┌──────────────────────────────┐  │
│   │   Ollama    │        │         Docker Desktop        │  │
│   │  (native)   │        │                              │  │
│   │             │        │  ┌────────────┐  ┌────────┐  │  │
│   │ • Chat LLM  │◄───────┼──│ Open-WebUI │  │  Tika  │  │  │
│   │ • Embeddings│        │  │  :3000     │  │  :9998 │  │  │
│   │  :11434     │        │  └─────┬──────┘  └───┬────┘  │  │
│   └─────────────┘        │        │              │       │  │
│                          │        ▼              │       │  │
│                          │  ┌──────────┐         │       │  │
│                          │  │ ChromaDB │◄────────┘       │  │
│                          │  │(embedded)│                 │  │
│                          │  └──────────┘                 │  │
│                          └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Data flow:**
1. User uploads a document (PDF, Word, etc.) to Open-WebUI
2. Open-WebUI sends the file to Tika for text extraction (with OCR for scanned docs)
3. Extracted text is chunked and sent to Ollama for embedding
4. Embeddings are stored in ChromaDB (vector database)
5. User asks a question — relevant chunks are retrieved from ChromaDB
6. Retrieved context + question are sent to the chat LLM via Ollama
7. LLM generates a grounded answer with source citations

---

## Stack Components

| Component | Version | Role | Deployment |
|---|---|---|---|
| [Ollama](https://ollama.com) | Latest | LLM inference + embeddings | Native macOS |
| [Open-WebUI](https://github.com/open-webui/open-webui) | Latest | Web interface + RAG orchestration | Docker |
| [Apache Tika](https://tika.apache.org) | 3.x (latest-full) | Document text extraction + OCR | Docker |
| [ChromaDB](https://www.trychroma.com) | Embedded | Vector database | Inside Open-WebUI |

---

## Models Used

| Model | Purpose | Size | Pull Command |
|---|---|---|---|
| `qwen3:8b` | Chat / RAG responses | 5.9 GB | `ollama pull qwen3:8b` |
| `mxbai-embed-large` | Document embeddings | 670 MB | `ollama pull mxbai-embed-large` |

**Why these models:**
- `qwen3:8b` — strong reasoning, excellent instruction following, built-in thinking mode for complex queries
- `mxbai-embed-large` — 768-dimension embeddings, higher retrieval accuracy than smaller alternatives. Outperforms `nomic-embed-text` on most benchmarks while remaining practical on 16GB unified memory

---

## Hardware

Tested and running on:
- **Apple MacBook Pro M1** — 16GB unified memory
- macOS with Docker Desktop
- Ollama running natively (uses Metal GPU acceleration — 100% GPU inference)

**Memory allocation:**
- Docker Desktop: 4GB RAM + 6GB swap
- Tika Java heap: 512MB–4GB (configured via `JAVA_OPTS`)
- Ollama: remainder of unified memory pool

---

## Installation

### Prerequisites

- [Ollama](https://ollama.com/download) installed and running on macOS
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed
- Docker Desktop configured with at least 4GB RAM *(Settings → Resources)*

### 1. Pull Ollama models

```bash
ollama pull qwen3:8b
ollama pull mxbai-embed-large
```

Verify models are available:
```bash
ollama list
```

### 2. Start Tika container

The `latest-full` image includes Tesseract OCR for scanned document support.

```bash
docker run -d \
  --name tika \
  -p 9998:9998 \
  -e JAVA_OPTS="-Xms512m -Xmx4g" \
  apache/tika:latest-full
```

**Why `-Xmx4g`:** The default Java heap is too small for large PDFs. Without this setting, Tika drops connections mid-processing on documents over ~50 pages, producing a `Connection aborted` error in Open-WebUI.

Verify Tika is running:
```bash
curl http://localhost:9998/tika
# Expected: "This is Tika Server X.X"
```

### 3. Start Open-WebUI container

```bash
docker run -d \
  --name open-webui \
  -p 3000:8080 \
  -v open-webui-data:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  ghcr.io/open-webui/open-webui:main
```

Open-WebUI will be available at: `http://localhost:3000`

---

## Configuration

Once Open-WebUI is running, configure it via **Admin Panel → Settings → Documents**:

| Setting | Value | Notes |
|---|---|---|
| Tika Server URL | `http://host.docker.internal:9998` | Do NOT use `localhost` — see troubleshooting |
| Embedding Backend | Ollama | Uses Metal GPU acceleration on M1 |
| Embedding Model | `mxbai-embed-large` | 768-dimension embeddings |
| Ollama URL | `http://host.docker.internal:11434` | Same for chat and embeddings |

> **Critical:** Always use `host.docker.internal` instead of `localhost` when referencing services on the Mac host from inside Docker containers. `localhost` inside a container refers to the container itself, not the host machine.

---

## Usage

### Adding documents to a knowledge base

1. Go to **Workspace → Knowledge**
2. Create a new knowledge base
3. Upload documents (PDF, Word, text, etc.)
4. Monitor progress via logs (see Monitoring section)

### Chatting with your documents

1. Start a new chat
2. Click the **+** icon next to the message box
3. Select your knowledge base
4. Ask questions — responses will include source citations from your documents

### Disabling Qwen3 thinking mode (faster responses)

Qwen3 has a built-in reasoning/thinking mode that improves accuracy but increases response time. To disable for faster answers, add to your message:

```
/no_think
```

---

## Monitoring

Watch processing in real time using two terminal tabs:

**Open-WebUI logs:**
```bash
docker logs -f open-webui
```

**Tika logs:**
```bash
docker logs -f tika
```

**Successful document ingestion looks like:**
```
file.content_type: application/pdf
generating embeddings for file-xxxx
embeddings generated 1108 for 1108 items
adding to collection xxxx
added 1108 items to collection xxxx
```

**Check active Ollama models:**
```bash
ollama ps
```

**Monitor Docker resource usage:**
```bash
docker stats
```

---

## Troubleshooting

### `No connection adapters were found for 'localhost:9998/tika/text'`

**Cause:** The Tika URL is missing the `http://` scheme prefix.

**Fix:** In Admin Panel → Settings → Documents, ensure the Tika URL is:
```
http://host.docker.internal:9998
```
Not `localhost:9998` or `tika:9998`.

---

### `Failed to resolve 'tika' — Name or service not known`

**Cause:** Open-WebUI has `tika` saved as the hostname, which only works if both containers share a Docker network.

**Fix:** Change the Tika URL to use the host gateway:
```
http://host.docker.internal:9998
```

Alternatively, put both containers on the same Docker network and use `http://tika:9998`.

---

### `Connection aborted — Remote end closed connection without response`

**Cause:** Tika is running out of Java heap memory while processing large PDFs. The default heap is very low (often 256MB).

**Fix:** Recreate the Tika container with increased Java heap:
```bash
docker stop tika && docker rm tika

docker run -d \
  --name tika \
  -p 9998:9998 \
  -e JAVA_OPTS="-Xms512m -Xmx4g" \
  apache/tika:latest-full
```

---

### `Collection expecting embedding with dimension of 384, got 768`

**Cause:** The ChromaDB collection was created with a previous embedding model (`nomic-embed-text` = 384 dimensions). After switching to `mxbai-embed-large` (768 dimensions), the dimensions are incompatible. ChromaDB cannot mix embedding sizes within a collection.

**Fix:** Delete the existing knowledge base in Open-WebUI and create a new one. The new collection will be initialised with the correct 768-dimension size.

> **Important:** Any time you change embedding models, all existing knowledge bases must be deleted and recreated. Documents need to be re-ingested to generate new embeddings with the new model.

---

### Scanned PDFs return empty text

**Cause:** The standard `apache/tika` Docker image does not include Tesseract OCR. Text extraction from scanned documents silently returns nothing.

**Fix:** Use the full image which includes Tesseract:
```bash
# Wrong — no OCR support
docker pull apache/tika

# Correct — includes Tesseract OCR
docker pull apache/tika:latest-full
```

To verify Tesseract is available in your running container:
```bash
docker logs tika | grep -i tesseract
# Expected: "Tesseract is installed and is being invoked"
```

---

### Slow response times

Expected performance on M1 16GB with `qwen3:8b`:

| Condition | Approximate Time |
|---|---|
| Simple query, no RAG | 20–40 seconds |
| RAG query with 6 source chunks | 60–120 seconds |
| First query (model loading) | Add 10–20 seconds |

Both Ollama models run at 100% GPU via Metal acceleration on Apple Silicon. Response time is normal for local inference at this model size — no cloud API is involved.

---

## File Persistence

By default, Open-WebUI stores uploaded files and the vector database inside a named Docker volume. To map this to a local Mac folder for easier backup:

```bash
# Replace the volume flag in the docker run command:
-v ~/open-webui-data:/app/backend/data
```

This ensures your knowledge base survives container deletion or recreation.

---

## Performance Notes

- **Unified memory:** On Apple Silicon, RAM is shared between CPU, GPU, and Neural Engine. Ollama, Docker, and other applications all draw from the same pool
- **Recommended minimum:** 16GB unified memory for this stack alongside normal desktop usage
- **Ollama model limit:** Set `OLLAMA_MAX_LOADED_MODELS=1` to prevent multiple models competing for memory simultaneously
- **Docker resources:** Allocate 4GB RAM and 6GB swap to Docker Desktop for comfortable processing of large documents

---

## Tested Document Types

| Format | Text-based | Scanned/OCR | Notes |
|---|---|---|---|
| PDF (digital) | ✅ | — | Fast extraction via PDFBox |
| PDF (scanned) | — | ✅ | Requires `latest-full` image + Tesseract |
| Word (.docx) | ✅ | — | Full support |
| Plain text | ✅ | — | Instant |

---

## License

This repository documents a personal homelab setup using open-source tools. All referenced software retains its original respective licences.

---

## Author

Built and documented by an IT infrastructure engineer with 20+ years of enterprise experience, currently exploring local AI deployment on Apple Silicon.
