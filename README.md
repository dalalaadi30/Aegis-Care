# Aegis-Care

![CI Pipeline](https://github.com/dalalaadi30/Aegis-Care/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.12-blue.svg)

A dual-interface clinical AI assistant. Patients get health Q&A grounded in NIH MedlinePlus content, and clinicians get a note extraction tool that turns unstructured clinical notes into structured JSON validated against medical knowledge.

Aegis-Care shows how controlled, auditable, API-accessible health information delivery works, as opposed to a general-purpose chatbot: retrieval is restricted to a vetted knowledge base, a safety layer intercepts risky queries before they reach the LLM, and every query is logged for monitoring.

---

## Table of Contents

- [Architecture](#architecture)
- [Key Results](#key-results)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Setup](#setup)
- [Project Structure](#project-structure)
- [A/B Experiment Details](#ab-experiment-details)
- [Safety Layer](#safety-layer)
- [Authentication Note](#authentication-note)
- [Acknowledgements](#acknowledgements)
- [License](#license)
- [Author](#author)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Streamlit Frontend                    │
│         Patient Q&A Page  │  Clinician Extractor Page   │
└──────────────┬────────────────────────┬─────────────────┘
               │                        │
               ▼                        ▼
┌─────────────────────────────────────────────────────────┐
│                     FastAPI Backend                      │
│   /ask  /extract-note  /experiment-results  /drift-status│
└──────────────┬────────────────────────┬─────────────────┘
               │                        │
       ┌───────▼───────┐       ┌────────▼────────┐
       │  RAG Pipeline  │       │Note Extractor   │
       │                │       │                 │
       │ Safety Check   │       │ pdfplumber/text │
       │ Cache Lookup   │       │ Groq LLM        │
       │ Dense/Hybrid   │       │ NIH Validation  │
       │ Retrieval      │       │ JSON Output     │
       │ Groq LLM       │       └─────────────────┘
       └───────┬────────┘
               │
┌──────────────▼──────────────────────────────────────────┐
│              PostgreSQL + pgvector                       │
│                                                          │
│  knowledge_chunks  │  query_logs  │  response_cache      │
│  experiment_runs   │                                     │
└─────────────────────────────────────────────────────────┘
```

---

## Key Results

| Metric | Value |
|--------|-------|
| Knowledge base | 1,318 chunks across 46 NIH MedlinePlus topics |
| Dense retrieval mean relevance | 0.7598 |
| Hybrid retrieval mean relevance | 0.7788 |
| A/B p-value | 0.298 (no statistically significant difference) |
| Semantic cache speedup | ~22x faster on cache hits (1.57s → 0.07s) |
| Safety layer | 3 categories: emergency, dosage, diagnosis |
| Unit tests | 8 passing, enforced on every push via GitHub Actions |
| API endpoints | 6: /health, /ask, /extract-note, /experiment-results, /drift-status, /knowledge-topics |

---

## Features

**Patient Interface**
- Conversational health Q&A grounded exclusively in NIH MedlinePlus content
- Hybrid retrieval combining dense vector search (pgvector) and BM25 keyword search via Reciprocal Rank Fusion
- Three-layer safety system: emergency pattern interception, out-of-scope blocking (dosage, diagnosis), confidence gating
- Every query logged with confidence score, strategy, and safety metadata for drift monitoring
- Semantic caching: semantically similar queries are served from cache in ~100ms

**Clinician Interface**
- Accepts pasted text or PDF upload of clinical notes
- Extracts structured fields: chief complaint, diagnoses, medications (with new medication flagging), follow-up actions, warning signs, dietary instructions
- Validates extracted diagnoses against the NIH knowledge base and flags conditions not found
- Downloads structured output as JSON

**Experimentation**
- A/B framework comparing dense vs hybrid retrieval on 20 health questions with ground truth answers
- Automated relevance scoring via cosine similarity between generated answers and ground truth
- Paired t-test with p-value and 95% confidence interval
- All runs logged to PostgreSQL and tracked via MLflow

**Monitoring**
- Evidently AI drift reports comparing query embedding distributions across time windows
- Query log analytics exposed via the `/drift-status` endpoint

**Infrastructure**
- GitHub Actions CI/CD: pytest + flake8 on every push to main
- Docker + docker-compose: database spins up with one command
- FastAPI backend with auto-generated docs at `/docs`

---

## Tech Stack

| Category | Tools |
|----------|-------|
| Embeddings | sentence-transformers (all-MiniLM-L6-v2) |
| Vector database | pgvector on PostgreSQL 16 |
| Keyword search | BM25 (rank-bm25) |
| LLM | Llama 3.3 70B via Groq API |
| Backend | FastAPI |
| Frontend | Streamlit |
| Monitoring | Evidently AI |
| Experiment tracking | MLflow |
| PDF parsing | pdfplumber |
| CI/CD | GitHub Actions |
| Containerization | Docker + docker-compose |

---

## Setup

**Prerequisites:** Python 3.12, Docker, Git

**1. Clone and install**
```bash
git clone https://github.com/dalalaadi30/Aegis-Care.git
cd Aegis-Care
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS / Linux
pip install -r requirements.txt
```

**2. Configure environment**
```bash
cp .env.example .env
# Edit .env and add your GROQ_API_KEY
# Get a free key at console.groq.com
```

**3. Start the database**
```bash
docker compose up -d
python src/database.py
```

**4. Ingest the knowledge base**
```bash
python src/ingest_medlineplus.py
```

**5. Run the application**
```bash
# Terminal 1 — API
cd src && uvicorn api:app --reload --port 8000

# Terminal 2 — UI
streamlit run app.py
```

Open `http://localhost:8501` in your browser. API docs are at `http://localhost:8000/docs`.

**Run the tests**
```bash
pytest tests/
```

---

## Project Structure

```
Aegis-Care/
├── src/
│   ├── database.py           # Schema setup: all PostgreSQL tables
│   ├── ingest_medlineplus.py # NIH MedlinePlus ingestion pipeline
│   ├── retrieval.py          # Dense + hybrid BM25/RRF retrieval
│   ├── llm.py                # Groq LLM integration
│   ├── safety.py             # Three-layer safety system
│   ├── cache.py              # Semantic caching layer
│   ├── pipeline.py           # End-to-end query pipeline
│   ├── extractor.py          # Clinical note extraction
│   ├── experiment.py         # A/B experimentation framework
│   ├── monitoring.py         # Evidently AI drift detection
│   └── api.py                # FastAPI application
├── pages/
│   ├── patient.py            # Patient Q&A Streamlit page
│   └── clinician.py          # Clinician extractor Streamlit page
├── tests/
│   └── test_pipeline.py      # 8 unit tests: safety + retrieval
├── notebooks/
│   └── exploration.ipynb     # A/B results, confidence distribution, KB coverage
├── data/
│   └── sample_notes/         # Sample clinical notes for testing
├── .github/workflows/
│   └── ci.yml                # GitHub Actions: test + lint on every push
├── app.py                    # Streamlit home page
├── conftest.py               # Pytest configuration
├── docker-compose.yml        # PostgreSQL + pgvector container
├── monitoring_report.html    # Sample Evidently AI drift report
├── .env.example              # Environment variable template
├── requirements.txt
└── LICENSE
```

---

## A/B Experiment Details

Two retrieval strategies were evaluated on 20 health questions with ground truth answers sourced from the NIH knowledge base:

- **Strategy A (Dense):** embed the query, find the top-5 chunks by cosine similarity in pgvector
- **Strategy B (Hybrid):** combine dense similarity and BM25 keyword scores via Reciprocal Rank Fusion (k=60)

Answer quality was scored automatically by computing cosine similarity between the generated answer and the ground truth answer, using the same sentence-transformers model.

A paired t-test found no statistically significant difference between the strategies (p=0.298, 95% CI [-0.018, 0.056]). Question-level analysis shows hybrid outperforms on treatment/prevention queries while dense performs better on symptom queries, which suggests the best strategy depends on the type of query.

See `notebooks/exploration.ipynb` for the full visualizations.

---

## Safety Layer

The system intercepts three categories of queries before they reach the LLM:

- **Emergency:** chest pain, stroke symptoms, breathing difficulty, overdose, suicidal ideation. Returns a hardcoded emergency-services directive.
- **Dosage:** medication dosage requests. Returns a pharmacist referral.
- **Diagnosis:** "do I have X" requests. Returns a healthcare provider referral.

A confidence gate additionally blocks responses when retrieval similarity falls below 0.45. The system refuses to answer rather than generate a low-confidence response.

> **Disclaimer:** Aegis-Care is an educational and demonstration project. It is not a medical device and does not provide medical advice, diagnosis, or treatment. Always consult a qualified healthcare professional.

---

## Authentication Note

The current version is a single application with two pages. In a production deployment, role-based authentication would separate patient and clinician access:

- Patients authenticate via email/password and access patient Q&A only
- Clinicians authenticate via institutional SSO and access both interfaces
- JWT tokens with role claims stored in the existing PostgreSQL session infrastructure
- All queries attributed to authenticated users for a full audit trail

---

## Acknowledgements

- This project is based on [CareCompanion](https://github.com/Mohit-03Sharma/CareCompanion) by **Mohit Sharma**, used under the terms of the MIT License. The original copyright notice is retained in the [LICENSE](LICENSE) file.
- Health content is sourced from [NIH MedlinePlus](https://medlineplus.gov/).
- LLM inference is provided by [Groq](https://groq.com/).

---

## License

Distributed under the MIT License. See [LICENSE](LICENSE) for details.

---

## Author

**Aaditya**
Maharshi Dayanand University

GitHub: [@dalalaadi30](https://github.com/dalalaadi30)
