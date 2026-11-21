# SetuGuard

**Linking Android banking malware to mule bank accounts.**

SetuGuard brings together two fraud-detection problems that are usually handled separately. It then links them through shared indicators:

- **PS1: APK evidence extraction.** Static analysis of Android APKs to pull out banking-trojan evidence, map it to MITRE ATT&CK for Mobile, write an analyst narrative and generate YARA rules.
- **PS2: mule account scoring.** An XGBoost model that ranks bank accounts by mule-account risk, with SHAP explanations.
- **Bridge: the join.** Exact matching on publisher certificate SHA-256 and normalised C2 host, connecting a malicious APK to the flagged accounts it is linked to.

Everything is wired into one Flask backend and a web dashboard.

## How it works

```
 APK upload ──► PS1: Androguard static analysis
                   ├─► FAISS retrieval over a MITRE ATT&CK knowledge base
                   ├─► rule-based verdict (score, severity)
                   ├─► LLM narrative (Mistral-7B, optional, never changes the verdict)
                   └─► YARA rule generation + validation gate
                                   │
                                   ▼  IOCs (cert SHA-256, C2 host)
                                 Bridge  ◄── PS2: XGBoost mule-account scores
                                   │                 (CSV upload, SHAP drivers)
                                   ▼
                           Dashboard alert card
```

**The LLM never decides anything.** `_rule_based_verdict()` is the only source of the verdict, confidence, risk score and severity. The language model adds a narrative and citations only. With the model server turned off, the same APK gets the same verdict (see `harness/browser_evidence/ollama_down/`).

## Results

**PS2 mule-account model.** Median [IQR] over 20 stratified 80/20 holdouts with fixed hyperparameters:

| Metric | Value |
|---|---|
| AUROC | 0.872 [0.851–0.907] |
| AUCPR | 0.271 [0.221–0.362] |
| Recall in top 5% of accounts | 53.1% [43.8–62.5] |
| Lift in top 1% of accounts | 25.2× [18.9–37.9] |

**PS1 against legitimate banking apps.** Measured against a pre-registered corpus of 51 real legitimate banking apps from 32 issuers versus 360 confirmed banking malware samples. PRIMARY AUC is **0.1444 [0.0905, 0.2081]**: static features alone rank real banking apps *above* malware. PS1 therefore ships as an **evidence extractor, not a detector**. It has no threshold and claims no false-positive rate. Full details are in `REPORT_FACTS.md` and `ERRATA.md`.

## Repository layout

| Path | Contents |
|---|---|
| `setuguard_app/backend/` | Flask API (`app.py`) that ties PS1, PS2 and the bridge together |
| `setuguard_app/frontend/` | Dashboard (HTML, CSS, JS, Chart.js) |
| `setuguard_ps1/` | PS1 pipeline: static analysis, knowledge base, RAG report, YARA generation, validation gate, test harnesses |
| `ps2/` | Offline PS2 research scripts (data audit, baseline, counterfactuals, graph features) |
| `models/` | Trained PS2 XGBoost model and its metrics |
| `bridge/` | IOC matcher and confusion-matrix validation |
| `harness/` | Corpus building, scoring, scaling tests, browser smoke tests and evidence |
| `demo/` | Demo runbook and fixtures |
| `data/` | Dataset feature descriptions |
| `docs/` | Evidence snapshots |

## API

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/analyze_apk` | Analyse an uploaded APK (max 50 MB) |
| POST | `/api/analyze_dataset` | Score an uploaded account CSV with the PS2 model |
| POST | `/api/bridge` | Link an analysed APK's IOCs to scored accounts |

## Getting started

### Prerequisites

- Python 3.10+
- Optional: [Ollama](https://ollama.com) with `mistral` and `nomic-embed-text` for the LLM narrative

### Run the backend

```bash
cd setuguard_app/backend
pip install -r requirements.txt
python app.py
```

The API runs on http://localhost:5000.

### Open the dashboard

Open `setuguard_app/frontend/index.html` in a browser. If you open it as a file rather than serving it from the backend, set the **Base URL** in the dashboard's Settings page to `http://localhost:5000`.

### Optional: LLM narrative

```bash
ollama pull mistral
ollama pull nomic-embed-text
```

Without Ollama, the app still returns the same verdicts. Only the narrative text is left out.

## Tech stack

Python, Flask, Androguard, YARA, FAISS, Ollama (Mistral-7B, nomic-embed-text), XGBoost, SHAP, scikit-learn, pandas, DiCE, NetworkX, Chart.js, Playwright

## Notes

- **No APK samples are included.** Malware and benign APK corpora are excluded by `.gitignore` and must never be committed to a public repo.
- **The bridge's account-to-APK links are constructed for demonstration.** No public dataset links real Android malware indicators to real bank accounts. The matching mechanism is real; the linkage it is demonstrated on is labelled as constructed.
- **Most of `ps2/` is offline research, not the shipping model.** The live model comes from `harness/train_ps2_model.py`. See `ps2/README.md`.
- **Further documentation:** `SETUGUARD_BRIEFING_v3.md` (system overview), `REPORT_FACTS.md` (quotable numbers), `ERRATA.md` (corrections), `setuguard_app/README.md` (app internals) and `demo/DEMO_RUNBOOK.md` (demo steps).
