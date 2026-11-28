# AI-Driven Dermatological Agent
*A multimodal framework for automated skin-lesion screening and decision support.*

> **Educational, preliminary screening support only. Not a medical device. No diagnosis. Does not replace a dermatologist.** See [DISCLAIMER.md](DISCLAIMER.md).

Upload a skin image + describe symptoms → get top model suggestions with confidence, a **Grad-CAM** heatmap, a rule-based **urgency category**,
**evidence-grounded** explanation with citations, precautions, referral guidance, and a RAG follow-up chat.

## Honest status
| Item | Status |
|---|---|
| Trained weights | **Not bundled.** Without `backend/models/skin_lesion_effnet_b0.pt` the API runs in `no_checkpoint` mode: **no image predictions**, symptom-based guidance only. |
| Untrained demo | `ALLOW_UNTRAINED_DEMO=true` loads *random* weights, labelled **DEMO** in API + UI; its output is excluded from risk scoring. |
| Model scope | Only the 7 HAM10000 classes. Cannot recognise rashes, eczema, psoriasis, infections, acne, nail/hair disease. |
| Accuracy | **None claimed.** Train, then run `ml/evaluate.py` for held-out metrics. |
| Knowledge base | Student-written educational summaries (`backend/app/knowledge/`), not clinician-reviewed. |
| Tested | Backend: 15 pytest tests pass. ML scripts: smoke-tested end-to-end on synthetic data (random init, CPU) only; **never trained on real HAM10000 here**. Frontend: type-check + production build + one headless-browser run of the full flow. Docker files: not exercised. |

## Layout
```
backend/   FastAPI app (app/main.py, api/routes.py, services/{vision,gradcam,rag,llm,risk,pipeline,preprocessing}.py,
           knowledge/*.md, tests/), models/ (put checkpoint here), requirements.txt, .env.example, Dockerfile
ml/        prepare_dataset.py, dataset.py, model.py, train.py, evaluate.py, utils.py, classes.py, colab_train_guide.ipynb
frontend/  React + TypeScript + Tailwind + lucide-react (Vite): Landing, ImageUpload, SymptomForm, ProgressSteps, ResultsDashboard, ChatPanel
docs/      ARCHITECTURE.md (stack rationale, trade-offs, known gaps)
scripts/   make_zip.py
```

## Quick start
```bash
# 1. Backend  (Python 3.10+)
cd backend && python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt        # CPU torch: add --extra-index-url https://download.pytorch.org/whl/cpu
cp .env.example .env
pytest -q                                  # 15 tests
uvicorn app.main:app --reload              # http://localhost:8000/docs

# 2. Frontend (Node 18+)
cd ../frontend && npm install && npm run dev   # http://localhost:5173  (proxies /api to :8000)
```
Just want to see the pipeline? Set `ALLOW_UNTRAINED_DEMO=true` in `backend/.env` (clearly labelled, meaningless output).

## Train a real model
```bash
cd ml && pip install -r requirements.txt
python prepare_dataset.py --download --raw-dir data/raw/HAM10000   # or download manually, see script docstring
python train.py                                                    # writes ../backend/models/skin_lesion_effnet_b0.pt
python evaluate.py                                                 # reports/ : metrics, per-class sensitivity, confusion matrix
```
No GPU? Use `ml/colab_train_guide.ipynb` (free T4, ~30-60 min). Restart the API: `GET /api/v1/health` should show `model_status: ready`.
Dataset notes: HAM10000 (10,015 images, ~67% nevi) is heavily imbalanced; the split is **by lesion_id** to prevent leakage; training uses
inverse-sqrt class weights and selects on validation macro-F1. Check the dataset licence (CC BY-NC 4.0 for HAM10000) before any non-academic use.

## LLM (optional)
Set `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL` in `backend/.env` (any OpenAI-compatible server, incl. local Ollama/vLLM). The LLM only writes the prose
explanation and chat answers from retrieved context. Precautions and referral lists always come from the knowledge base. If the key is missing or the call fails,
a template/extractive fallback is used. Free-text patient notes are never sent.

## API
| Endpoint | Purpose |
|---|---|
| `GET /api/v1/health` | status, model name/status, RAG backend & chunk count, LLM configured |
| `POST /api/v1/analyze` | multipart: `image` (JPEG/PNG/WebP <= 8 MB) + `metadata` (JSON, see `ClinicalMetadata`) → predictions, Grad-CAM (base64 PNG), risk, explanation, precautions, referral, citations |
| `POST /api/v1/chat` | `{question, history, context_labels, risk_level}` → grounded answer + citations |

Errors are `{"error": {"code", "message"}}` (413 too large, 415 wrong type, 422 invalid image/metadata).

## Extending the knowledge base
Add `backend/app/knowledge/<topic>.md` (`# Title`, `## Section`). Lesion files must be `lesion_<class>.md` with `Overview`, `Typical features`, `Precautions`, `When to seek care`.
Have a clinician review before real use (see `_PROVENANCE.txt`).

## Known limitations
No auth/rate limiting, no probability calibration, no out-of-distribution rejection, no skin-tone fairness evaluation, no frontend tests. Details in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).
