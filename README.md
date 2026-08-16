# Multimodal RAG Full-Stack GenAI PoC

A full-stack Generative AI proof of concept that answers questions about complex
PDFs — including their tables and images, not just their text. Documents are
parsed into text, table, and image records, embedded with OpenAI, indexed in
Qdrant, and served through a ChatGPT-style Streamlit interface that returns
answers grounded in the retrieved source material.

## How it works

```text
PDF ──► parsing.py ──► ingestion.py ──► Qdrant ──► retriever.py ──► generation.py ──► Streamlit UI
        text/tables/     chunk +          vector     semantic          grounded
        images + OCR     embed            store      retrieval         answer
```

1. **Parse** — `ComplexPDFParser` extracts text, tables, embedded images, and
   full-page renders, running Tesseract OCR over images to recover text that
   would otherwise be lost.
2. **Ingest** — `MultimodalDocumentIngestion` chunks the parsed records, creates
   embeddings, and upserts them into a Qdrant collection with metadata.
3. **Retrieve** — `MultimodalQdrantRetriever` embeds the user's question and
   pulls the most relevant text, table, and image chunks.
4. **Generate** — `MultimodalRAGGenerator` sends the question plus retrieved
   context to an OpenAI chat model and returns an answer grounded in the
   sources.
5. **Serve** — the Streamlit app ties the pipeline together with document
   upload, chat, and source inspection.

## Prerequisites

- [uv](https://docs.astral.sh/uv/)
- Python 3.12
- Git
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) — required for
  image text extraction
- An OpenAI API key
- A Qdrant Cloud cluster (or a self-hosted Qdrant instance)

Install Tesseract:

```bash
# macOS
brew install tesseract

# Ubuntu / Debian
sudo apt install tesseract-ocr
```

On Windows, install from the
[UB Mannheim build](https://github.com/UB-Mannheim/tesseract/wiki) and set
`TESSERACT_PATH` in `.env` to the installed `tesseract.exe`.

## Setup

### 1. Clone the repository

```bash
git clone <repository-url>
cd mm-rag-full-stack-genAI-PoC
```

### 2. Create and activate the virtual environment

```bash
uv venv env --python 3.12
```

macOS or Linux:

```bash
source env/bin/activate
```

Windows Command Prompt:

```cmd
env\Scripts\activate.bat
```

Confirm the interpreter:

```bash
python --version   # should report Python 3.12.x
```

### 3. Install dependencies

```bash
uv pip install -r requirements.txt
```

### 4. Configure environment variables

Copy the template and fill in your own credentials:

```bash
cp .env.example .env
```

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `OPENAI_API_KEY` | yes | — | Embeddings and chat completion |
| `QDRANT_URL` | yes | — | Qdrant cluster endpoint |
| `QDRANT_API_KEY` | yes | — | Qdrant authentication (a JWT, starts with `eyJ`) |
| `QDRANT_COLLECTION_NAME` | yes | — | Target collection |
| `OPENAI_EMBEDDING_MODEL` | no | `text-embedding-3-large` | Embedding model |
| `OPENAI_EMBEDDING_DIMENSION` | no | `3072` | Must match the collection's vector size |
| `OPENAI_CHAT_MODEL` | no | `gpt-4.1-mini` | Answer generation model |
| `TESSERACT_PATH` | no | resolved from `PATH` | Tesseract binary location |

`.env` is excluded by `.gitignore` — never commit real keys.

## Running the application

```bash
streamlit run ui/app.py
```

If `streamlit` resolves to a different environment (a conda `base`, for
example), the app will fail with `ModuleNotFoundError` even though the packages
are installed. Bypass `PATH` entirely by invoking the venv interpreter
directly:

```bash
./env/bin/python -m streamlit run ui/app.py
```

Each pipeline stage can also be run standalone for debugging:

```bash
python src/parsing.py       # parse a PDF into text/table/image records
python src/ingestion.py     # embed and upsert records into Qdrant
python src/retriever.py     # run a retrieval query
python src/generation.py    # run an end-to-end generation
```

## Project structure

```text
mm-rag-full-stack-genAI-PoC/
|-- .env                    # Local credentials (not committed)
|-- .env.example            # Template for .env
|-- .gitignore
|-- README.md
|-- requirements.txt
|-- get_library_version.py  # Utility: report installed package versions
|-- data/
|   |-- *.pdf               # Source documents
|   |-- uploads/            # Runtime UI uploads (not committed)
|   `-- parsed_pdf_output/  # Generated parser artifacts (not committed)
|-- src/
|   |-- parsing.py          # ComplexPDFParser — text, tables, images, OCR
|   |-- ingestion.py        # MultimodalDocumentIngestion — chunk, embed, upsert
|   |-- retriever.py        # MultimodalQdrantRetriever — semantic retrieval
|   `-- generation.py       # MultimodalRAGGenerator — grounded answers
|-- ui/
|   `-- app.py              # Streamlit chat interface
|-- prompt_library/
|   `-- prompt.py           # Central prompt definitions
|-- logger/
|   `-- custom_logger.py    # Structured logging (structlog) to logs/
`-- exception/
    `-- custom_exception.py # Project exception types
```

`data/parsed_pdf_output/` and `data/uploads/` are generated at runtime and
excluded from version control. Parser output is reproducible from the source
PDFs via `src/parsing.py`.

## Troubleshooting

**`ModuleNotFoundError: No module named 'qdrant_client'`**
Streamlit is running from a different Python environment than the one holding
your dependencies. Run `which streamlit` — if it is not inside `env/`, use
`./env/bin/python -m streamlit run ui/app.py`.

**`QDRANT_URL is missing`**
`QDRANT_URL` is unset, misspelled, or its value contains stray whitespace.
Variable names are case-sensitive. Avoid quoting values in `.env`; a quoted
value keeps any padding inside the quotes.

**`UnexpectedResponse: 403 (Forbidden)`**
`QDRANT_API_KEY` is invalid. The key is a JWT and must begin with `eyJ` — check
for a character accidentally prepended or truncated during copy-paste.

**Vector dimension mismatch on ingestion or query**
`OPENAI_EMBEDDING_DIMENSION` must match both the embedding model and the vector
size the Qdrant collection was created with. Changing the embedding model means
recreating the collection.

**OCR returns nothing**
Confirm Tesseract is installed and reachable: `tesseract --version`. If it is
installed but not on `PATH`, set `TESSERACT_PATH` in `.env`.

## Development guidelines

- Use Python 3.12 and activate the virtual environment before running commands.
- Add new dependencies to `requirements.txt`.
- Keep credentials in `.env`; document any new variable in `.env.example`.
- Update this README whenever setup or run commands change.

## Status

The end-to-end pipeline is implemented and working: parsing, ingestion,
retrieval, generation, and the Streamlit interface. Evaluation, automated
tests, and a separate backend API are not yet built.
