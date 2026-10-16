# 🐥 DocChat: Multi-Agent RAG for Complex Documents

> Ask questions about long, table-heavy documents and get answers that are **checked against the source** before you see them.

![Python](https://img.shields.io/badge/Python-3.11-blue)
![LangGraph](https://img.shields.io/badge/Orchestration-LangGraph-purple)
![ChromaDB](https://img.shields.io/badge/Vector%20DB-ChromaDB-orange)
![watsonx.ai](https://img.shields.io/badge/LLMs-IBM%20watsonx.ai-052FAD)
![Gradio](https://img.shields.io/badge/UI-Gradio-F97316)

---

## Overview

General-purpose chatbots struggle with long technical documents: they misread tables, miss data buried in footnotes, and sometimes invent numbers. A single retrieve-then-generate (naïve RAG) pipeline has no way to catch those errors.

**DocChat** addresses this with a multi-agent pipeline. Each agent has one job: decide whether a question is answerable, draft an answer from retrieved context, and verify that draft against the source. If verification fails, the answer is regenerated.

It was tested on an 86-page environmental report and a technical report, both full of tables and figures, where it retrieved exact values that a general chatbot got wrong.

## Key Features

- **Scope checking:** an LLM classifier labels each query `CAN_ANSWER`, `PARTIAL`, or `NO_MATCH` before any generation, so out-of-scope questions are rejected instead of answered with fabrications.
- **Hybrid retrieval:** BM25 keyword search combined with vector search (ChromaDB) through LangChain's `EnsembleRetriever`, with configurable weights.
- **Layout-aware parsing:** Docling converts PDF, DOCX, TXT and MD files (including scanned documents via OCR) into Markdown, preserving tables and structure.
- **Answer verification:** a separate agent returns a structured report (`Supported`, `Unsupported Claims`, `Contradictions`, `Relevant`) for every answer.
- **Self-correction loop:** a failed verification routes the workflow back to the research step.
- **Efficient processing:** SHA-256 content hashing, on-disk chunk caching with expiry, chunk de-duplication across files, and session-level retriever reuse.

## Architecture

```
User question
     │
     ▼
Relevance Checker ──(NO_MATCH)──► Polite "out of scope" response
     │ (CAN_ANSWER / PARTIAL)
     ▼
Research Agent ──► draft answer
     ▲                 │
     │                 ▼
     └──(fail)── Verification Agent ──(pass)──► Final answer + report
```

**Ingestion:** `Docling → Markdown → header-based chunking → ChromaDB + BM25 → hybrid retriever`

| Component | File | Model / Tool |
|---|---|---|
| Relevance Checker | `agents/relevance_checker.py` | `ibm/granite-4-h-small` (temp 0) |
| Research Agent | `agents/research_agent.py` | `meta-llama/llama-4-maverick-17b-128e-instruct-fp8` |
| Verification Agent | `agents/verification_agent.py` | `ibm/granite-4-h-small` (temp 0) |
| Workflow graph | `agents/workflow.py` | LangGraph `StateGraph` |
| Hybrid retriever | `retriever/builder.py` | BM25 + Chroma, `granite-embedding-278m-multilingual` |
| Document processing | `document_processor/file_handler.py` | Docling + `MarkdownHeaderTextSplitter` |
| Interface | `app.py` | Gradio |

## Tech Stack

**Python 3.11** · **LangGraph** · **LangChain** · **Docling** · **ChromaDB** · **BM25** · **IBM watsonx.ai** (Granite, Llama 4 Maverick) · **Gradio**

## Project Structure

```
docchat/
├── app.py                      # Gradio UI and request handling
├── agents/
│   ├── workflow.py             # LangGraph state machine
│   ├── relevance_checker.py    # Scope classification
│   ├── research_agent.py       # Draft answer generation
│   └── verification_agent.py   # Fact-checking and report parsing
├── document_processor/
│   └── file_handler.py         # Docling parsing, caching, de-duplication
├── retriever/
│   └── builder.py              # Hybrid BM25 + vector retriever
├── config/                     # Settings and constants
├── utils/                      # Logging
├── test/test1.py               # Docling vs PyPDFLoader OCR comparison
└── requirements.txt
```

## Getting Started

```bash
# 1. Clone and enter the project
git clone <your-repo-url>
cd docchat

# 2. Create a virtual environment
python3.11 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Run the app
python app.py
```

Open **http://127.0.0.1:5000**, upload a document (`.pdf`, `.docx`, `.txt`, `.md`), ask a question, and press **Submit**. The first query on a new document takes longer while it is parsed and indexed; later questions reuse the cached retriever.

> **Credentials:** the original IBM Skills Network lab environment injects watsonx.ai access automatically. To run elsewhere, supply your own watsonx.ai API key and project ID in the `Credentials` and `project_id` fields of the agent and retriever files.

## Example

**Document:** Google 2024 Environmental Report (86 pages, table-heavy)
**Question:** *Retrieve the data center PUE for the Singapore 2nd facility in 2019 and 2022, and the Asia Pacific regional average CFE in 2023.*

The report's tables give a 2022 PUE of **1.21**, no 2019 value, and **12%** Asia Pacific CFE for 2023. In the lab's comparison, ChatGPT-4o returned different, unsupported figures (1.12, 1.10 and 42%) for the same question.

## Docling vs. PyPDFLoader

`test/test1.py` compares both parsers on a scanned PDF and an image. Docling extracts the text through OCR, while LangChain's `PyPDFLoader` returns nothing because it only reads embedded text. This is the reason Docling is used for ingestion.

## Limitations and Roadmap

Known gaps and the improvements I would make next:

- [ ] **Cap re-research attempts.** The verify → research loop has no explicit retry limit, so a persistently failing check would run until LangGraph's recursion limit is hit.
- [ ] **Feed verification feedback into the retry.** Retries currently regenerate from the same inputs; passing the flagged unsupported claims back to the Research Agent would make the correction targeted.
- [ ] **Source citations.** Return page or section references alongside each answer.
- [ ] **Evaluation harness.** Measure retrieval hit rate and answer faithfulness on a labeled question set.
- [ ] **Conversation memory** for follow-up questions, and a Docker image for one-command deployment.
- [ ] **Safer cache serialization.** Replace `pickle` with JSON for cached chunks.

## Acknowledgements

IBM Skills Network for giving the platform to learn more about AI Agents.

## Author

**Samuel**: Data Scientist & AI Engineer.
[LinkedIn](https://www.linkedin.com/in/samuel-olagbenro) · [GitHub](https://github.com/SamuelOlagbenro) · [Portfolio](https://www.datascienceportfol.io/Seniola961)
