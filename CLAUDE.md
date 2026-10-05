# korovas

Продукт: бонитировка (балльная оценка) мясного КРС по фото. Два инференс-пакета
(друг друга по HTTP не вызывают; datapipe — клиент обоих):

| Пакет | Роль | Docs |
|---|---|---|
| `cow_measure` | GPU: фото → промеры (cm) и вес (kg) | [README](cow_measure/README.md), [deploy](cow_measure/deploy.md) |
| `cow_scoring` | CPU: фото + возраст/пол → баллы приказа N 270 | [README](cow_scoring/README.md), [deploy](cow_scoring/deploy.md) |

Полная картина (решения, очередь, источники) живёт в графе `docs/wiki/` — этот
файл только структурный указатель и жёсткие операционные правила. «Что сделано»
и «что дальше» — из [docs/wiki/INDEX.md](docs/wiki/INDEX.md) и experiment-узлов,
не из CLAUDE.md. Предметная логика пакетов — в их README, не дублировать сюда.

## Production DB and S3 — READ-ONLY (CRITICAL, no exceptions)

The Postgres database (`rc1b-hl1x61akq81cvefa.mdb.yandexcloud.net`) and the S3
bucket (`dc-project-korovas` on Yandex Object Storage) used by this project
are **production systems**, not dev/staging copies. All access from this
repository — scripts, ad-hoc queries, agents — is **read-only**. This is not
negotiable and does not get relaxed for convenience, debugging, or "just this
once" fixes.

### Postgres — forbidden

- Any data modification: `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `MERGE`
- `COPY FROM` (data-loading direction; `COPY TO` for reading is fine only via
  a read-only connection)
- Any DDL: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`
- `SET`, `RESET`, or any session/role/config mutation
- `SELECT INTO` (creates a table as a side effect)
- Data-modifying CTEs (`WITH x AS (INSERT ... RETURNING ...) ...`)
- Running migrations against the production host, from any tool or framework
- `EXPLAIN ANALYZE` (it executes the query with side effects if the query
  itself is not read-only — use plain `EXPLAIN`)

### Postgres — allowed

- `SELECT`
- Read-only `WITH` (CTEs that only `SELECT`)
- `EXPLAIN` (without `ANALYZE`)
- `SHOW`
- `VALUES`
- `TABLE`

### S3 — forbidden

- Uploading, overwriting, or deleting objects in `dc-project-korovas`
- Any write operation against the bucket from scripts or agents

### S3 — allowed

- Downloading / reading objects (research scripts under `cow_scoring/scripts/research/`)

### Enforcement

- `cow_scoring/src/cow_scoring/research/db/guard.py` validates every SQL string against a
  keyword whitelist/blacklist before it reaches the database — treat this as
  the code-level backstop, not the primary control.
- `cow_scoring/src/cow_scoring/research/db/client.py` opens connections with
  `default_transaction_read_only=on` and `connection.read_only=True`, and
  always issues `ROLLBACK` regardless of outcome — never commit.
- The scoring HTTP service (`cow_scoring`) holds **no DB credentials**. Only
  datapipe writes inference/score tables (`cow_inference_result`,
  `cow_inference_result_aggregated`, `cow_score_result`) to the remote admin DB.
- The real guarantee should eventually come from database-level permissions
  (a role with `SELECT`-only grants), not solely from application code. Until
  that role exists, the app-level guard is the only enforcement — do not
  bypass or weaken it.

If a task genuinely requires writing to the production DB or S3 (e.g. a
migration, a data fix), stop and get explicit user confirmation first — do
not route around the guard.

## Зависимости (CRITICAL)

`cow_scoring` — **pixi** (`cd cow_scoring && pixi run -e vlm …` / `-e db`).
`cow_measure` и `experiments/key_points_regression_datapipe` — **uv**
(`cd <pkg> && uv run …`). Новая библиотека — в `pyproject.toml` того пакета
(pixi feature или uv dependency-group), lock коммитится (`pixi.lock` /
`uv.lock`).

