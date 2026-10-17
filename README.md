GDPR RAG: plain bi-encoder retrieval versus cross-encoder reranking over the same 607 chunks.

## run

tested on Windows, Python 3.11, uv and Ollama:

```powershell
uv venv .venv --python 3.11
uv pip install --python .venv/Scripts/python.exe -r requirements.txt
.venv/Scripts/python -m ipykernel install --user --name rag-rerank --display-name rag-rerank
ollama pull gemma2:9b-instruct-q4_K_M
.venv/Scripts/python -m nbconvert --to notebook --execute --inplace --ExecutePreprocessor.kernel_name=rag-rerank --ExecutePreprocessor.timeout=3600 --KernelManager.transport_encryption=auto rag_rerank.ipynb
```

keep Ollama running. the labelled answers stay cached; timing cells make live calls. the notebook already includes outputs.

## files

```text
rag_rerank.ipynb       all code and outputs
requirements.txt      runtime package pins
data/                 PDF, eval/stress questions, proposed answer labels
results/              cached answers, tables, figures, run_info.json
```

data/ holds inputs; results/ holds notebook outputs. private development notes are gitignored under archive/.

## results

16 fixed eval questions; the 24 stress questions are examples only.

| pipeline | hit@3 | facts@3 | sufficient@3 | mrr@10 |
|---|---:|---:|---:|---:|
| baseline | 0.438 | 0.396 | 0.375 | 0.376 |
| rerank | 0.562 | 0.500 | 0.438 | 0.594 |

top-10 fact-coverage ceiling: 0.646; 6/16 questions have a required fact missing from the pool.
proposed eval-answer labels: 25.0% fully correct for baseline and 25.0% for rerank; these labels await final review.
mean retrieval: 4.46 -> 19.42 ms; live end to end: 4.83 -> 6.14 s.
full-corpus cross-encoding at 607 chunks: 459.8 ms/query. larger sizes in scaling.csv are extrapolated, not benchmarked.

![measured latency](results/figures/latency.png)

## notes

RTX 4070 Laptop, 8 GB VRAM; CUDA 12.8; encoders on cuda, FAISS on CPU; Ollama placement 24%/76% CPU/GPU.
retrieval has five warm-ups and 160 trials; generation has one live call per question/pipeline, alternating order. prefix reuse affects prefill; [prompt counters](https://docs.ollama.com/api/chat) are kept as reported.
timings changed between builds on this shared desktop; the CSVs record this run.
bi-encoder: `sentence-transformers/msmarco-MiniLM-L6-cos-v5` @ `14ca9be4bbcf1402eac0f43a2e2ccb6e0f994ba3`.
cross-encoder: `cross-encoder/ms-marco-MiniLM-L6-v2` @ `233902d25c440f23af6f7d6e94d2946bac0bee0a`.
LLM: `gemma2:9b-instruct-q4_K_M`, digest `c20bec88025f3aefa420efd64dca0ee4abc8e36078d6b74b0f660cdcbd5d1982`; options and runtime versions are in run_info.json.
[original 2016 PDF](https://publications.europa.eu/resource/cellar/3e485e15-11bd-11e6-ba9a-01aa75ed71a1.0006.01/DOC_1), 88 pages; sha256 `bd84e63f5b622b739a83389afc3b30d240f792bb88d8eb03a816c9a82b0c2499`.
corpus omits the title/citations before the recitals, footnotes, and the closing/signature material after article 99. some chunks need other paragraphs for context.
answer labels include borderline notes; registered support is not exhaustive, notably in stress s17. see answer_labels.yaml and the case-study diagnostics.
cached answers are preserved from 9d60343; the 16-question set is small and source-specific. development exposure is logged in eval_queries.yaml.
