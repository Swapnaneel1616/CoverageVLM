# CoverageVLM

Production-oriented visual copilot for **urban transmitter placement**.

Given a city building map from [RadioMapSeer](https://radiomapseer.github.io/), the system:

1. Ranks candidate transmitter sites with a trained **Coverage UNet** (+ score ranking)
2. Shows predicted (and optional ground-truth) **coverage overlays**
3. Uses a **VLM** (e.g. GPT-4o) to explain dead zones and support refine turns

## Repo status

Scaffold + plan in progress. See `Plan.md` for architecture and phases.

## Dataset (not in this repo)

Download **RadioMapSeer** separately and place it at:

```text
SemesterProject/RadioMapSeer/
  png/buildings_complete/
  antenna/
  gain/DPM/
```

The dataset is gitignored (~3GB). Do not commit it.

## Setup

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
pip install -r requirements.txt
```

Copy secrets locally (never commit):

```text
.env
OPENAI_API_KEY=your_key_here
```

## Config

Default settings live in `config/default.yaml` (coverage threshold, splits, train/API params).

## Roadmap (high level)

| Phase | Focus |
|-------|--------|
| 0 | Tooling / config |
| 1 | Build labels + splits for all maps |
| 2 | Train Coverage UNet |
| 3 | Rank sites via UNet scores |
| 5 | FastAPI + UI + VLM explain |
| 6 | Docker / polish |

## License / data credit

RadioMapSeer: Yapar et al., arXiv:2212.11777. Cite the dataset paper if you use the data.
