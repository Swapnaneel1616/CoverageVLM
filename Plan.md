# CoverageVLM — Production Plan (Track 1+2)

**Product:** Visual copilot for urban transmitter placement  
**Stack decision:** Train **Coverage UNet** + **site ranking** on full RadioMapSeer, serve via **API + UI**, use **VLM for explanations**  
**Data:** RadioMapSeer (local): `png/buildings_complete`, `antenna/*.json`, `gain/DPM/{map}_{tx}.png`

---

## 1. Goal

Ship a production-style application that:

1. Loads a city building map  
2. Ranks transmitter candidates (trained model, not GPT-only)  
3. Shows coverage overlay (predicted + optional ground-truth toggle)  
4. Explains dead zones in language (VLM)  
5. Supports multi-turn refine  
6. Beats random / heuristic / zero-shot VLM on held-out maps  

**Not the main path:** SAM/DINO as placement baselines.  
**Optional later:** dedicated ranker network if UNet×80 is too slow.

---

## 2. High-level architecture

```text
┌─────────────────────────────────────────────────────────────┐
│  UI  (Streamlit v1 → Next.js optional)                       │
│  map · top-k TX · overlay · scores · chat explain · refine   │
└──────────────────────────────┬──────────────────────────────┘
                               │ REST
┌──────────────────────────────▼──────────────────────────────┐
│  API  (FastAPI)                                              │
│  /maps  /rank  /coverage  /explain  /health                  │
└───────────────┬────────────────────────────┬────────────────┘
                │                            │
     ┌──────────▼──────────┐      ┌──────────▼──────────┐
     │ Placement service   │      │ Explanation service │
     │ Coverage UNet       │      │ GPT-4o / DeepSeek-VL│
     │ + score / rank      │      │ (language only)     │
     └──────────┬──────────┘      └─────────────────────┘
                │
     ┌──────────▼──────────────────────────────────────┐
     │ Data: RadioMapSeer + labels.parquet + checkpoints│
     │ Cache: (map, tx) → score / overlay               │
     └─────────────────────────────────────────────────┘
```

### Runtime loop

```text
Building map (+ TX candidates)
        │
        ▼
┌───────────────────┐
│  Coverage UNet    │  → predicted pathloss/gain map
└─────────┬─────────┘
          │
          ├─► overlay in UI
          │
          ▼
┌───────────────────┐
│  Site ranking     │  → scores for 80 candidates
└─────────┬─────────┘
          │
          ├─► top-k TX suggestions
          │
          ▼
┌───────────────────┐
│  VLM explainer    │  → language about dead zones / refine
└───────────────────┘
```

### Component roles

| Component | Role |
|-----------|------|
| **Coverage UNet** | buildings + TX marker → predicted gain map |
| **Site ranking** | score each of 80 candidates via UNet prediction (v1); optional dedicated ranker later |
| **VLM** | explain coverage / suggest refine in natural language (not sole ranker) |
| **RadioMapSeer GT** | training labels + optional GT overlay toggle in UI |

---

## 3. Models

### 3.1 Coverage UNet (Model 1) — required

| Item | Spec |
|------|------|
| Input | 2×256×256: building mask + TX channel (dot/Gaussian at antenna coords) |
| Output | 1×256×256 predicted gain (match DPM PNG scale 0–255 or normalized 0–1) |
| Loss | L1/MSE on non-building pixels (building mask optional) |
| Train data | All maps × 80 TX ≈ 56,080 pairs from `gain/DPM` |
| Split | **By map id** — e.g. 80% / 10% / 10% train/val/test |
| Metrics | RMSE/MAE; optional coverage-% error at fixed `thr` |
| Export | `artifacts/unet.pt` (+ optional ONNX) |

### 3.2 Site ranking (Model 2 path) — required via UNet scoring

**v1 (ship this):**
For candidates `t = 0..79`:
1. `pred = UNet(buildings, tx_t)`  
2. `score[t] = coverage_score(pred, buildings, thr=100)`  
3. Rank by `score` → top-k  

**v2 (optional speed-up):**
- Separate ranker: buildings → 80 scores  
- Labels = true DPM coverage %  
- Loss = MSE or ranking loss  
- One forward pass for all candidates  

### 3.3 VLM explainer — API, not primary trainer

- Input: candidates image + coverage overlay + scores/history  
- Output: JSON `{id?, reason}` for UX refine  
- Baselines only: zero-shot GPT-4o / DeepSeek-VL vs trained ranker  

### 3.4 Explicit non-goals for comparison

- Do **not** use SAM / DINOv2 as primary “placement baselines”  
- DINOv2 may appear later only as an encoder backbone experiment  

---

## 4. Coverage score (frozen definition)

```text
free = buildings < 128
good = (gain >= thr) & free
score = 100 * sum(good) / max(sum(free), 1)
thr = 100   # keep fixed for all train/eval/app metrics
```

Report always:
- raw score %  
- rank among 80  
- % of optimum = score / best_score_on_that_map  

---

## 5. Data layout

```text
RadioMapSeer/
  png/buildings_complete/{map}.png
  antenna/{map}.json          # 80 [x,y]
  gain/DPM/{map}_{tx}.png     # tx = 0..79
```

Ignore for v1: `polygon/`, cars variants, IRT4-only subsets.

### Labels to precompute

- `labels/scores.npy` shape `[n_maps, 80]`  
- `labels/splits.json` → `{train: [...], val: [...], test: [...]}`  
- `labels/meta.json` → thr, paths, version  

---

## 6. Application flows

### 6.1 Rank + overlay

1. User selects map  
2. API runs UNet ranking (or cached)  
3. UI shows top-k markers + scores  
4. User selects TX → predicted coverage overlay  
5. Optional toggle: ground-truth DPM  

