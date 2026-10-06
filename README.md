# korovas

Бонитировка мясного КРС по фото: линейные промеры (CV) и баллы приказа Минсельхоза N 270 (VLM). Два HTTP-сервиса; datapipe вызывает оба. Друг друга не дергают.

```
  photos (bytes)
       ├─ cow_measure  GPU  → cm (+ kg)     POST /v1/measure
       └─ cow_scoring  CPU  → баллы статей  POST /v1/score
              (промеры опциональны в JSON; маппинг ключей — на вызывающем)
```

| Package | What | Docs |
|---|---|---|
| [`cow_measure`](cow_measure/README.md) | YOLO + Metric3D + mask + ground plane → промеры | [deploy](cow_measure/deploy.md) |
| [`cow_scoring`](cow_scoring/README.md) | OpenRouter VLM по статям шкалы (возраст/пол) | [deploy](cow_scoring/deploy.md) |

Scoring **не** подставляет CV автоматически (решение: только приказ + форма). Measure labels ≠ form field names.

Полный контракт скоринга: [docs/integration/vlm-inference-service.md](docs/integration/vlm-inference-service.md), поля: [vlm-service-contract.md](docs/integration/vlm-service-contract.md).

## Quick start

Measure (GPU, repo root, submodule + weights first — see deploy):

```bash
docker compose -f cow_measure/docker-compose.yml up --build -d
curl -s http://127.0.0.1:8000/health
```

Scoring (CPU):

```bash
cd cow_scoring
cp env.example .env    # OPENROUTER_API_KEY
pixi install -e vlm
pixi run -e vlm vlm-serve                 # :8000
# or: docker compose up --build           # host 8100 → 8000
```

`cow_scoring` — **pixi**. `cow_measure` и datapipe — **uv**. Prod Postgres/S3 — **read-only** ([CLAUDE.md](CLAUDE.md)).

## Research (scoring)

| Command | |
|---|---|
| `pixi run -e vlm vlm-score --dry-run …` | plan, no API |
| `pixi run -e vlm python -m scripts.research.run_inference` | offline dataset |
| `pixi run -e vlm python -m scripts.research.metrics --results …` | vs expert |
| `pixi run -e db python -m scripts.research.download_dataset` | prod dump, read-only |

## Knowledge

- [docs/wiki/INDEX.md](docs/wiki/INDEX.md) — decisions, experiments, sources
- [CLAUDE.md](CLAUDE.md) — invariants (DB, pixi/uv, tasks in git)
