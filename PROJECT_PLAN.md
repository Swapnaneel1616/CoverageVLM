# CoverageVLM — Success Flow & Semester Plan

**Title:** CoverageVLM: A Visual Copilot for Urban Transmitter Placement  
**Track:** Application  
**End goal:** An interactive demo that proposes a transmitter on a city map, shows coverage, explains dead zones in language, and refines placement over a few turns using RadioMapSeer ground truth.

**Architecture diagram:** `CoverageVLM_architecture_diagram.jpg`

---

## What “success” means

You succeed if a classmate can watch **one map** and understand:

1. Where the transmitter was placed  
2. What the coverage overlay shows  
3. Why the system moved it to a better spot  
4. That your method beats **random** on coverage % / rank  

No live ray tracing. No multi-transmitter. No fine-tuning required.

---

## Easy success flow (follow this order)

```text
  [1] DATA WORKS
        Load building map + look up coverage for any TX ID
              │
              ▼
  [2] MANUAL LOOP
        You pick an ID → see overlay + score → pick again
              │
              ▼
  [3] VLM LOOP
        AI sees map → picks valid ID → explain → refine 3–5 turns
              │
              ▼
  [4] BASELINES + NUMBERS
        Compare vs random / heuristic / text-only LLM
              │
              ▼
  [5] DEMO POLISH
        One clear story map + short narration + latency OK
```

**Rule:** Do not start the next step until the current one works end-to-end.

---

## High-level system (abstract)

| Piece | Role |
|--------|------|
| **RadioMapSeer** | Truth table: building maps + ~80 precomputed coverages |
| **App / Orchestrator** | Draw candidates, validate ID, fetch coverage, score & rank |
| **VLM Agent** | Sees images, picks a valid candidate ID, explains dead zones |
| **UI Demo** | Map + overlay + chat + score vs best |
| **Baselines** | Random, heuristic, text-only LLM — prove you are better |

**Runtime loop (each turn):**

```text
Map + numbered candidates
        →  pick TX ID (+ reason)
        →  lookup coverage in RadioMapSeer
        →  show overlay + % covered + rank among 80
        →  VLM sees result → suggest better ID
        →  repeat 3–5 times
```

---

## Build layers (bottom → top)

1. **L1 Data** — `(map_id, tx_id) → coverage image + score`  
2. **L2 Scoring** — coverage %, rank, % of optimum, overlay image  
3. **L3 Agent** — constrained VLM (only valid IDs) + short rationale  
4. **L4 Demo/Eval** — UI, baselines, metrics, demo script  

---

## Semester plan (suggested)

### Phase 0 — Lock scope (Days 1–2)
- Confirm: 1 transmitter, RadioMapSeer lookup, 3–5 refine turns, API VLM  
- Cut: multi-TX, fine-tuning, live ray tracing  
- Split work: Data | Agent/prompts | UI | Eval  

**Done when:** 1-page scope note both teammates agree on.

### Phase 1 — Data foundation (Week 1–2)
- Download / organize RadioMapSeer  
- Load one building map  
- List ~80 candidate TX IDs for that map  
- Lookup coverage for a chosen ID  
- Define coverage score (e.g. % above a pathloss threshold)  
- Know the best ID on that map (for “% of optimum”)  

**Done when:** a notebook/script prints score + shows overlay for any ID, no AI involved.

### Phase 2 — Manual propose → overlay loop (Week 2)
- Numbered candidates drawn on the map  
- User types/clicks an ID  
- App shows coverage overlay + % + rank  
- Repeat a few times by hand  

**Done when:** you can demo the loop without any LLM.

### Phase 3 — VLM agent (Week 3–4)
- Call one API VLM (e.g. GPT-4o) with the map image  
- Force answer format: valid candidate ID + short reason  
- Reject invalid IDs and retry  
- Turns 2–5: send coverage overlay + history  
- Keep latency “seconds” for a live demo  

**Done when:** 3–5 automatic refine turns work on several maps.

### Phase 4 — Baselines & evaluation (Week 4–5)
- Random baseline  
- Simple heuristic (center / open area)  
- Text-only LLM (coords, no image)  
- Your VLM method  
- Metrics: coverage % of optimum, rank among 80, improvement across turns, latency  

**Done when:** a table shows VLM beating random (and ideally improving over turns).

### Phase 5 — Demo polish (Final weeks)
- Pick 1–2 “story” maps  
- Script: propose → overlay → explain → refine → final vs best  
- Optional stretch: open VLM (e.g. Qwen2.5-VL) side-by-side  
- README: run demo in a few commands  

**Done when:** a classmate understands why the transmitter moved.

---

## Weekly checklist (copy into your tracker)

| Week | Focus | Exit criteria |
|------|--------|----------------|
| 1 | Data load + one map lookup | Overlay for any TX ID |
| 2 | Manual UI/script loop | Hand-driven place → see → place |
| 3 | VLM turn 1 | Valid ID + rationale from image |
| 4 | Multi-turn refine | 3–5 turns with overlay feedback |
| 5 | Baselines + metrics | Beat random on held-out maps |
| 6+ | Demo + report | Live demo + short write-up |

---

## Team split (suggested)

| Owner | Owns |
|--------|------|
| Teammate A | RadioMapSeer I/O, scoring, overlays |
| Teammate B | UI + demo narrative |
| Shared | VLM prompts, eval harness, report |

Swap roles if better for skills; keep one person “demo owner.”

---

## Risks & how to stay safe

| Risk | Mitigation |
|------|------------|
| Dataset format confusing | Start with 1 map; freeze score formula early |
| VLM invents invalid IDs | Strict allow-list; retry on bad output |
| API cost / latency | Cache images; small eval set; short prompts |
| Scope creep | Stick to cut list: no multi-TX, no fine-tune |
| Weak demo story | Choose maps where dead zones are visually obvious |

---

## Suggested repo shape (when you start coding)

```text
SemesterProject/
  data/                 # RadioMapSeer paths / loaders
  scoring/              # coverage %, rank, overlay
  agent/                # VLM prompts + ID validation
  ui/                   # Streamlit / Gradio / web demo
  eval/                 # baselines + metrics
  docs/
  CoverageVLM_architecture_diagram.jpg
  PROJECT_PLAN.md
  README.md
```

---

## Final deliverable checklist

- [ ] Interactive propose → overlay → refine demo  
- [ ] RadioMapSeer ground-truth coverage (no live ray tracing)  
- [ ] Language explanation of dead zones  
- [ ] 3–5 refinement turns  
- [ ] Baselines: random, heuristic, text-only  
- [ ] Numbers: beat random on % of optimum / rank  
- [ ] Latency OK for live demo  
- [ ] Short report + references [1]–[5] from proposal  
