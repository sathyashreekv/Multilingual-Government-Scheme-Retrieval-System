# 🇮🇳 Multilingual Government Scheme Search Engine

> Search Indian government schemes in **English, Hindi (हिंदी), and Kannada (ಕನ್ನಡ)** — by typing or by voice.

---

## 📌 Overview

This project is an intelligent, multilingual retrieval system built on top of the [MyScheme](https://www.myscheme.gov.in/) dataset. Citizens can discover relevant government welfare schemes using natural language queries in any of three languages — even with spelling mistakes or spoken audio input.

The system combines three complementary retrieval strategies into a single ranked result:

| Strategy | What it handles |
|---|---|
| **Semantic Search** (FAISS + Sentence Transformers) | Meaning-based matching across language barriers |
| **Keyword Search** (BM25 Okapi) | Exact and partial term matching |
| **Fuzzy Name Matching** (RapidFuzz) | Misspellings, transliterations (e.g. `"pradan"` → `"Pradhan"`) |

Voice queries are transcribed via **OpenAI Whisper** with a browser-based microphone recorder — no external API keys required.

---

## 🏗️ Architecture

```
User Query (Text or Voice)
        │
        ▼
┌───────────────────┐
│  Whisper STT      │  ← Voice only (medium model, 16kHz mono WAV)
│  (if audio)       │
└────────┬──────────┘
         │ transcribed text
         ▼
┌─────────────────────────────────────────────┐
│              Language Detection              │
│  Kannada U+0C80–0CFF / Hindi U+0900–097F    │
└────────┬────────────────────────────────────┘
         │
         ├──────────────────────────────────┐
         ▼                                  ▼
┌─────────────────┐              ┌──────────────────────┐
│  Fuzzy Match    │              │  Semantic Search      │
│  (RapidFuzz)    │              │  FAISS + MiniLM       │
│  token_sort_    │              │  (cosine similarity)  │
│  ratio ≥ 40     │              └──────────┬───────────┘
└────────┬────────┘                         │
         │                       ┌──────────┴───────────┐
         │                       │   BM25 Keyword Search │
         │                       │   ( BM25)        │
         │                       └──────────┬───────────┘
         │                                  │
         └──────────────┬───────────────────┘
                        ▼
              ┌──────────────────┐
              │  Merge & Re-rank │
              │  Semantic: 70%   │
              │  BM25:     30%   │
              └────────┬─────────┘
                       ▼
              Top-K Ranked Results
              (scheme name in EN/HI/KN,
               tags, category, URL, snippet)
```

---

## ✨ Features

- **Trilingual search** — one query pipeline handles English, Hindi, and Kannada without switching modes
- **Hybrid retrieval** — tunable semantic/keyword weighting (`SEMANTIC_WEIGHT`, `BM25_WEIGHT`)
- **Fuzzy tolerance** — catches transliteration variants and typos automatically
- **Voice input** — browser mic → ffmpeg → Whisper → search, entirely in-notebook
- **Persistent indexes** — embeddings and FAISS index are cached to disk; no re-encoding on restart
- **Multilingual `search_text`** — each row merges EN + HI + KN fields so a query in any language can match any scheme
- **Retrieval evaluation** — built-in Hit Rate @K, Mean Precision @K, and MRR metrics with per-language breakdown

---

## 🛠️ Tech Stack

| Component | Library / Tool |
|---|---|
| Embedding model | `paraphrase-multilingual-MiniLM-L12-v2` (Sentence Transformers) |
| Vector search | `faiss-cpu` — `IndexFlatIP` (inner product = cosine on L2-normalised vectors) |
| Keyword search | `rank_bm25` — BM25Okapi |
| Fuzzy matching | `rapidfuzz` — `token_sort_ratio` |
| Speech-to-text | `openai-whisper` — `medium` model |
| Audio conversion | `ffmpeg` (16kHz, mono, PCM WAV) |
| Data handling | `pandas`, `numpy` |
| Notebook env | Google Colab |

---

## 🚀 Getting Started

### 1. Clone & Open in Colab

```bash
git clone https://github.com/sathyashreekv/Multilingual-Government-Scheme-Retrieval-System.git
```

Open `nlp_rag.ipynb` in [Google Colab](https://colab.research.google.com/).

### 2. Upload your dataset

Place your CSV at `/content/finalized_data (1) (1).csv` or update `CSV_PATH` in the config cell.

The CSV should include these columns (extra columns are ignored):

| Column | Description |
|---|---|
| `scheme_name` | Scheme name in English |
| `scheme_name_hindi` | Scheme name in Hindi |
| `scheme_name_kannada` | Scheme name in Kannada |
| `tags`, `tags_hi`, `tags_kn` | Tags in all three languages |
| `schemeCategory` | Category (e.g., Agriculture, Health) |
| `level` | Central / State |
| `details`, `details_hi` | Scheme details (truncated for indexing) |
| `eligibility` | Eligibility criteria |
| `benefits` | Scheme benefits |
| `slug` | URL slug for `myscheme.gov.in` |

### 3. Run cells top-to-bottom

All dependencies are installed in the first cell:

```bash
pip install sentence-transformers faiss-cpu numpy pandas rank_bm25 openai-whisper rapidfuzz
apt-get install -y ffmpeg
```

On first run, embeddings are generated and saved to:
- `scheme_embeddings.npy`
- `scheme_faiss.index`

Subsequent runs load from disk — much faster.

---

## 🔍 Usage

### Text Search

```python
display_hybrid_results("pradhan mantri awas yojana", top_k=5)
display_hybrid_results("गरीब छात्रों के लिए छात्रवृत्ति", top_k=5)
display_hybrid_results("ಮಹಿಳೆಯರಿಗೆ ಸ್ವಯಂ ಉದ್ಯೋಗ ಸಾಲ", top_k=5)
```

### Interactive Input

```python
user_query = input("Enter your query (English / Hindi / Kannada): ")
display_hybrid_results(user_query, top_k=TOP_K)
```

### Voice Search (Colab only)

1. Run the language selector cell and pick your language
2. Run `record_and_search()` — a Stop button appears in the output
3. Speak your query, press Stop — results appear automatically

```python
# Force a language for better accuracy
search_schemes_by_voice("query.webm", language="hi")   # Hindi
search_schemes_by_voice("query.webm", language="kn")   # Kannada
search_schemes_by_voice("query.webm", language=None)   # Auto-detect
```

---

## ⚙️ Configuration

All tunable parameters live in the config cell:

```python
EMBED_MODEL      = "paraphrase-multilingual-MiniLM-L12-v2"
WHISPER_MODEL    = "medium"    # small / medium / large
TOP_K            = 5           # results to return
SEMANTIC_WEIGHT  = 0.7         # 70% semantic score
BM25_WEIGHT      = 0.3         # 30% BM25 score
```

Increase `SEMANTIC_WEIGHT` for better conceptual matching; increase `BM25_WEIGHT` for stricter keyword results.

---

## 📊 Evaluation

The notebook includes a retrieval evaluation suite with 22 test cases across all three languages covering topics like housing, scholarships, pensions, disability support, and skill training.

```python
results = evaluate_text_retrieval(test_cases, top_k=TOP_K)
```

**Metrics reported:**

- **Hit Rate @K** — % of queries with at least one relevant result in top-K
- **Mean Precision @K** — average fraction of top-K results that are relevant
- **MRR (Mean Reciprocal Rank)** — how high the first relevant result appears

Results are broken down per language (English 🇬🇧 / Hindi 🇮🇳 / Kannada 🌿) for targeted debugging.

---

## 📂 Project Structure

```
.
├── nlp_project.ipynb       # Main notebook
├── scheme_embeddings.npy       # Cached embeddings (auto-generated)
├── scheme_faiss.index          # FAISS index (auto-generated)
└── README.md
```

---

## 🗺️ Roadmap

- [ ] Streamlit / Gradio web UI
- [ ] Support for more Indian languages (Telugu, Tamil, Marathi)
- [ ] Re-ranking with a cross-encoder model
- [ ] REST API wrapper (FastAPI)
- [ ] Docker container for offline deployment

---

## 🙏 Acknowledgements

- Scheme data sourced from [MyScheme — Government of India](https://www.myscheme.gov.in/)
- Embedding model: [`paraphrase-multilingual-MiniLM-L12-v2`](https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2) by UKPLab
- STT: [OpenAI Whisper](https://github.com/openai/whisper)

---

## 📄 License

MIT License — feel free to use, modify, and distribute.
