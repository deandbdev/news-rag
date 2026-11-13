# News RAG — local semantic search and LLM analysis for Bulgarian news

A fully local retrieval-augmented generation (RAG) system for exploring thousands of news articles in Bulgarian. You ask a question, the system finds the relevant passages by meaning (not just keywords), shows you the sources, and lets an LLM analyse, interpret or forecast based only on the passages you select.

> Status: working personal project / prototype. Built to learn how every layer of a RAG pipeline fits together, so the components are simple and inspectable rather than hidden behind a framework.

<!-- TODO: add a screenshot of the search tab and a screenshot of the analysis result -->

## What it does

1. **Ingests** `.txt` articles (or pulls them from The Guardian API, translates them to Bulgarian with an LLM, and stores them).
2. **Indexes** them: sentence-aware chunking, quantized multilingual embeddings, named entities, full-text index.
3. **Searches** by meaning with a hybrid of vector search and full-text search.
4. **Lets you choose** which passages are relevant.
5. **Analyses** the chosen context with an LLM (summary, analysis, interpretation, forecast), including the extracted entities and a pre-computed structured analysis of each article.
6. **Logs and caches** every LLM request, so repeated questions are free and every answer is traceable.

## Scale of the current index

| Item | Count |
|---|---|
| Documents | 2,090 |
| Indexed chunks | 7,596 |
| Extracted named entities | 15,051 |
| Guardian articles translated and analysed | 1,856 |
| Logged LLM queries | 175 |

## Architecture

```
 .txt / Guardian API
        |
        v
  indexer.py  --> chunks + embeddings (ONNX) + NER (classla) --> SQLite (FTS5 + sqlite-vec)
        |
        v
  query_server.py (FastAPI, localhost:5001)  <---  PHP backend (REST, cache, history)  <---  Web UI (Alpine.js + DaisyUI)
                                                              |
                                                              v
                                                   LLM API (OpenAI / Anthropic / local Ollama)
```

## Stack

- **Embeddings:** AllMiniLM-L6-v2, quantized ONNX (~23 MB) via `onnxruntime`
- **NER and sentence splitting:** `classla` (Bulgarian)
- **Storage:** SQLite with FTS5 (full-text) and `sqlite-vec` (vector KNN search)
- **Indexer:** Python batch job
- **Query service:** FastAPI
- **Backend:** PHP (REST API, response cache, query history)
- **Frontend:** HTML, Alpine.js, DaisyUI
- **LLM:** any of OpenAI, Anthropic or a local Ollama model (configurable)

## Design decisions

- **Local first.** Embeddings, NER and the index run on your own machine. Only the final analysis step calls an LLM, and only with the passages you picked.
- **Human in the loop.** You select the context before the LLM sees it, which keeps answers grounded and cheap.
- **Everything is logged.** Each LLM call stores the question, mode, selected chunks, answer, sources, latency and model in `queries.sqlite`, and identical requests within a configurable window return the cached answer.
- **Pre-computed article analysis.** Each article gets a one-time structured analysis (summary, themes, facts versus attributed quotes) that is injected into the prompt for analysis modes.

## Quick start

Requires Python and PHP installed locally (tested on Windows).

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r python\requirements.txt

python python\download_model.py      # downloads the models (~500 MB first time)
python python\init_db.py             # creates the database
python python\gen_demo_news.py 10    # optional: generate 10 demo articles into data\
python python\indexer.py             # index them

python python\query_server.py        # start the query service (separate terminal)

copy config.example.php config.php   # add your LLM API key
php -S localhost:8000 -t public backend\index.php
```

Open http://localhost:8000.

## Optional: real news from The Guardian

Get a free developer key from the Guardian Open Platform, put it in `config.php`, then:

```powershell
python python\ingest_guardian.py --sections world,business --limit 50
```

The script downloads, translates to Bulgarian with your configured LLM, and indexes the articles. Only date, category, title and body are stored.

## Roadmap

<!-- Dean: replace with your real plans. Ideas: evaluation set to measure retrieval quality, reranking, entity graph view. -->

## Author

Dean Bouzekow — [LinkedIn](https://www.linkedin.com/in/dean-b-513b24162/)