**Запрещено**: `pip install`, `pip install --user`, `conda install`, `brew
install` ради разовой задачи, а также использование системного python
(`/usr/bin/python3`, `~/miniconda3`) для скриптов проекта. Если для работы не
хватает библиотеки — это не повод её «быстро поставить», это повод завести
feature/группу в `pyproject.toml`.

`cow_scoring` pixi environments:

| Env | Зачем |
|---|---|
| `vlm` (base+vlm) | FastAPI + LiteLLM, HTTP `POST /score` и `POST /v1/score` |
| `default` / base | pytest, black, isort, pylint, mypy |
| `db` | psycopg + boto3: GT download; **не** нужен Docker-образу |

`cow_measure` — GPU FastAPI промеров (`uv sync --group gpu`). Datapipe —
отдельный `uv` проект, вызывает оба сервиса по HTTP.

## Схема данных бонитировки живёт НЕ в БД

Не ищи в Postgres колонки под показатели бонитировки — их нет и не будет.
Схема `collector` (8 таблиц) — транспортный слой. Вся предметная информация
лежит одной text-колонкой `package_session.manifest_json`
(`{package_id, project_id, form_id, form_name, form_version, created_at,
data{…}, submitted_by{…}}`); форма версионируется на стороне мобильного
приложения (data-collector, `project_id: krs-label`), а не миграциями
Postgres. Полный разбор таблиц, полей и трёх поколений формы — в
[docs/wiki/raw/2026-08-04-korovas-dataset-audit.md](docs/wiki/raw/2026-08-04-korovas-dataset-audit.md).

Ловушки, о которые уже разбивались (проверено на живых данных):

- `form_version` не отражает схему — в проде `2.1` стоит у трёх разных наборов
  полей. Определять схему только по фактическому набору ключей `data`.
- Пустая строка `''` — это отсутствие значения, а не ноль.
- Три формата дат в одной записи (`package_session.created_at` с `+00:00`,
  `manifest_json.created_at` с `Z`, `data.scan_time` без зоны).
- Баллы `*_real` и описания `*_real_desc` — парные поля, парсить вместе.

Текущее состояние датасета (43 пакета, три поколения формы, 41 пригодный для
GT) и решение строить каркас против старой схемы — в experiment-узле ниже, не
здесь.

## Текущая работа и очередь задач

Полная картина — очередь задач, критический путь, история решений — живёт в
графе знаний, не в этом файле:

- [docs/wiki/INDEX.md](docs/wiki/INDEX.md) — точка входа: decisions,
  experiments, raw sources.
- [docs/wiki/experiments/2026-08-04-vlm-scoring-harness.md](docs/wiki/experiments/2026-08-04-vlm-scoring-harness.md)
  — основной experiment-узел: T1–T6, R0–R13, очередь задач, критический путь.
- [docs/wiki/experiments/2026-08-04-knowledge-base-ingestion.md](docs/wiki/experiments/2026-08-04-knowledge-base-ingestion.md)
  — поглощение методичек в граф (D1–D6).
- [docs/wiki/log.md](docs/wiki/log.md) — append-only журнал по датам.

Не дублируй содержимое этих узлов сюда прозой «что сделано» — при завершении
шага обнови узел и `log.md`, а этот файл трогай только если поменялось само
правило (новая uv-группа, новый инвариант, новый агент/скилл).

## Hooks (`.claude/hooks/` — project-local, ported from Adagio 2026-08-06)

| Hook | Event | Purpose |
|---|---|---|
| `block-secrets.sh` | PreToolUse (Read\|Bash) | block reading `cow_scoring/.env` (real Postgres/S3 creds) — `env.example` stays allowed |
| `block-unauthorized-push.sh` | PreToolUse (Bash) | block `git push` unless `.claude/.push-authorized` holds the current HEAD SHA (user must write it after explicit confirmation); consumed single-use |
| `dev-cycle-check.sh` | Stop | if source changed since HEAD but `docs/wiki/log.md` has no entry for today, or a new PDF landed in `docs/bonitirovka_doky/` without a `docs/wiki/SOURCES.md` row, block session end |
| log entry agent (inline in `settings.json`) | PostToolUse (Edit\|Write\|MultiEdit) | after a real edit, append one line to `docs/wiki/log.md` for today if none exists yet — the actual enforcement mechanism behind "Задачи ведутся в git" below, so this isn't just a written rule |

