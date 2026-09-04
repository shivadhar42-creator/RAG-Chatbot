# RAG-Chatbot
# 📚 RAG Chatbot — Chat With Your Own Documents

A retrieval-augmented generation (RAG) chatbot that answers questions grounded
in your own PDFs, notes, or text files — instead of hallucinating from general
training data. Ask it something outside the documents and it'll tell you it
doesn't know, rather than making something up.

**Stack:** Python · ChromaDB (vector store) · sentence-transformers (local embeddings) · Claude (generation) · Streamlit (UI)

## Why this exists

Most "chat with your PDF" tutorials stop at a Jupyter notebook. This is a
small but complete pipeline: document loading → chunking → embedding →
vector search → grounded generation → both a CLI and a web UI, with
tests on the parts that don't need an API key to verify.

## How it works

1. **Ingest** — PDFs/text files are loaded and split into overlapping chunks
   (overlap prevents facts from getting severed at a chunk boundary).
2. **Embed** — each chunk is embedded locally using `sentence-transformers`,
   so indexing your documents never costs API credits or leaves your machine.
3. **Store** — embeddings are persisted in a local ChromaDB collection.
4. **Retrieve** — a question is embedded and matched against the closest
   chunks by vector similarity.
5. **Generate** — the top matches are stuffed into a prompt and sent to
   Claude, which is instructed to answer *only* from that context and to
   cite its sources.

```
 PDFs/TXT/MD ──▶ chunk ──▶ embed (local) ──▶ ChromaDB
                                                  │
                              question ──▶ embed ─┼─▶ top-k chunks ──▶ Claude ──▶ grounded answer
```

## Setup

```bash
git clone https://github.com/yourusername/rag-chatbot.git
cd rag-chatbot
python -m venv venv && source venv/bin/activate   # or your preferred env manager
pip install -r requirements.txt

cp .env.example .env
# then add your Anthropic API key to .env
```

## Usage

**1. Add documents**

Drop PDFs, `.txt`, or `.md` files into the `data/` folder.

**2. Index them**

```bash
python -m src.cli ingest --data-dir data
```

**3. Chat**

CLI:
```bash
python -m src.cli chat
```

Or the web UI:
```bash
streamlit run app.py
```

## Project structure

```
rag-chatbot/
├── app.py                 # Streamlit web UI
├── src/
│   ├── ingest.py           # loading + chunking documents
│   ├── vectorstore.py      # ChromaDB wrapper, local embeddings
│   ├── rag.py               # retrieval + Claude generation
│   └── cli.py                # command-line interface
├── tests/
│   └── test_ingest.py      # chunking logic tests (no API key required)
├── data/                    # put your documents here
└── requirements.txt
```

## Design decisions worth knowing about

- **Local embeddings, remote generation.** Embedding every chunk through an
  API gets slow and expensive at scale. Embeddings run locally via
  `sentence-transformers`; only the final answer generation calls out to
  Claude.
- **Overlapping chunks.** A naive split can sever a fact exactly at a chunk
  boundary and lose it during retrieval. A 150-character overlap (configurable)
  mitigates this.
- **Explicit "I don't know."** The system prompt instructs Claude to say so
  when the retrieved context doesn't answer the question, instead of
  filling the gap with general knowledge — the whole point of RAG is
  grounding, not just prompt-stuffing.
- **Idempotent ingestion.** Chunk IDs are deterministic (`filename::chunk_index`),
  so re-running ingestion on the same files updates rather than duplicates them.

## Possible extensions

- Swap ChromaDB for a hosted vector DB (Pinecone, Weaviate) for multi-user deployments
- Add re-ranking (e.g. a cross-encoder) on top of the initial vector search
- Support `.docx` and `.html` ingestion
- Stream responses token-by-token in the Streamlit UI
- Add conversation memory so follow-up questions can refer to earlier turns

## Running tests

```bash
pytest tests/ -v
```

## License

MIT
