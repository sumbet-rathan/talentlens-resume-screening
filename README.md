# 🎯 TalentLens: Explainable Resume Screening with RAG

Rank resumes against a job description with a transparent 0–100 ATS score, or chat with your resumes and get answers with sources.

TalentLens is built so every number can be explained. You can see which skills matched, which are missing, and how each part of the score was calculated.

**Built with:** Python · LangChain · ChromaDB · OpenAI · Streamlit

---

## ✨ Features

- **JD screening and ranking.** Paste a job description and get a ranked shortlist of candidates.
- **Explainable ATS score.** The score is a weighted average of five parts, shown per candidate.
- **Hybrid retrieval.** MMR + similarity search + BM25 keyword search, combined with an ensemble retriever.
- **Ask the resumes.** A chatbot that answers only from retrieved resume text, with cited sources and follow-up question support.
- **Smart ingestion.** Reads PDF and DOCX files, detects changes with SHA-256 hashes, and never creates duplicates.
- **Exports.** Results are saved to SQLite and can be downloaded as CSV.

---

## 🧠 How it works

```
Resumes (PDF/DOCX)
   │  clean → chunk (600 chars, 100 overlap) → embed
   ▼
Chroma vector store
   │
Job description ──► LLM extracts requirements (skills, years, education)
   │
   ▼
Hybrid retrieval (one query per required skill) → candidate pool
   │
   ▼
Per-candidate scoring → ranked shortlist + 2-sentence explanation
```

1. **Ingest.** Resumes are cleaned, split into chunks, embedded with `text-embedding-3-large` and stored in Chroma. Only new or changed files are re-indexed.
2. **Parse the JD.** An LLM returns structured data: role, required skills, nice-to-have skills, minimum years of experience and education.
3. **Retrieve.** Each required skill becomes its own search query, so more candidates get a chance to appear.
4. **Score.** Each candidate gets a 0–100 ATS score (see below).
5. **Explain.** The LLM writes two short sentences per candidate, using only the facts the code already calculated.

### ATS score

| Part | Weight | How it is calculated |
|------|--------|----------------------|
| Required skills | 40% | Share of required skills found in the resume |
| Experience | 20% | Candidate years ÷ required years (capped at 1) |
| Semantic match | 20% | Similarity between the JD and the best resume chunks |
| Nice-to-have skills | 10% | Share of preferred skills found |
| Education | 10% | Candidate level vs. required level |

- If the JD doesn't ask for a part (for example, no education line), that part is skipped and the remaining weights are re-scaled. Nobody is penalised for something the recruiter didn't request.
- Skill matching is done in plain code, not by the LLM. It handles aliases (`ML` = `Machine Learning`, `Postgres` = `PostgreSQL`) and symbols like `C++` and `.NET`.
- Fit bands: **Strong** (75+), **Partial** (50–74), **Weak** (below 50).

---

## 📁 Project structure

```
├── app/
│   ├── config.py        # All settings: models, chunking, weights
│   ├── ingestion.py     # Load, clean, chunk and index resumes
│   ├── vectorstore.py   # Chroma helpers
│   ├── retrievers.py    # Hybrid retriever (MMR + similarity + BM25)
│   ├── jd_parser.py     # LLM job description parser
│   ├── ats_scoring.py   # Explainable scoring engine
│   ├── matching.py      # Full JD → ranked candidates pipeline
│   ├── rag_chain.py     # Resume chatbot with citations
│   ├── storage.py       # SQLite + CSV export
│   ├── schemas.py       # Pydantic models
│   └── llm.py           # Shared ChatOpenAI instance
├── dashboard/
│   └── app.py           # Streamlit UI
├── data/
│   └── resumes/         # Put your PDF/DOCX resumes here
└── .env                 # Your OpenAI key (not committed)
```

> Adjust the `dashboard/` folder name to match where your `app.py` lives.

---

## 🚀 Getting started

### 1. Install

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install langchain langchain-openai langchain-chroma langchain-community \
            langchain-classic langchain-text-splitters rank_bm25 \
            pymupdf docx2txt streamlit pandas numpy pydantic python-dotenv altair
```

### 2. Add your API key

Create a `.env` file in the project root:

```
OPENAI_API_KEY=your-key-here
```

Optional settings: `LLM_MODEL` (default `gpt-4.1-mini`), `RESUME_DIR`, `LOG_LEVEL`.

### 3. Add resumes and index them

Put `.pdf` or `.docx` files in `data/resumes/`, then run:

```bash
python -m app.ingestion              # incremental
python -m app.ingestion --reindex    # wipe and rebuild
```

You can also click **Sync** or **Rebuild** in the dashboard sidebar.

### 4. Run the dashboard

```bash
streamlit run dashboard/app.py
```

---

## 🖥️ Command-line usage

```bash
# Rank resumes against a JD
python -m app.matching --file jd.txt --top 5

# Score every indexed resume, skipping the retrieval filter
python -m app.matching --file jd.txt --all

# Ask a question
python -m app.rag_chain "Who has experience with Power BI?"
```

---

## ⚙️ Configuration

Everything tunable lives in `app/config.py`:

- Chunk size and overlap
- Retriever settings and ensemble weights
- ATS score weights (validated to sum to 1.0)
- Fit band thresholds
- Chat memory length

---

## ⚠️ Limitations

- **Scanned (image-only) PDFs aren't supported.** There is no OCR, so these files are reported as failed during ingestion.
- Candidate names are guessed from the filename first, then the resume header. Check the detected names after indexing.
- The semantic score uses a calibration range tuned for `text-embedding-3-large`. Adjust `SEMANTIC_FLOOR` and `SEMANTIC_CEIL` in `ats_scoring.py` if you change the embedding model.
- ATS scores are a screening aid, not a hiring decision. A human should review every shortlist.

---

## 🛣️ Ideas for next steps

- OCR support for scanned resumes
- REST API (`POST /match`)
- Side-by-side candidate comparison
- Analytics dashboard (skill frequency, fit distribution)

---

## 📄 License

Add your license here (for example, MIT).