### 6.2 Explain + refine

1. Send map/candidates/coverage (+ last score) to VLM  
2. Show natural-language dead-zone explanation  
3. Optionally accept suggested next ID or re-rank excluding tried set  
4. Log session history  

---

## 7. API contract (FastAPI)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | model loaded, version |
| GET | `/maps` | available map ids |
| GET | `/maps/{id}/buildings` | building image |
| POST | `/maps/{id}/rank` | `{top_k, scores[], ids[]}` |
| GET | `/maps/{id}/coverage/{tx}?source=pred\|gt` | coverage image + score |
| POST | `/explain` | VLM explanation / optional next id |
| GET | `/maps/{id}/oracle` | best TX from GT (eval/demo only) |

Config via `.env` (`OPENAI_API_KEY`, model paths). Never commit secrets.

---

## 8. Repo structure (target)

```text
SemesterProject/
  Plan.md
  PROJECT_PLAN.md                 # earlier proposal-oriented plan (reference)
  .env                            # gitignored
  .gitignore
  configs/default.yaml
  data/RadioMapSeer/              # or existing ./RadioMapSeer
  labels/                         # generated scores + splits
  src/
    data/                         # loaders, scoring, splits
    models/unet.py
    models/ranker.py              # optional v2
    train/train_unet.py
    train/train_ranker.py
    infer/predict_coverage.py
    infer/rank_sites.py
    api/main.py
    agent/explain.py
  ui/app.py                       # Streamlit v1
  artifacts/                      # checkpoints
  scripts/build_labels.py
  tests/
  docker-compose.yml
  README.md
  test.ipynb                      # exploration / early baselines only
```

---

## 9. Phased execution plan

### Phase 0 — Freeze & tooling
- [ ] Confirm GPU (local / Colab / school)  
- [ ] `.gitignore` includes `.env`, `artifacts/`, large data  
- [ ] Freeze score `thr=100`, DPM-only, 256×256  

### Phase 1 — Data factory (whole dataset)
- [ ] `scripts/build_labels.py` over all maps × 80  
- [ ] Write `labels/scores.npy` + `splits.json`  
- [ ] Sanity: map 0 best TX matches notebook (~43 @ ~55.6%)  

### Phase 2 — Train Coverage UNet
- [ ] Dataset + UNet implementation  
- [ ] Train on train split; early stop on val RMSE  
- [ ] Save best checkpoint  
- [ ] Test-set RMSE/MAE report  

### Phase 3 — Ranking via UNet scores
- [ ] Batch infer 80 TX per map  
- [ ] Metrics vs random / center heuristic / oracle  
- [ ] Confirm test-set mean rank & % of optimum  

### Phase 4 — Optional ranker network (if latency needs it)
- [ ] Train buildings → 80 scores  
- [ ] Compare accuracy vs UNet-ranking  
- [ ] Swap into API if faster and accuracy OK  

### Phase 5 — API + UI
- [ ] FastAPI endpoints above  
- [ ] Streamlit: map select, rank, overlay, GT toggle  
- [ ] VLM explain panel (reuse notebook prompts)  
- [ ] Refine loop with history  

### Phase 6 — Production hardening
- [ ] Docker Compose (`api` + `ui`)  
- [ ] Caching for ranks/overlays  
- [ ] Eval script → results table for write-up  
- [ ] README: install, train, serve, demo script  

---

## 10. Evaluation & comparison matrix

**Eval only on test maps.**

| Method | Site selection | Overlay source |
|--------|----------------|----------------|
| Random | uniform 0–79 | GT DPM |
| Center heuristic | nearest to (128,128) | GT |
| Zero-shot GPT-4o | VLM image pick | GT |
| **UNet + score rank (ours)** | trained | Pred (+ GT toggle) |
| Oracle | argmax GT scores | GT |

**Metrics:** mean coverage %, mean rank, % of optimum, UNet RMSE, latency/map, API cost for VLM-only baseline.

---

## 11. What we already validated (notebook)

- Dataset paths work locally  
- Score + rank + overlays work  
- GPT-4o can pick IDs and run refine turns  
- Naive visual VLM refine can lose to random; score-aware refine can beat random (e.g. TX 41 ~34.8% vs random ~27% on map 0)  
- Absolute % looks “low” because urban shadowing + `thr=100`; **rank / % of optimum** are the right product metrics  

Notebook stays as **lab / baseline evidence**, not the production app.

---

## 12. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| GPU / train time | Smaller UNet, fp16, smoke on 10 maps then full |
| 80 UNet forwards slow | Batch TX; later dedicated ranker |
| Scope creep | Ship UNet+rank+API+UI before ranker v2 / Next.js |
| Key leakage | `.env` gitignored; rotate if exposed |
| Course vs product | This plan is production-oriented; align with instructor if graded on proposal |

---

## 13. Success criteria (production v1)

- [ ] UNet trains on full train split and reports test RMSE  
- [ ] UNet-ranking beats random & heuristic on test maps  
- [ ] FastAPI serves rank + pred/GT coverage  
- [ ] UI demo: place → overlay → explain → refine  
- [ ] One-command or Docker demo path  
- [ ] Comparison table exported for write-up  

---

## 14. Decision lock

**Track 1+2 locked:**
1. Coverage UNet trained on full RadioMapSeer DPM pairs  
2. Site ranking via predicted coverage scores (optional dedicated ranker later)  
3. FastAPI + UI + VLM explanations for production demo  

**Immediate next step after this plan:** Phase 0/1 scaffolding (`.gitignore`, `build_labels.py`, package layout) — only when explicitly started.