These are intentionally scoped-down vs Adagio's originals: no full-lint-on-every-Stop (`uv run pytest` / mypy across cow_measure plus vendored Metric3D takes minutes — too slow to run on every turn), and no spec-graph checks (this repo has none, see `spec-graph.md`).

## Задачи ведутся в git, а не в трекере сессии (CRITICAL)

Любая задача, договорённость или следующий шаг **записывается в файл и
коммитится**. Трекер задач Claude Code (`TaskCreate` / `TaskUpdate`) —
вспомогательный: он живёт внутри одной сессии, не попадает в репозиторий и
разрывает граф знаний. Задача, существующая только там, для следующей сессии не
существует.

Куда писать:

- **очередь задач** — таблица «Очередь задач» в
  [experiment-узле](docs/wiki/experiments/2026-08-04-vlm-scoring-harness.md):
  идентификатор, что блокирует, проверяемый критерий готовности;
- **развёрнутая постановка** — отдельный подраздел там же, рядом с остальными
  T-/R-задачами;
- **одна строка в** [docs/wiki/log.md](docs/wiki/log.md) — журнал append-only;
- **ссылки** в `docs/wiki/INDEX.md` и в `## Related` затронутых узлов, иначе
  узел выпадает из графа и его не видно ни в Obsidian, ни следующей сессии.

`TaskCreate` допустим только как зеркало уже закоммиченной задачи, для
отображения прогресса внутри сессии. Заводить задачу **сначала** в трекере и
надеяться, что она переживёт сессию, — ошибка. Порядок всегда: файл → коммит →
(опционально) трекер.

Это же правило распространяется на решения (`docs/wiki/decisions/`) и на
разбор внешних источников (`docs/wiki/raw/`): пока не закоммичено — не
сделано.

## Граф знаний и его аудит

Предметные знания живут в `docs/wiki/`. Полнота графа — не гигиена, а
предпосылка корректных баллов: промпт собирается из рубрики, и пропущенный
источник означает неверную оценку реального животного.

- [docs/wiki/SOURCES.md](docs/wiki/SOURCES.md) — **реестр источников**: каждый
  файл в `docs/bonitirovka_doky/` обязан иметь здесь строку со статусом
  покрытия (`не начат` / `частично` / `полностью` / `нерелевантен`), даже если
  он ещё не разобран. Появился новый документ — строка заводится сразу, до
  всякого разбора. **Приоритет источников**: нормативный только приказ N 270;
  где числа расходятся с методичками, приказ побеждает, расхождение
  фиксируется явно.
- Агент **`knowledge-graph-auditor`** (`.claude/agents/`) — восемь проверок:
  полнота реестра, честность статусов, связность узлов, висячие ссылки,
  отставание журнала, наличие задач в git, зафиксированность расхождений между
  источниками, неизменяемость тела `raw/`. Запускать после разбора любого
  источника и перед каждым коммитом, меняющим `docs/wiki/`. Агент только
  читает.

## Методички кроме приказа

Пользователь присылает печатные источники PDF-ами в `docs/bonitirovka_doky/`.
**Эти PDF не коммитятся** (решение 2026-08-04, единственное исключение —
приказ N 270, он в git через LFS как `.docx`) — из них извлекается содержимое
в `docs/wiki/raw/`, а файлы остаются untracked.

Статус разбора каждого источника, что взято и что нет, независимость/зависимость
источников друг от друга — всё в [docs/wiki/SOURCES.md](docs/wiki/SOURCES.md).
Не дублируй это здесь.
