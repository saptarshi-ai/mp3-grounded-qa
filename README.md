# MP3 · Grounded Q&A — a cited document assistant

Week 12 graded mini project for the IITM Pravartak AI & RAG Engineering programme.

I extended the Mini-RAG pipeline from MP2 into an assistant I'd actually trust over a small document set. It:

1. **Cites** every claim inline with `[n]`, and checks each marker maps to a chunk that was really retrieved.
2. **Abstains** with "I don't know based on the provided documents." when nothing relevant comes back.
3. **Measures itself** with an eval harness that reports retrieval hit rate, grounded response rate, abstention accuracy and hallucination rate.

Corpus: the five Sherlock Holmes story summaries from the course kit (`corpus/`, 73 chunks after ingest).
Stack: OpenAI `gpt-4o-mini` + `text-embedding-3-small`, Qdrant, the `openai` and `qdrant-client` libraries directly. No frameworks, no new dependencies.

## Results

Final run (`mp3_eval.txt`, top-K 8, threshold 0.40, 6 answerable + 3 should-abstain questions):

| Metric | Score |
|---|---|
| Retrieval hit rate | 1.0 |
| Grounded & cited rate | 1.0 |
| Abstention accuracy | 1.0 |
| Hallucination rate | 0.0 |

With the kit defaults (top-K 3, threshold 0.30) the grounded rate was 0.5 (`logs/eval_baseline_k3_t030.txt`). The answer chunks for two questions sat at rank 4 and rank 7, so the model never saw them. The sweep that got me to k=8 / 0.40 is in `logs/threshold_sweep.txt`, and the full write-up is in `mp3_reflection.md`.

## Demo

Recording of a live run: [`demo/mp3_demo.mp4`](demo/mp3_demo.mp4) (about 2 minutes). It asks three questions through `answer()`: a cited answer (Julia Stoner), a two-chunk answer (the blue carbuncle) and a correct "I don't know" (Professor Moriarty), then shows `mp3_eval.txt`.

## Run it

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env    # add OPENAI_API_KEY and QDRANT_URL, then: source .env

python mp3_grounded_qa.py selftest   # offline checks, no keys needed
python mp3_grounded_qa.py ingest     # build the Qdrant collection (once)
python mp3_grounded_qa.py ask        # interactive Q&A
python mp3_grounded_qa.py eval       # the evaluation table
```

I ran Qdrant locally in Docker (`docker run -p 6333:6333 qdrant/qdrant`, `QDRANT_URL=http://localhost:6333`, no API key). On Windows, set `PYTHONUTF8=1` before redirecting `eval` to a file, otherwise the ✓/✗ characters fail to encode.

## Configuration

| Variable | Default | What it does |
|---|---|---|
| `RETRIEVAL_TOP_K` | 8 | Chunks passed to the model. I added this; the kit hard-coded 3 |
| `RETRIEVAL_THRESHOLD` | 0.40 | Abstain if the best chunk scores below this (cosine). Kit default was 0.30 |

## What I implemented

The pipeline (load → chunk → embed → store → retrieve → answer) came with the starter. I wrote the four functions that make it trustworthy:

| Function | Job |
|---|---|
| `format_context` | Number the retrieved chunks `[1..n]` for the prompt and return a parallel sources list |
| `validate_citations` | Parse `[n]` markers (including grouped ones like `[1, 2]`) and flag any that don't map to a real source |
| `should_abstain` | Return `True` when there are no chunks or the top score is below the threshold |
| `evaluate` | Score the assistant over a question set and return the four rates |

## Files

| Path | What |
|---|---|
| `mp3_grounded_qa.py` | The assistant |
| `mp3_eval.txt` | Output of `python mp3_grounded_qa.py eval` |
| `data/learner_questions.jsonl` | My three questions: a multi-hop Blue Carbuncle question, a Red-Headed League question, and a Hound of the Baskervilles question that must abstain |
| `data/predefined_questions.jsonl` | The kit's question set |
| `mp3_reflection.md` | Reflection, including the threshold / top-K sweep |
| `logs/` | Selftest, ingest, ask, the k=3 baseline, the sweep summary and all nine raw sweep runs |
