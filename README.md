# BM25 RAG Party-Planning Example

This project is a small Retrieval-Augmented Generation (RAG) example. It gives an LLM an on-demand search tool over a short collection of superhero-party ideas, then asks the agent to use those ideas to answer a planning request.

The implementation is in [`bm25_Rag.py`](./bm25_Rag.py).

## What the example does

When you run the script, it:

1. Selects a model provider and creates a model client. The example currently supports Ollama through its OpenAI-compatible API.
2. Wraps five sample party-planning ideas in LangChain `Document` objects. Each document has text and a `source` metadata value.
3. Splits the documents into chunks using `RecursiveCharacterTextSplitter`.
4. Creates a BM25 search retriever that can return up to five relevant chunks.
5. Exposes that retriever to a `smolagents` `CodeAgent` as the `party_planning_retriever` tool.
6. Asks the agent for luxury superhero-party ideas and prints its answer.

The LLM does not receive the whole knowledge base automatically. The agent can call the retriever tool; the tool returns matching text, which the agent then uses as context to formulate its response.

## Requirements

- Python 3.10 or newer
- [Ollama](https://ollama.com/) installed and running locally
- An Ollama model downloaded to your computer
- The Python packages listed below

No separate vector database or embedding model is needed. BM25 searches text by term relevance rather than by embedding similarity.

## Installation

### 1. Create and activate a virtual environment

From this project directory:

```bash
python3 -m venv .venv
```

On macOS or Linux:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

### 2. Install Python dependencies

```bash
python -m pip install --upgrade pip
python -m pip install "smolagents[openai]" langchain-community langchain-text-splitters rank-bm25
```

`rank-bm25` is the BM25 implementation used by LangChain's `BM25Retriever`. The `openai` extra supplies the OpenAI-compatible client used by `OpenAIServerModel`; Ollama itself serves that API locally.

## Set up the local LLM with Ollama

Install Ollama, then download a model. For example:

```bash
ollama pull llama3.2
```

Make sure the Ollama service is running. It commonly listens at:

```text
http://localhost:11434
```

If needed, start the service in another terminal:

```bash
ollama serve
```

The script defaults to model ID `llama:latest` and API base `http://localhost:11434/v1`. Ollama model names must match models installed locally, so if you pulled `llama3.2`, set `OLLAMA_MODEL` accordingly.

### Configure the script

The script reads these environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `MODEL_PROVIDER` | `ollama` | Selects the model provider. Only `ollama` is implemented. |
| `OLLAMA_MODEL` | `llama:latest` | Ollama model name/tag to use. |
| `OLLAMA_BASE_URL` | `http://localhost:11434/v1` | Ollama's OpenAI-compatible API base URL. |

For example, on macOS or Linux:

```bash
export MODEL_PROVIDER=ollama
export OLLAMA_MODEL=llama3.2
export OLLAMA_BASE_URL=http://localhost:11434/v1
```

In Windows PowerShell:

```powershell
$env:MODEL_PROVIDER = "ollama"
$env:OLLAMA_MODEL = "llama3.2"
$env:OLLAMA_BASE_URL = "http://localhost:11434/v1"
```

The code passes the literal API key `"ollama"` because the OpenAI-compatible client expects a key value. For a local Ollama server, it is a placeholder, not a secret or an account credential.

## Run it

With the virtual environment active and Ollama running:

```bash
python bm25_Rag.py
```

The agent's request is currently written directly in the script. It asks for entertainment, catering, and decoration suggestions for a luxury superhero-themed party. The answer is printed to the terminal. The model's wording and the agent's tool-call choices may vary between runs.

## Concepts demonstrated

### Retrieval-Augmented Generation (RAG)

RAG combines two operations:

- **Retrieval:** Find relevant information in a knowledge source.
- **Generation:** Give that information to a language model so it can produce a natural-language answer.

Here, the party ideas are the knowledge source, BM25 is the retrieval method, and the agent/model generates the final response. This is a tiny in-memory example; it does not load external files or persist an index.

### BM25 keyword retrieval

BM25 is a lexical ranking algorithm. It scores documents based on query-term matches, with adjustments for how often a term appears and how long the document is. It is useful for finding text that shares words with a query.

BM25 is **not semantic or embedding search**: it does not inherently understand that different words have similar meanings. The tool's description says “semantic search,” but the implementation uses BM25, so keyword-based search is the accurate description.

### Documents, metadata, and chunking

Each LangChain `Document` contains `page_content` and metadata such as its source label. The splitter is configured with a maximum chunk size of 500 characters and an overlap of 50 characters. Overlap can preserve context when a long document is divided across chunk boundaries. These sample ideas are short, so chunking may leave each idea as a single chunk.

The retriever ranks the chunks and is configured with `k=5`, meaning it returns at most five results. The tool formats the result using each chunk's text. Although source metadata is attached to the documents, the tool does not include those labels in its returned text.

### Agent and tool use 

`CodeAgent` is the orchestration layer. It receives a task, can decide to call its registered tool, and uses the tool's return value while preparing its answer. The retriever itself does not generate an answer; it only returns matching text.

### Local inference

Ollama runs the selected model on the local machine and exposes an API compatible with the OpenAI client protocol. In this example, `OpenAIServerModel` connects to that local endpoint. The script sets temperature to `0.5` and the maximum output length to `2096` tokens.

Local inference can keep model requests on your machine and avoid a hosted model API, but the model must be downloaded and your machine needs enough resources to run it. The model's quality and speed depend on its size and your hardware.

## Limitations and useful extensions

- The knowledge base is hard-coded and contains only five examples.
- BM25 relies on term overlap; try a vector/embedding retriever if meaning-based matches are important.
- The tool returns text only, not source labels, even though documents have source metadata.
- The agent is free to decide whether to call the tool. A tool-enabled agent is not the same as a workflow that forces retrieval on every request.
- To experiment, add party ideas to `party_ideas`, change the query in `agent.run(...)`, or adjust the chunk size and retriever's `k` value.

## Troubleshooting

- **Connection refused:** Confirm Ollama is installed, running, and reachable at `OLLAMA_BASE_URL`.
- **Model not found:** Check the local model names with `ollama list`, then set `OLLAMA_MODEL` to an installed name.
- **Unsupported provider:** `create_model()` currently accepts only `MODEL_PROVIDER=ollama`.
- **Missing BM25 dependency:** Install `rank-bm25` in the same Python environment used to run the script.
