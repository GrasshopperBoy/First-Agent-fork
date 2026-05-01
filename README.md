# First-Agent

Репозиторий, в котором я собираю **собственного LLM-агента** вместе с
[devin.ai](https://devin.ai). Этот README — единый ориентир: что за проект, где
он сейчас и куда движется.

> **Статус:** `scaffolding complete → first module creation`.
> Feedback-loop поднят; следующий модуль — deterministic chunker для
> Mechanical Wiki / brain v0.1.

---

## 1. Зачем это

**First-Agent** — учебно-исследовательский проект по созданию автономного
LLM-агента. Цель — пройти весь путь от формулировки задачи до работающего
прототипа: выбрать модель, придумать архитектуру, собрать минимальный набор
инструментов, завести память между сессиями, написать первый модуль и
раскрутиться оттуда итеративно.

Опора — экосистема Devin (как референс для хороших практик) и официальные
гайды Cognition:

- [When to use Devin](https://docs.devin.ai/essential-guidelines/when-to-use-devin)
- [Coding Agents 101](https://devin.ai/agents101)
- [docs.devin.ai](https://docs.devin.ai)

---

## 2. Текущее состояние

- [x] Репозиторий создан.
- [x] Базовая вики про работу с Devin собрана (см. [`docs/`](./docs/README.md)).
- [x] Слот под долговременную память (ADR, промпты, обзор) создан
      ([`knowledge/`](./knowledge/README.md)).
- [x] Заполнено видение проекта
      ([`knowledge/project-overview.md`](./knowledge/project-overview.md)).
- [x] Проведено исследование по ключевым развилкам — итог в
      [`knowledge/llms.txt`](./knowledge/llms.txt) (флэт-индекс всех
      артефактов) и в [`knowledge/research/`](./knowledge/research/).
- [x] Приняты **ADR-1..ADR-6** (см.
      [`knowledge/adr/README.md`](./knowledge/adr/README.md));
      ADR-7 зарезервирован под inner-loop / tool-contract.
- [x] Поднят тулинг (lint/types/tests/CI/pre-commit, `Makefile`).
- [x] Зафиксирована convention для stacked / sequenced PR'ов
      ([`AGENTS.md` §Stacked / sequenced PRs](./AGENTS.md#stacked--sequenced-prs)).
- [ ] Написан первый модуль (chunker для Mechanical Wiki).

Нулевое, но важное: первый feature-модуль пишем только после feedback-loop.
Этот gate закрыт; дальше идёт chunker для v0.1 brain.

---

## 3. Scope — что входит и что не входит

Полная версия — в
[`knowledge/project-overview.md`](./knowledge/project-overview.md) §4–§5.
Краткая выжимка:

### В scope (v0.1)

- **UC1** — coding + PR-write end-to-end (FA + 1–2 controlled-list
  репозитория пользователя).
- **UC3** — local-docs-to-wiki (`fa ingest <path-or-url>`,
  chunk-aware retrieval, Q&A).
- Static role-routing LLM tiering (Planner / Coder / Debug),
  mechanical-wiki memory (no embeddings/graph в v0.1), SQLite FTS5
  индекс, sandbox + path allow-list для тулов.

### Вне scope (v0.1)

- **UC2** continuous multi-source research — best-effort.
- **UC4** multi-user Telegram chat — deferred.
- **UC5** semi-autonomous multi-LLM research/experiment — deferred
  (см. [ADR-1 Amendment 2026-05-01](./knowledge/adr/ADR-1-v01-use-case-scope.md)).
- Production-деплой, мульти-тенантность, биллинг, собственный веб-UI,
  обучение/дообучение моделей, агент-общего-назначения «на всё».

---

## 4. После scaffolding'а — первый модуль

Подробнее в [`docs/workflow.md`](./docs/workflow.md). Коротко:

1. **Scaffolding.** `pyproject.toml`, `ruff` + `mypy` + `pytest`,
   CI на GitHub Actions, `Makefile`, `pre-commit` — сделано.
2. **Первый модуль.** Chunker для Mechanical Wiki:
   `src/fa/chunker/`, `Chunk` dataclass, `Chunker` Protocol,
   `CompositeChunker`, `universal-ctags` для кода и `markdown-it-py`
   для Markdown / plain text. Ручная проверка: `fa chunk <path>`.
   Контракт — в
   [ADR-5](./knowledge/adr/ADR-5-chunker-tool.md);
   sample-tests — в
   [`knowledge/research/chunker-design.md` §8](./knowledge/research/chunker-design.md#8-sample-test-plan-pre-implementation).
3. **Далее — итеративно.** Каждый модуль = отдельный PR. Каждое
   значимое решение = ADR.

---

## 5. Как работать с этим репо

Полный inventory всех документов — в
[`knowledge/llms.txt`](./knowledge/llms.txt) (one-fetch индекс,
[llmstxt.org](https://llmstxt.org/) convention). Конвенции по
структуре и работе — в [`AGENTS.md`](./AGENTS.md).

Для нового человека / агента:

1. Прочитать [`AGENTS.md`](./AGENTS.md) — repo conventions, PR
   checklist, query routing.
2. Прочитать [`knowledge/llms.txt`](./knowledge/llms.txt) — карта
   репо в одном fetch'е.
3. Просмотреть [`knowledge/project-overview.md`](./knowledge/project-overview.md)
   — что v0.1 ships и что non-goal.
4. Просмотреть индекс ADR — [`knowledge/adr/README.md`](./knowledge/adr/README.md).
5. Проверить [`HANDOFF.md`](./HANDOFF.md) — текущий snapshot
   состояния репо для cross-LLM сессий.

Дальше — по необходимости (ADR / research-нота / промпт). Не нужно
загружать всё в контекст сразу; routing-table в
[`AGENTS.md` §Query Routing](./AGENTS.md#query-routing).

---

## 6. Полезные ссылки

**Официальные:**

- [docs.devin.ai](https://docs.devin.ai)
- [When to use Devin](https://docs.devin.ai/essential-guidelines/when-to-use-devin)
- [Coding Agents 101](https://devin.ai/agents101)

**Внутри репо:**

- [`AGENTS.md`](./AGENTS.md) — конвенции и инструкции для AI-агентов.
- [`HANDOFF.md`](./HANDOFF.md) — snapshot состояния для cross-LLM сессий.
- [`docs/README.md`](./docs/README.md) — вики по работе с Devin.
- [`knowledge/README.md`](./knowledge/README.md) — как устроена память
  проекта (frontmatter schema, конвенции, supersession-rule).
- [`knowledge/llms.txt`](./knowledge/llms.txt) — one-fetch индекс
  всех документов.
- [`knowledge/adr/README.md`](./knowledge/adr/README.md) — индекс ADR.

---

*Статус документа — living. Правится по мере изменения состояния
репо; последняя ревизия — 2026-05-01 (см. git history).*
