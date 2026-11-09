# IMF Country Report → Text-to-Dashboard LLM Pipeline

Coursework for **Data Analysis with AI (7PAM2032), Assignment 1: LLM Coding Case Study**.

The pipeline turns an unstructured IMF Article IV country report (PDF) into summaries, thematic statistics,
schema-valid indicator JSON, grounded RAG answers and a multi-plot dashboard. Each LLM output is checked by a second
"judge" LLM, by deterministic rules and against a small set of human-verified values.

**Document:** *Switzerland: 2026 Article IV Consultation — Press Release; and Staff Report*, IMF Country Report
No. 26/233 (staff report dated 4 August 2026; the Board concluded the consultation on a lapse-of-time basis on 25 August 2026).

## Run it

[Open in Colab](https://colab.research.google.com/github/<YOUR_GITHUB_USER>/<YOUR_REPO>/blob/main/imf_llm_pipeline.ipynb)

1. Open the notebook with the link above and choose **Runtime → Run all**. No GPU and **no API key** are needed.
2. The first cell clones this repository so the committed `cache/`, `outputs/` and `figures/` folders are available.
3. **Upload the report PDF when asked.** The IMF website blocks automated downloads from Colab (HTTP 403), so the
   notebook asks for an upload. Download *IMF Country Report No. 26/233* from imf.org, or use the allocated copy, then
   click **Choose Files**. Alternatively, place the file at `data/imf_report.pdf` before running.
4. **Replay mode (default):** without an API key, all 104 LLM responses are replayed from `cache/llm_cache.json`, so
   every number and figure in the report is reproduced exactly. The run takes about 3–5 minutes, mostly downloading
   the embedding model. If a newer library version changes the extracted text slightly, responses are matched to their
   pipeline step instead.
5. **Live mode (optional):** add a Colab secret named `GEMINI_API_KEY` (key icon in the left sidebar) with notebook
   access switched on. A full live run makes about 104 Gemini calls. The free tier allows only about 20 requests per
   model per day, so a paid key is needed for a complete live run.

## Pipeline

| Step | Implementation | Brief task |
|---|---|---|
| Parsing and cleaning | `pypdf` per page. Removes running headers/footers, repairs pypdf decimal splitting in tables (`1 .2` → `1.2`), drops chart residue and the figure pages (calibrated detector), rebuilds paragraphs, and drops near-duplicate pages (Jaccard). | 1 |
| Chunking | Sentence units packed to 350 DistilBERT tokens with a 50-token overlap. Tokens are counted with the embedding model's own tokenizer. | 1 |
| Summaries | Executive summary (6–8 bullets with page citations) and one summary per section (< 100 words, **enforced in code** with a repair loop). | 1 |
| Evaluator | A second Gemini model acts as LLM-as-a-Judge. It checks claims first, then scores consistency, completeness, factual alignment and constraint adherence (1–5). | 1, 2 |
| Themes | LLM multi-label classification of every chunk into the 7 themes, compared with a regex keyword baseline. | 2 |
| Risk insights | 5–7 strengths and 5–7 vulnerabilities, each with a verbatim quote that is checked against the source. | 2 |
| Indicator JSON | Retrieval per field, then extraction against the strict 7-field schema plus a separate evidence object. Then `jsonschema` validation, plausibility checks, **fallback = `null`**, judge audit, and comparison with human ground truth. | 2 |
| Trends | The LLM reads Tables 1 and 6. A deterministic regex table parser cross-checks every cell. The staff-report copy of Table 1 (p.40) is **embedded as JPEG images with no text layer**, so the pipeline automatically uses the identical text copy in the press release (p.6). | 2 |
| RAG | `sentence-transformers/multi-qa-distilbert-cos-v1` + FAISS `IndexFlatL2` on L2-normalised vectors (equivalent to cosine ranking), top-k = 4. Grounded prompt with `DATA NOT FOUND`, RAG-triad judging, and a closed-book baseline. | 3 |
| Extension | Strict `system_instruction` + native JSON schema vs a generic user prompt, compared on schema compliance, latency, accuracy and judge score. | Extra credit |
| Dashboard | Tidy data with `pd.melt`; 6-panel dashboard plus individual figures at 300 dpi. | 4 |

## Repository layout

```
imf_llm_pipeline.ipynb      # the notebook (all code, prompts and figures)
cache/llm_cache.json        # every LLM response, keyed by a SHA-256 hash of (model, prompts, schema, temperature)
outputs/*.json              # machine-readable results (indicators, evidence, audits, themes, trends, RAG, summaries)
figures/*.png               # dashboard and individual figures (300 dpi)
requirements.txt
LICENSE                     # MIT
```

Key output files:
- `core_indicators.json`: the indicators in the exact target schema.
- `core_indicators_evidence.json`: the page, quote, reference period, alternatives seen, fallback log and retrieval diagnostics for each value.
- `core_indicators_audit.json`: the judge's verdicts, compared with the human ground truth.

## Reproducibility and integrity
- Temperature 0 and a fixed seed are used for every call, and the model names are recorded in `outputs/run_config.json`.
- The ground-truth values in the notebook were checked by hand against the PDF and are used **only for evaluation**.
- The IMF PDF is marked "Not for Redistribution", so it is not committed (see `.gitignore`).
- All report-specific settings (sections, table titles, ground truth, RAG questions, plotted series) live in the configuration cell.

## References
- International Monetary Fund (2026) *Switzerland: 2026 Article IV Consultation — Press Release; and Staff Report*. IMF Country Report No. 26/233. Washington, DC: IMF.
- Johnson, J., Douze, M. and Jégou, H. (2019) 'Billion-scale similarity search with GPUs', *IEEE Transactions on Big Data*.
- Lewis, P. et al. (2020) 'Retrieval-augmented generation for knowledge-intensive NLP tasks', *NeurIPS*.
- Reimers, N. and Gurevych, I. (2019) 'Sentence-BERT', *EMNLP-IJCNLP*.
- Zheng, L. et al. (2023) 'Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena', *NeurIPS*.

## License
MIT — see `LICENSE`.
