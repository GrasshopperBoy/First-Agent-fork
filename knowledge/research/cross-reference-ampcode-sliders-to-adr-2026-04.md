---
title: "Cross-reference review — ampcode + SLIDERS notes vs ADR-1..5 (2026-04-29)"
compiled: "2026-05-01"
# Originally compiled 2026-04-29; date bumped to 2026-05-01 when §11
# Q-1 / Q-2 supersession blockquotes were added (citing 2026-05-01
# events). Per AGENTS.md rule #4 `compiled:` ≥ all dates cited in text.
# Original compile date preserved in title and in commit history.
source:
  - knowledge/research/how-to-build-an-agent-ampcode-2026-04.md
  - knowledge/research/sliders-structured-reasoning-2026-04.md
  - knowledge/adr/ADR-1-v01-use-case-scope.md
  - knowledge/adr/ADR-2-llm-tiering.md
  - knowledge/adr/ADR-3-memory-architecture-variant.md
  - knowledge/adr/ADR-4-storage-backend.md
  - knowledge/adr/ADR-5-chunker-tool.md
  - knowledge/project-overview.md
  - docs/architecture.md
  - docs/workflow.md
  - knowledge/research/agent-roles.md
  - knowledge/research/memory-architecture-design-2026-04-26.md
  - knowledge/research/chunker-design.md
  - knowledge/research/agentic-memory-supplement.md
  - knowledge/research/llm-wiki-critique-first-agent.md
chain_of_custody: >
  Этот файл — синтез поверх двух недавно вмердженых research-нот
  (ampcode, SLIDERS) и пяти принятых ADR. Все factual-цифры по
  ampcode и SLIDERS уже верифицированы в parent-нотах
  (chain_of_custody там); здесь — кросс-проверка против ADR и
  consequent-проектные предложения, не пересказ источников.
  Любые цитаты из ampcode-статьи и SLIDERS-paper вынесены в
  английские блоки и помечены секцией parent-ноты.
status: research
tier: draft
supersedes: none
extends: []
related:
  - knowledge/adr/ADR-1-v01-use-case-scope.md
  - knowledge/adr/ADR-2-llm-tiering.md
  - knowledge/adr/ADR-3-memory-architecture-variant.md
  - knowledge/adr/ADR-4-storage-backend.md
  - knowledge/adr/ADR-5-chunker-tool.md
  - knowledge/research/how-to-build-an-agent-ampcode-2026-04.md
  - knowledge/research/sliders-structured-reasoning-2026-04.md
  - knowledge/research/agent-roles.md
  - knowledge/research/memory-architecture-design-2026-04-26.md
  - knowledge/research/chunker-design.md
links:
  - "../adr/ADR-1-v01-use-case-scope.md"
  - "../adr/ADR-2-llm-tiering.md"
  - "../adr/ADR-3-memory-architecture-variant.md"
  - "../adr/ADR-4-storage-backend.md"
  - "../adr/ADR-5-chunker-tool.md"
  - "./how-to-build-an-agent-ampcode-2026-04.md"
  - "./sliders-structured-reasoning-2026-04.md"
  - "./agent-roles.md"
  - "./memory-architecture-design-2026-04-26.md"
  - "./chunker-design.md"
mentions:
  - "ampcode"
  - "Thorsten Ball"
  - "SLIDERS"
  - "Stanford OV AL"
  - "Anthropic"
  - "Claude"
  - "Mem0"
  - "Variant A"
  - "Variant D"
confidence: opinion
claims_requiring_verification:
  - "Утверждение, что mid-tier OSS Coder (Nemotron 3 Super / Qwen 3.6 27B
    из ADR-2) хуже Claude 3.7 справляется с string-replace edit-форматом —
    эмпирически не проверено. Это риск, не факт. До implementation PR
    нужен фикстурный прогон 5–10 правок на каждой целевой модели."
  - "Оценка «cheap to add provenance fields сейчас» опирается на
    предположение, что v0.1 chunker ещё не написан и схема `chunks`
    ещё не залита миграцией. На момент 2026-04-29 это так
    (Phase S завершён, модулей в `src/fa/` нет). Если на момент
    чтения этой ноты chunker уже существует — оценка стоимости
    другая."
  - "Предложение про `tool_protocol: native | prompt` в models.yaml
    основано на предположении, что часть OSS-моделей в ADR-2
    (Nemotron 3 Super, Qwen 3.6 27B) к моменту implementation
    либо умеет, либо не умеет native tool-calling — это надо
    проверять на каждом конкретном слаге через provider, не
    утверждать заранее."
  - "Все архитектурные рекомендации §10 — input для будущих ADR,
    не сами ADR. Решения принимает пользователь."
---

# Cross-reference review — ampcode + SLIDERS vs ADR-1..5

> **Статус:** research note, 2026-04-29.
> **Что внутри:** систематический проход по двум недавно вмердженым
> research-нотам (ampcode, SLIDERS) и пяти принятым ADR (ADR-1 ..
> ADR-5), с явным выпиской того, **где исследования усиливают
> текущую архитектуру**, **где обнажают пробелы** и **где
> намечают тонкие натяжки**, которые лучше закрыть до старта
> implementation-фазы (memory + chunker + agent loop).
>
> **Эта нота не предлагает менять ADR.** Она готовит структурированный
> input — пронумерованные рекомендации (§10) и открытые вопросы
> (§11), на которые принимает решение проектный лид. Сами правки в
> ADR (если они нужны) — отдельный PR после согласования.
>
> **Adressovano:** будущему Architect/Coder-агенту FA, реквестеру
> и человеку-ревьюеру PR. Форма — нумерованные блоки, явные таблицы
> mappinga, явные TL;DR на каждом разделе.

---

## 0. TL;DR — пять выводов на одной странице

1. **ADR-1..5 совместимы с обеими research-нотами**: ничего
   из ampcode/SLIDERS не отменяет ни одной принятой ADR-decision.
   Variant A (mechanical wiki, ADR-3) сохраняется. Static role
   routing (ADR-2) сохраняется. SQLite FTS5 (ADR-4) и
   universal-ctags + markdown-it-py (ADR-5) тоже не меняются.
2. **Главный gap — на стыке ADR-1/ADR-2 и inner-loop'а**: ни одна
   ADR не описывает **Tool registry contract**, **tool sandbox /
   path allow-list**, **tool-protocol negotiation (native vs
   prompt-only)**, **input validation** и **structured tool-call
   audit log**. Все пять — обязательные для UC1 (ingest → search →
   edit → push → PR), и все пять отсутствуют в текущем плане
   Phase M. Подходящая форма закрытия — один новый ADR
   («Agent inner-loop and tool contract for v0.1») и
   небольшой не-ADR style-guide для tool-описаний.
3. **SLIDERS подсказывает дешёвые v0.1 расширения схемы chunks
   (ADR-4)**: добавить `parent_title`, `breadcrumb`, `byte_start`,
   `byte_end` уже в v0.1, чтобы при росте корпуса в v0.2 у нас
   была готовая база для extraction-layer (Variant D), без
   полного re-index'а. Это additive change в ADR-4 §Decision —
   не «новая ADR», а уточнение полей.
4. **SLIDERS-style extraction layer существует как конкретная
   v0.2 опция**, которую стоит зарезервировать в ADR-3 как
   **Variant D** — рядом с уже существующими A / B / C. На v0.1
   это ничего не меняет; на v0.2 это даёт явный roadmap-указатель
   и экономит «ещё одну research-фазу при выборе следующего
   ADR».
5. **Натяжка между agent-roles.md (4 роли: Specifier, Planner,
   Executor, Critic) и ADR-2 (4 роли: Planner, Coder, Debug,
   Eval)** не разрешена. Ampcode показывает, что современные
   модели self-correct через tool feedback — отдельный per-turn
   Critic не обязателен. SLIDERS — наоборот — добавляет
   per-question structured roles (extractor, reconciler,
   answer-synth). Это не противоречие, а **два разных горизонта**:
   v0.1 inner-loop без Critic, v0.2 extraction layer с Critic-
   like reconciler. Полезно явно зафиксировать.

Подробности — ниже. Чёткое разделение: §2–§4 — ampcode-side,
§5–§7 — SLIDERS-side, §8 — cross-cutting, §9 — тонкие
натяжки в текущей архитектуре, §10 — пронумерованные
рекомендации, §11 — открытые вопросы под решение лида,
§12 — что эта нота **намеренно не покрывает**.

---

## 1. Scope, метод

**Coverage.** Все пять ADR целиком (не summary); обе только что
вмердженые research-ноты (`how-to-build-an-agent-ampcode-...`,
`sliders-structured-reasoning-...`); контекстные docs
(`project-overview.md`, `docs/architecture.md`,
`docs/workflow.md`); связанные research-ноты
(`agent-roles.md`, `memory-architecture-design-2026-04-26.md`,
`chunker-design.md`, `agentic-memory-supplement.md`,
`llm-wiki-critique-first-agent.md`).

**Не покрыто.** Сам код (`src/fa/` пуст); собственные perf /
cost benchmark'и (все цифры заёмные); security как формальный
domain (sandbox — на уровне policy); multi-agent / Workforce /
handoff паттерны (`agent-roles.md` §3.7-3.10) — v0.2+.

**Метод.** Для каждого паттерна в parent-нотах задаются три
вопроса: (1) совместим ли с ADR? (2) подкреплён в ADR /
project-overview или обнажает пробел? (3) менять/добавлять до
Phase M или v0.2 carry-forward? При ответе «до Phase M» —
рекомендация в §10 с cost-tag (cheap | medium | expensive).

**Ограничения.** Single-pass; upstream-репо ampcode и
stanford-oval/sliders не разбирались построчно; фикстурные
тесты не запускались («string-replace проседает» — гипотеза,
не measurement); peer-review от second LLM не привлекался.

---

## 2. Ampcode → ADR-1 (UC1 use-case scope)

### 2.1 Совместимость

ADR-1 §Decision §«Concrete v0.1 in-scope list» утверждает
end-to-end UC1:

> ingest folder → search → edit code → push branch → open PR
> via `gh` CLI.

Ampcode-нота §1 (TL;DR) утверждает, что весь этот цикл
помещается в ~400 строк Go и три tool'а (`read_file` /
`list_files` / `edit_file`). Это **прямая поддержка ADR-1**:
acceptance scenario действительно реализуем за минимальный
LOC при наличии inner-loop'а и трёх базовых tool'ов.

→ **Совместимо. Ампкод — empirical evidence**, что UC1 в
указанной формулировке — не overscoped.

### 2.2 Что ADR-1 НЕ покрывает (gap, который обнажает ampcode)

ADR-1 описывает **что** должно быть в v0.1, не **как** оно
склеивается. Именно эту склейку даёт ampcode-нота. Конкретно:

| ADR-1 говорит | Ampcode добавляет | Закрыт ли gap? |
|---|---|---|
| «ingest folder» | `list_files` tool с trailing-slash-конвенцией для директорий | **Нет** — нет ADR на tool registry |
| «search» | grep + `read_file` (и в будущем — FTS5-tool) | **Частично** — ADR-4 покрывает FTS5 на storage-уровне, но не tool-API |
| «edit code» | `edit_file` через string-replace | **Нет** — нет ADR на edit-shape (string-replace vs diff vs AST) |
| «push branch → open PR» | в ампкоде **отсутствует** — это вне scope трёх базовых tool'ов | **Нет** — нужен 4-й tool `git_push_pr` |

Ампкод-нота §9 (Concrete actions for FA) предлагает три PR:
loop, tool registry, и три базовых tool'а — но **не покрывает
git/PR-tool**. Это означает, что для UC1 acceptance v0.1 нужна
четвёртая утилита, которая **не описана** ни в ампкоде, ни в
ADR. Фактически — это `gh` CLI shell-out tool с allow-list по
`~/.fa/repos.toml` (project-overview §4 уже намекает на этот
allow-list).

→ **Gap-1 (ADR slot):** «Inner-loop + tool registry contract»
+ **Gap-2 (toolset):** «git/PR shell-out tool with repo
allow-list». Оба для v0.1, оба до Phase M.

### 2.3 Best-effort UC2 (LLM-fan-out на top-k чанков)

ADR-1 §Decision §«Concrete v0.1 in-scope list»:

> UC2 best-effort: LLM-fan-out на top-k chunks for cross-source
> questions; no graph layer, no special infra.

Ампкод об UC2 не говорит. Логика «fan-out на top-k chunks»
в терминах ампкода — это **multiple `read_file` calls in
sequence**, агрегированных через conversation. На однопоточном
loop'е это медленно, но работает. Ампкод-нота §8 пункт 2
честно фиксирует: статья не разбирает parallel tool calls;
у нас они потенциально дают +N× ускорение.

→ **Не conflict, но напоминание**: при имплементации loop'а
(Gap-1) сразу учесть параллелизуемость `read_file`'ов.

### 2.4 UC3 (local docs → wiki)

UC3 **в ампкод-нот не упомянут**. UC3 — это в основном
write-time pipeline (chunker → FTS5), который ADR-3/ADR-4/ADR-5
покрывают. Ампкод даёт `read_file` (полезно агенту, который
отвечает на вопросы по ingested wiki), но не chunker и не
storage. Это нормально — ampcode не претендует на это.

→ **Совместимо.** UC3 — отдельный pipeline, ампкод его не
ломает.

---

## 3. Ampcode → ADR-2 (LLM tiering)

### 3.1 Совместимость на уровне абстракций

Ампкод-нота §7 даёт mapping-таблицу:

| Уровень FA | Что у Ball'а | Что в FA уже есть | Что достроить |
|---|---|---|---|
| Inner-loop (Coder/Debug) | ~50 строк | ADR-2 указывает роли | ADR-2 не описывает цикл |
| Tool registry | Go-struct + jsonschema | нет | Добавить `Tool`-Protocol |
| Conversation accumulator | локальный slice | implicit | Добавить `Session.conversation` |
| Memory | отсутствует | ADR-3 | дополнительно к ампкоду |
| Role-routing | один Claude | ADR-2 | каждая роль — свой client |

ADR-2 фиксирует **выбор моделей и провайдеров**; ампкод
фиксирует **что внутри одного role-call'а**. Они на разных
уровнях, и это правильно: ADR-2 — про tier, ампкод — про
loop. Совместимо.

### 3.2 Tension: edit-format зависит от модели

Ампкод-нота §5.3 (`edit_file`) явно цитирует:

> *«Claude 3.7 loves replacing strings (experimentation is how
> you find out what they love or don't), so we're going to
> implement edit_file by telling Claude it can edit files by
> replacing existing text with new text.»*

ADR-2 §Decision routing-таблица:

- **Coder** → mid-tier OSS (Nemotron 3 Super / Qwen 3.6 27B).
- **Debug** → DIFFERENT top-tier OSS / top tier (через
  AnyProvider / OpenRouter).

То есть **Coder в подавляющем большинстве turn'ов — НЕ Claude
3.7**. У нас нет данных, насколько хорошо Nemotron-3 Super /
Qwen 3.6 27B работают со string-replace edit-форматом. Ампкод
прямо предупреждает (claims_requiring_verification §):

> «Утверждение «Claude 3.7 любит string-replace» — может
> протухнуть при смене модели (Claude 4, Sonnet/Opus, Haiku) или
> провайдера (OpenRouter / OSS). Перед тем как фиксировать
> `edit_file` через string-replace в FA, нужно прогнать 5–10
> фикстурных правок на каждой целевой модели из ADR-2.»

→ **Tension-1**: edit-shape выбран ампкодом эмпирически под
Claude. На моделях ADR-2 он не валидирован. Это не
блокер ADR-2, но **хард-гейт** перед PR-ом «implement
edit_file». Конкретный mitigation — фикстурный прогон
(см. §10 рекомендация R-3).

### 3.3 Gap: native tool-calling vs prompt-only

Ампкод-нота §6 («wink/raise arm») показывает, что tool-use
**можно эмулировать через prompt** для моделей без native
tools. ADR-2 предполагает, что все три провайдера дают
native tool-calling, но **не различает это явно**:

- Anthropic — native (Claude tool_use blocks).
- OpenRouter — depends on the underlying model. GLM 5.1 / Kimi
  2.6 / Mimo 2.5 — у разных свой статус. Нужно проверять
  per-model.
- Local vLLM (Coder через Nemotron / Qwen) — depends on
  fine-tune; Qwen 3 Coder seria умеет, Nemotron 3 Super
  частично.

→ **Gap-3**: ADR-2 не имеет поля `tool_protocol: native | prompt`
в models.yaml. Это не conflict — это пропущенный спот.
Mitigation — ADR-2 amendment или новая mini-ADR на эту
тему. Cheap to add.

### 3.4 Eval role в ADR-2 vs critic в ампкоде

ADR-2 имеет **четыре роли** (Planner / Coder / Debug / Eval).
**Eval** в ADR-2 — это **LLM-as-judge offline baseline**
(version-pinned, см. §Consequences «judge: role being
version-pinned»). Это **не** per-turn critic.

Ампкод-loop не имеет critic вообще: модель сама решает,
надо ли продолжать tool-use или ответить пользователю
(пустой `tool_use` content в ответе → loop returns control).
Это «self-correcting через tool feedback» паттерн — модель
сама видит результат своего же `read_file`, корректирует
себя.

`agent-roles.md` §5.1 предлагает **другие четыре роли**:
Task Specifier (расплывчатый ввод → prompt, CAMEL §3.5),
Planner (specified task → шаги), Executor (шаг плана; ReAct
внутри), Critic/Reflector (per-turn structured feedback,
CAMEL §3.6 + Reflexion).

Маппинг ADR-2 ↔ agent-roles.md: Planner=Planner и
Coder≈Executor совпадают; Debug (top-tier escalation) и Eval
(offline LLM-as-judge) — только в ADR-2; Task Specifier и
Critic/Reflector — только в agent-roles.md. Это не
противоречие, а несогласованность: ADR-2 про выбор моделей,
agent-roles.md про pipeline ролей.

Ампкод свидетельствует: **per-turn Critic не обязателен**
(модель self-corrects через tool feedback). Это поддерживает
позицию, что в v0.1 inner-loop'е достаточно одной роли (Coder
= Executor = LLM-в-loop'е), без Critic. Critic появляется
позже, когда мы добавим reflexion-цикл (cм. agent-roles.md
§4.7) — это v0.2-кандидат.

→ **Tension-2**: ADR-2 (4 роли по тиерам) и agent-roles.md
(4 роли по pipeline-функции) — не несовместимы, но
**не consolidated**. v0.1 implementer должен знать, что
он строит: одну единственную Coder-роль с loop'ом (по ADR-2
+ ампкоду) или 4-ролевой pipeline (по agent-roles.md). Это
надо явно зафиксировать. Рекомендация — §10 R-7.

### 3.5 Static role routing vs ampcode stand-alone agent

ADR-2 §Decision: «no auto-escalation; Coder fails loudly on
hard tasks». Ампкод-агент — single-LLM, один Claude на
всё. Из этого ADR-2 — это **обёртка вокруг ампкод-агента**:
для каждой роли свой Claude/Nemotron/Qwen-instance с
собственным tool-set'ом. Совместимо.

Тонкий момент: **общается ли Planner с Coder**? ADR-2 этого не
описывает. По умолчанию — да, через какой-то `task_handoff`
паттерн. Но в ампкод-loop'е такого нет — там одна модель.
Это и есть tension-2 в раскрытом виде.

→ **Не conflict**, но требует имплементационного выбора:
один loop (один LLM, который может сам менять «mode»
prompt'ами) или несколько loop'ов с handoff (CAMEL-стиль).
Это §10 R-7.

---

## 4. Ампкод-нота §8 — что НЕ покрыто (cross-cutting gaps в FA)

Ампкод-нота §8 «Что в статье намеренно опущено» содержит **семь
пунктов**, которые ампкод-paper не покрывает. Каждый из них —
это потенциальный gap в FA. Проверим, какие закрыты ADR'ами,
какие — нет.

| # | Аспект (по ампкоду) | Текущее покрытие в FA | Gap? | v0.1 надо? |
|---|---|---|---|---|
| A1 | **Безопасность tool'ов / sandbox** | project-overview §4 «PR-write restricted to FA itself + 1-2 user repos via ~/.fa/repos.toml»; ничего про read-side `read_file` allow-list, ничего про edit-allow-list для notes/ vs ~/.ssh | **Да** | **Да** (UC1 включает edit на disk) |
| A2 | **Параллельные tool calls** | ничего в ADR | **Да** | Желательно (UC1 read N файлов перед edit'ом) |
| A3 | **Streaming** | ничего в ADR; project-overview §6 latency budget «≤10s p95» намекает, что streaming нужен для UX | **Да** | Желательно для UX, не блокер для UC1 acceptance |
| A4 | **Контроль контекста / token-truncation** | ADR-3 hot.md→архив (но без конкретики truncate-стратегии в loop'е) | **Частично** | Важно для UC1 на больших проектах |
| A5 | **Schema validation tool inputs** | ничего; ампкод в Go просто Unmarshal'ит | **Да** | Важно для OSS-моделей (mid-tier hallucinates schema) |
| A6 | **Tool-call audit log** | project-overview §4 «session log → notes/sessions/<date>.md»; форма tool-calls в этом логе не описана | **Частично** | Важно для UC1 review (PR содержит trail) |
| A7 | **Cancellation (Ctrl-C)** | ничего | Минор | v0.2-deferable |

Из семи — пять (A1, A2, A4, A5, A6) **на границе блокирующих
для UC1 acceptance**. Они не отменяют ADR-1..5, они
**добавляют слой**, который сейчас в архитектуре пропущен.

→ **Cross-cutting gap-cluster G-A**: «inner-loop hardening».
Form: один новый ADR «Agent inner-loop and tool contract for
v0.1» (см. §10 R-1).

### 4.1 Sandbox / path allow-list (A1) — почему это критично

UC1 acceptance scenario: «ingest folder → search → edit code
→ push branch → open PR». Без sandbox `edit_file('/etc/passwd',
old_str='root', new_str='hacked')` — валидный tool-call с
точки зрения ампкод-loop'а.

В FA:

- **Read side** (`read_file`, `list_files`) — допустима
  read-allow-list (можно читать любой репо, на котором стоит
  `git`-ремоут, но **не** домашний каталог пользователя
  целиком; SSH ключи, GPG-keystore — за пределами).
- **Write side** (`edit_file`) — **должна** быть write-allow-list:
  только пути внутри `~/.fa/notes`, `~/.fa/state`, и тех
  репо, что в `~/.fa/repos.toml`. Любое edit-обращение вне
  списка → tool возвращает error «forbidden path».

Это **policy**, не код проверки путей. Имплементация —
тривиальная (`pathlib.Path.resolve` + `is_relative_to`-чек),
но **policy-формулировка** должна быть в ADR, не в коде.

→ См. §10 R-2.

### 4.2 Parallel tool calls (A2)

Anthropic API позволяет модели возвращать N `tool_use` блоков
в одном `assistant`-сообщении. Стандартный паттерн:

```python
# pseudo, по мотивам ампкода + Anthropic docs
for tool_call in assistant_message.content:
    if tool_call.type == "tool_use":
        result = run_tool(tool_call)
        results.append(ToolResult(id=tool_call.id, content=result))
# отправляем все результаты в одном `user`-блоке
conversation.append({"role": "user", "content": results})
```

Эта тривиальная реализация **последовательна**. Если модель
просит `read_file` × 5 — пять последовательных файловых
чтений. На SSD-системе разница ничтожна; на сетевом storage
— может быть. Для **остальных tool'ов** (особенно `execute_sql`
в v0.2 SLIDERS-style) параллелизм может давать значимый speed-up.

Переход на `asyncio.gather` за сравнимой с ампкодом
сложностью. Не блокер, но **знак ставится в loop ADR**: «tool
calls в одном assistant-блоке должны быть параллелизуемы по
дизайну».

→ См. §10 R-1, подпункт.

### 4.3 Schema validation (A5) — pydantic vs dataclass vs raw json

Ампкод (Go): `json.Unmarshal` → struct. Если модель прислала
`"path": 42` (int вместо string) — Go раскастит / упадёт
clearly. На Python с `dataclass` без validation — будет
runtime AttributeError позже. На OSS-моделях это
случается чаще.

Решение — валидация на границе: либо `pydantic.BaseModel.
model_validate(json)` (с автоматической coercion), либо
`jsonschema` library. Pydantic уже фактический стандарт в
Python-LLM-tooling (Instructor, LangChain v0.3+). Ничего не
надо изобретать.

→ См. §10 R-1, подпункт.

---

## 5. SLIDERS → ADR-3 (Variant A / memory architecture)

### 5.1 Совместимость

ADR-3 §Decision: **Variant A «Mechanical Wiki»** — filesystem
canon, deterministic chunker, FTS5 BM25, no embeddings, no
graph, no Mem0 в v0.1.

SLIDERS-нота §6 «Mapping на ADR FA» утверждает буквально:

> «SLIDERS не требует изменений ни в одном из принятых ADR.
> Но он рисует чёткую картину что добавляется, когда v0.1
> закроется и начнётся v0.2.»

И ADR-3 §«Explicit non-goals (for v0.1)»:

> «No graph extraction (typed edges, PPR). No embeddings or
> vector index. No Mem0 4-op API.»

SLIDERS — это **не** graph, **не** embeddings, **не** Mem0
4-op. SLIDERS — это **structured extraction в реляционную
БД с reconciliation**. Это **другая ось** относительно ADR-3
non-goals. То есть формально ADR-3 не запрещает SLIDERS-style
расширение. **Совместимо.**

### 5.2 Где SLIDERS усиливает Variant A

ADR-3 §Cons «Multi-hop reasoning weak — UC2 falls back to
LLM-fan-out (acceptable per ADR-1)». SLIDERS §1 (TL;DR)
показывает paper-level evidence того, что **на корпусах
свыше ~360k токенов наивный full-context / RAG не справляется**:

| Benchmark | Tokens | Лучший baseline | SLIDERS | Δ |
|---|---|---|---|---|
| FinanceBench | ≤360k | 84.67 | 89.33 | +4.7 |
| WikiCeleb100 | 3.9M | 59.80 | 78.91 | +19.1 |
| FinQ100 | 36M | 31.41 | 55.22 | +23.8 |

Это **support для ADR-1** «UC2 best-effort на v0.1 корпусе»
— пока корпус FA маленький, fan-out работает. Но как только
UC3 раскачает `notes/inbox/` или появится UC2 с реальным
research-corpus'ом, **fan-out лопнет**.

SLIDERS-нота §2 «aggregation bottleneck» — ровно тот эффект,
который мы должны иметь в виду. Это не делает Variant A
«временно правильным выбором» — это делает его правильным
**сейчас** и даёт сигнал, **когда** надо переключаться.

→ **Не gap, поддержка ADR-3.**

### 5.3 Gap: «Variant D» отсутствует в design space

ADR-3 §«Options considered» имеет три варианта (A / B / C),
повторяющих `memory-architecture-design-2026-04-26.md` §3-§6.
Variant A — Mechanical Wiki (наш v0.1). Variant B — Hybrid
Brain (Mem0-pipeline). Variant C — Layered KG (typed edges).

SLIDERS — это **четвёртая точка design space'а**:
**Variant D — Extraction Layer** (structured extraction over
filesystem canon, реляционные `entities` / `facts` /
`provenance` / `rationale` таблицы поверх ADR-4 SQLite,
SQL-tool в registry).

Различия Variant'ов на retrieval-axis (см.
`memory-architecture-design-2026-04-26.md` §3 «Ось C»):

| Variant | Retrieval | Когда побеждает |
|---|---|---|
| A | grep + BM25 (+ vectors v0.2) | Малый корпус, UC1 lookup |
| B | Vector search + Mem0-graph | Conversational memory, UC4 multi-user |
| C | BM25 + graph traversal | Multi-hop, code call-graph |
| **D (SLIDERS)** | **SQL-loop поверх structured DB** | **Aggregation на больших corpora, UC2/UC3 при росте** |

Variant D — **не замена** Variant A, **а её апгрейд**: тот же
filesystem canon, та же SQLite-инстанция (ADR-4 §«Adding the
v0.2 vector layer is additive — same DB file, new table»),
просто **другой набор таблиц** (вместо/рядом с `chunks` —
`entities`, `facts`, `provenance`, `rationale`) и **другой
агентский паттерн над ними** (extract-once, then SQL-QA per
question).

→ **Gap-4**: ADR-3 не упоминает Variant D как явный v0.2
кандидат. Это не ошибка ADR-3 — на момент его написания
SLIDERS-нота не существовала. **Cheap fix:** add Variant D
section в ADR-3 или в parent
`memory-architecture-design-2026-04-26.md`. См. §10 R-4.

### 5.4 Натяжка: hot.md → session archive vs SLIDERS extract

`hot.md` в ADR-3 — per-session conversation summary (Markdown,
append-only, grep-retrieval, archive). SLIDERS-extract §3.5 —
per-corpus structured DB (SQL, UPSERT+reconcile, SQL-
retrieval, durable cache). Они на разных горизонтах и
дополняют друг друга. Но в `docs/architecture.md`
§«Архитектура памяти» (working/episodic/semantic/procedural)
SLIDERS-style является пятым типом «structured semantic» —
это не отражено.

→ **Не conflict**, дополнение к taxonomy. См. §10 R-4.

### 5.5 Tension: schema-induction зависит от вопроса

SLIDERS-нота §3.2 «Schema Induction»: для каждого нового
вопроса — новая схема; авторы amortize'ят (§6) через
topic-clustering. Для FA это UX-сложность: при ingestion нужен
`--topic`-флаг (`fa ingest --topic finance ./reports/`) и поле
`<topic>` в frontmatter, иначе mixed-bag corpus = дорогая
schema-induction per question.

→ **Не v0.1 блокер**, но cheap forward-compat: добавить
`topic:` (optional) в frontmatter-schema уже сейчас. См. §10 R-6.

---

## 6. SLIDERS → ADR-4 (storage backend)

### 6.1 Совместимость

ADR-4 §Decision: SQLite FTS5 как disposable index. Tables:

```text
chunks(id, path, anchor, lang, body, mtime, sha256)
chunks_fts(body, content='chunks', content_rowid='id')
meta(key, value)
```

SLIDERS использует SQL поверх extracted DB. SQLite — натуральная
платформа для обоих. ADR-4 §Cons «Adding the v0.2 vector layer
is additive — same DB file, new table, same connection pool»
явно говорит, что DB-схема расширяема. SLIDERS-extension
схемы — **точно та же модель**: добавить `entities`, `facts`,
`provenance`, `rationale` таблицы, переиспользуя тот же
`~/.fa/state/index.sqlite`.

→ **Совместимо.**

### 6.2 Gap: chunks-схема не имеет provenance-fields

SLIDERS §3.3 (Structured Extraction) хранит для каждой
extracted-cell `value` (нормализованное), `provenance`
(минимальный текстовый span) и `rationale` (LLM-generated). §8
SLIDERS-ноты предлагает вынести provenance-fields в
chunks уже в v0.1, иначе v0.2 extraction-layer потребует
full reindex.

Текущая схема ADR-4 §Decision имеет `id`, `path`, `anchor`,
`lang`, `body`, `mtime`, `sha256`. `path` ≈ `source_path` (OK);
`anchor` — это heading slug, не document title.

Не хватает четырёх полей:

- **`parent_title`** — название самого документа (frontmatter
  `title:` / первая H1 / filename для кода).
- **`breadcrumb`** — иерархия секций до anchor'и
  (например `["README", "Setup"]`); SLIDERS §3.1 называет это
  `m_L` (local metadata).
- **`byte_start` / `byte_end`** — байтовые офсеты чанка
  («минимальный текстовый span» в SLIDERS-терминах).
- **`topic`** (optional, для v0.2 amortization).

ADR-5 `Chunk` dataclass уже имеет `line_start`/`line_end`, но
ADR-4 их не хранит — это current inconsistency между ADR-4
и ADR-5.

→ **Gap-5**: cheap fix — расширить `chunks` колонками из
Chunk dataclass'а плюс четыре новых поля. См. §10 R-5.

### 6.3 Tension: scaling SQLite на extraction-layer

SLIDERS §7.4 — extracted DB для FinQ100 (36M tokens) имеет
тысячи rows в `entities`-таблице. SQLite справляется с
миллионами rows, **но** в multiple-writer-single-reader
сценарии возможны блокировки. Если в v0.2 inbox-watcher
+ extraction-pipeline + read-side SQL-tool работают
одновременно — могут быть write-amplification issues.

ADR-4 §Cons:

> «SQLite locking semantics need attention if we add a daemon
> writer + CLI reader; for v0.1 we run single-process.»

Это **правильное предостережение**. SLIDERS не отменяет его,
наоборот — усиливает: для v0.2 при extraction-layer внутри
single-process модель остаётся OK, при daemon-watcher —
надо переходить либо на WAL-mode (ADR-4 этого явно не
включает), либо на async writes c queue.

→ **Не v0.1 блокер**, но note для v0.2 ADR. В §10 как R-9.

### 6.4 Кросс-проверка: «filesystem canon stays authoritative»

ADR-4 §Decision: «Filesystem canon stays authoritative.
SQLite is a cache.» SLIDERS-style extraction layer **создаёт
вторичный канон** — `entities` / `facts` таблицы, которые
**не выводимы** из filesystem простым re-chunk'ом, потому что
extraction делал LLM. Re-extraction на тех же данных может
дать другой результат (LLM non-determinism).

Это **тонкая натяжка**: «SQLite is a cache» становится менее
правдивым при добавлении extraction layer. Технически — да,
это всё ещё cache (можно перегенерить), но **дорогой cache**:
re-extract WikiCeleb100 = 16 минут, $13. На FinQ100 — час+.

Решение SLIDERS-нот: расход **амортизируется** между
вопросами, потому что extract делается **один раз на корпус**
(не на каждый ask). Но семантически: «filesystem canon»
правда теряет свой 100%-канон-статус для extraction-layer
data.

→ **Концептуальная заметка**, не гэп: при расширении к Variant
D будет нужен новый principle «extracted DB is *durable cache*
— rebuildable but expensive». См. §10 R-4 (часть ADR-3
amendment).

---

## 7. SLIDERS → ADR-5 (chunker tool selection)

### 7.1 Совместимость

ADR-5 §Decision: universal-ctags + markdown-it-py. Symbol-level
chunks для кода, heading-level для md, file-level для config.

SLIDERS §3.1 «Contextualized Chunking»:

> «Каждому документу `d` назначаются метаданные `m_d = (m_G,
> m_L)`: `m_G` (global) — сгенерированный заголовок документа +
> краткое описание; `m_L` (local) — структурные сигналы:
> section headers, ...»

SLIDERS делает chunking **layout-aware с document-context**.
ADR-5 chunker делает chunking **symbol-aware с heading-anchor**.
Это **близко, но не совпадает**.

| | SLIDERS chunking | ADR-5 chunker |
|---|---|---|
| Granularity | section / paragraph | symbol / heading |
| Doc-level metadata | `m_G` (title + description) | **отсутствует** в эмиссии чанка |
| Section hierarchy | `m_L` breadcrumb | в `anchor` только last heading |
| Cross-doc relationships | через primary keys | через `path` (но не PK) |

→ **Не conflict**, но ADR-5 chunker эмитит **меньше metadata**
на чанк, чем SLIDERS-style требует. Это согласовано с тем,
что SLIDERS — v0.2-фичa, не v0.1. Но **forward-compat**: если
chunker в v0.1 не эмитит `parent_title` / `breadcrumb`, то v0.2
extraction layer должен будет re-chunk весь корпус.

### 7.2 Gap: ADR-5 chunker не эмитит doc-level metadata

ADR-5 §Decision Chunk dataclass — `path`, `anchor`, `lang`,
`body`, `line_start`, `line_end`. Нет `parent_title`,
`breadcrumb`, `byte_offset`. Cheap forward-compat: расширить
Chunk dataclass четырьмя полями — `parent_title`,
`breadcrumb` (tuple section-headings), `byte_start`, `byte_end`.
Эта правка additive, не breaking; ADR-5 §Decision явно
говорит «Stable interface… future TreeSitterChunker swap must
not change interface» — расширение полей без удаления back-
compat.

→ **Gap-6**: Chunk dataclass без `parent_title` / `breadcrumb`
/ `byte_offsets` потребует re-chunk при добавлении extraction
layer. См. §10 R-5.

### 7.3 ADR-5 §Consequences «500-line+ symbols become single
chunks»

ADR-5 §Cons:

> «No intra-symbol splitting: a 500-line PowerShell function
> becomes one chunk… UC1 retrieval can push more tokens into
> context than ideal.»

SLIDERS §3.1 неявно отвечает: для извлечения evidence более
важен **layout-coherent chunk** (со всем контекстом секции),
чем **maximally short chunk**. То есть **500-строчный chunk
плох для retrieval, но норм для extraction**, при условии что
он structurally meaningful (одна функция).

Это **подтверждает** ADR-5-выбор: ctags (symbol-level) —
правильное для extraction-style v0.2, не bad для v0.1
retrieval (просто token-cost-up).

Натяжка: для UC1 (read функцию перед edit'ом) 500-строчный
chunk **дорогой**. ADR-5 §«Re-evaluation triggers» предлагает
revisit'ить если retrieval recall плохой. SLIDERS на это не
влияет.

→ **Не gap.** Подтверждение ADR-5.

### 7.4 PowerShell как weak spot

ADR-5 §Cons: «PowerShell ctags support not yet end-to-end-tested
on the user's actual 1500-line `.ps1` profile». SLIDERS-paper
PowerShell не разбирает (paper целит в finance / wiki / Loong).
Это означает: **на момент v0.2 при добавлении SLIDERS layer**
у нас не будет cross-validation chunker'a на PowerShell-data
для extraction.

→ **Не gap по SLIDERS**, но напоминание: PowerShell sample-test
(ADR-5 §Cons «Sample-test is a hard gate before the
implementation PR») делать в любом случае. SLIDERS этого
требования не снижает.

---

## 8. Cross-cutting issues (на стыке ампкода + SLIDERS)

### 8.1 Provenance / chain-of-custody на уровне tools

Ампкод-нота §8 пункт 6 (audit log) и SLIDERS §3.3
(provenance) — **обе** требуют
**audit-trail для каждого LLM-генерируемого факта**.

В FA это уже частично есть:

- **Frontmatter** (`source:` / `chain_of_custody:` /
  `claims_requiring_verification:`) — **manual** провенанс для
  research-нот. Описано в `knowledge/README.md` §Conventions.
- **AI-Session trailer** в commit'ах — провенанс на уровне
  changeset'а. AGENTS.md §«Development Workflow».

Чего не хватает:

- **Tool-call audit log** — каждый `read_file` / `edit_file`
  должен иметь свой entry в session-log с argument, return
  (или summary), и timestamp.
- **Per-fact provenance** при extraction (v0.2) — `provenance`
  колонка в `facts`-таблице.

→ **Cross-cutting recommendation R-1.5**: при имплементации
loop'а сразу выводить tool-calls в session-log. См. §10 R-1.

### 8.2 Tool registry shape — общая для read и write side

Ампкод-стиль tool-registry — `{name, description, input_schema,
fn}`. SLIDERS использует SQL-loop tool — это **тот же** shape,
просто `name = "execute_sql"`, `description = "Run a SQL
SELECT against the extracted DB"`, и так далее.

Если v0.1 ADR-1 (inner-loop + tool registry) — well-designed,
то v0.2 SLIDERS-tool **просто регистрируется** в том же
registry без изменений шаблона. Это поддерживает
рекомендацию **сделать единый Tool-Protocol**, а не отдельный
для каждой роли.

→ См. §10 R-1.

### 8.3 LLM-tier разделение для extraction vs answer-synth

SLIDERS §7.2 явно показывает: **schema-induction стабильна
к выбору модели** (≤3.3 pts variance между GPT-4.1, GPT-4.1-mini,
GPT-5). То есть если в v0.2 мы делаем SLIDERS-style — schema
можно делать на mid-tier OSS (Coder), не обязательно top-tier.

Что **требует top-tier** в SLIDERS — answer-synth (final QA
агент). Это укладывается в ADR-2 как:

| SLIDERS phase | ADR-2 role candidate |
|---|---|
| Contextualized chunking | (deterministic, no LLM) |
| Schema induction | Coder (mid-tier OSS) или Planner |
| Structured extraction | Coder (mid-tier OSS) — массовые цели |
| Reconciliation | Planner (top-tier OSS) — нетривиальная классификация |
| Answer synth (SQL loop) | Planner или Debug в зависимости от сложности |

То есть SLIDERS-pipeline **естественно** расходится по тиерам
ADR-2. Никаких новых ролей **не требуется**, можно
переиспользовать существующие. Это упрощает v0.2 ADR.

→ **Поддержка ADR-2**, не gap. Заметка для будущей v0.2 ADR.

### 8.4 Privacy / PII при extraction (SLIDERS §7.5)

SLIDERS-нота §7.5:

> «Privacy / data-leak. Прямо в paper'е не обсуждается. В FA-
> контексте: schema-induction + extraction отправляют сэмплы
> документа в LLM. Если пользователь хочет UC3 над личными
> заметками, схема может включать имена / адреса / etc. ADR-2
> (LLM-tiering) должен учитывать чувствительный корпус →
> только локальная модель.»

ADR-2 не имеет понятия «sensitive corpus». project-overview
§6: «Privacy — remote API ≈ 99 % of traffic; user is OK с
TG-data going to providers in v0.2. No special data-residency
/ PII-redaction requirements в v0.1».

То есть **сейчас** privacy-aspect signed off. Но при добавлении
v0.2 extraction layer — можно ввести `privacy_tier:` в
frontmatter (`public | local-only | sensitive`) и роутить
extraction по этому полю.

→ **v0.2-deferable**, но cheap to scaffold (1-line frontmatter
field). См. §10 R-6.

### 8.5 Eval gold-set: phrase-level, не answer-level

SLIDERS-нота §8 пункт 4:

> «Eval gold-set: phrase-level gold, не just answer-level.
> SLIDERS-paper предупреждает: ошибки часто в gold. Если мы
> делаем gold-набор, делаем его с провенансом — где в
> документе живёт правильный ответ.»

ADR-1 §«Follow-up work this unlocks»:

> «LLM-as-judge eval baseline для UC1/UC3 acceptance (gstack
> scaled-down).»

ADR-1 говорит про eval-baseline в общем, не специфицирует
формат gold. SLIDERS-нота даёт конкретный совет: **phrase-level
с провенансом**. Это влияет на структуру eval-fixture'ов уже
в v0.1 (даже если экстракция layer не имплементируется).

→ **Note для eval-implementation PR**, не блокер.

### 8.6 Roles-tension consolidation

§3.4 уже подняла tension-2 (ADR-2 vs agent-roles.md). Здесь —
дополнительно: SLIDERS добавляет **per-corpus roles**:
extractor, reconciler, answer-synth. Они **не** появляются в
v0.1 inner-loop, но появятся в v0.2 extraction layer.

v0.2 mapping: к v0.1 набору (Planner / Coder / Debug / Eval)
SLIDERS добавляет четыре новых роли: **Schema-Inducer**
(mid-tier OSS), **Extractor** (mid-tier, parallel), **Reconciler**
(top-tier OSS), **SQL-Agent** (top-tier). agent-roles.md
отдельно добавляет Task Specifier и per-turn Critic.
Ни SLIDERS, ни agent-roles.md не пересекаются друг с
другом и оба — future work. В ADR-2 §Consequences явно
зафиксировать: v0.1 inner-loop — ровно четыре роли ADR-2,
никаких Specifier / Critic / Schema-Inducer / Extractor /
Reconciler / SQL-Agent. Иначе implementer попытается
реализовать «полный набор» из agent-roles.md без необходимости.

→ См. §10 R-7.

---

## 9. Тонкие натяжки в текущей архитектуре

Это места, где ADR-1..5 не противоречат research-нотам, но
где параллельное чтение всех документов вызывает
дискомфорт. Не баги, а «то, что стоит знать перед Phase M».

### 9.1 `chunker` эмитит `line_start/line_end`, ADR-4 их не хранит

См. §6.2 (gap-5). ADR-5 Chunk dataclass имеет `line_start /
line_end`, ADR-4 chunks-схема — нет. Implementer заметит и
напишет миграцию; но это **inconsistency между ADR-4 и
ADR-5** существует уже сейчас, **до** SLIDERS-нот.

### 9.2 ADR-2 Eval-роль и agent-roles.md Critic-роль

См. §3.4 (tension-2). ADR-2 говорит «Eval = LLM-as-judge,
version-pinned, offline». agent-roles.md §5.1 говорит «Critic
= per-turn, structured feedback». Это **две разные роли**, у
них **похожее имя** в общем-LLM-обиходе. Без явной
дизамбигуации implementer может перепутать.

Mitigation — терминология: ADR-2 «Eval» = «judge». «Critic» —
не v0.1 роль (deferred). Это надо зафиксировать в
`docs/glossary.md`. См. §10 R-8.

### 9.3 «PR-write allow-list» (project-overview §4) — read-side не
покрыт

project-overview §4: «PR-write is restricted to FA itself + a
controlled allow-list of 1–2 user repos (config)». Это **write
side**. **Read side** — `read_file`, `list_files` — может
читать **что угодно**. Безопасно ли это?

Сейчас FA — single-user single-machine. Чтения с локального
диска — это «своё», agent ничего не утекает в сеть, кроме
LLM-вызовов. Но **LLM-вызов отправляет содержимое** в облако
(если Coder = OpenRouter / Anthropic). Это де-facto **read
allow-list нужен тоже**, не только write.

ADR-1 / ADR-2 этого не покрывают; project-overview §6
(privacy) утверждает «no special PII / data-residency»,
поэтому формально OK, но это **blanket signing-off**, не
дизайн-решение. Mitigation — sandbox ADR (см. §10 R-2)
покрывает обе стороны.

### 9.4 ADR-3 §«Reserved vector slot» vs SLIDERS-extraction layer

ADR-3 §Decision: «Vector layer scaffolded (interface defined)
but not implemented в v0.1 — see ADR-4». ADR-4 §Decision:
«Adding the v0.2 vector layer is additive — same DB file, new
table».

То есть «v0.2 расширение Variant A» по умолчанию = vector layer.
SLIDERS показывает, что для UC2/UC3 на больших корпусах
**vector layer не помогает** (см. SLIDERS-нота §1, §2). Помогает
**extraction layer**.

Это значит: **v0.2 первый апгрейд может быть НЕ vector**, а
extraction. Текущие ADR не препятствуют, но и не сигнализируют
этого. Implementer Phase M, читая ADR-3, может «зарезервировать
вектора» в коде, и потом удивляться, почему «вектор не помог».

Mitigation — ADR-3 amendment, добавляющий Variant D как
**первичный** v0.2 кандидат рядом с vectors. См. §10 R-4.

### 9.5 hot.md → notes/sessions/<date>.md cycle vs ampcode
conversation accumulator

ADR-3 пишет «hot.md auto-archived at session end» но не
специфицирует *когда* промежуточные writes. Ампкод
conversation — in-RAM без диск-сериализации. При Ctrl-C / OOM
между tool-call'ами данные теряются. Выбор «append к hot.md
после каждого tool-result» vs «clean shutdown only» —
implementation-decision в §10 R-1 (loop ADR).

### 9.6 ADR-2 fallback chains не учитывают tool-protocol

ADR-2 per-role `primary → fallback` не выбирает между native-
tool и prompt-only protocol — это меняет shape вызова.
Либо запретить mixing в одной роли (models.yaml validator),
либо унифицировать на prompt-only с native как оптимизацией.
См. §10 R-1.

### 9.7 «No auto-escalation» (ADR-2) vs SLIDERS reconciliation
iteration

ADR-2 «no auto-escalation» = no cross-tier (Coder не
превращается в Debug автоматически). SLIDERS-Reconciler имеет
intra-role retry-loop (1.28 avg iterations) — это внутри
одной роли, формально не противоречит. В v0.2-implementer
может неправильно прочитать. Зафиксировать
терминологию в ADR-2 amendment / glossary. См. §10 R-8.

---

## 10. Пронумерованные рекомендации

> **Эти рекомендации — input для будущих ADR/PR, не сами ADR.**
> Каждая помечена estimate стоимости (cheap / medium /
> expensive) и horizon'ом (v0.1 / v0.2). Решение — за лидом.
> Конкретные имена/слаги в коде — иллюстративные, не
> авторитетные.

### R-1. Новый ADR — «Agent inner-loop and tool contract for v0.1»

- **Why:** §2.2 / §3.1 / §4 — ампкод даёт shape inner-loop'а и
  tool-registry, но ни одна из ADR-1..5 не специфицирует **как
  Coder LLM ↔ tools общаются**. Без этого ADR Phase M PR
  «implement loop» либо изобретёт свой shape, либо застрянет
  на review.
- **What:** один accepted ADR со следующими разделами:
  1. **Tool-Protocol** — `{name, description, input_schema,
     fn}` per ампкоду §4. JSON-Schema через pydantic.
  2. **Inner-loop** — псевдокод (~50 строк): conversation
     accumulator, parallel tool-use в одном assistant-блоке,
     truncate-стратегия при context overflow.
  3. **Validation** — pydantic.model_validate на tool inputs.
  4. **Tool-call audit-log** — append к session-log JSON-line с
     `{timestamp, tool, args_hash, return_summary, duration}`.
  5. **Cancellation** — Ctrl-C → flush conversation в hot.md →
     exit-code 130. Не deferable, простой signal handler.
  6. **Tool-protocol negotiation** — `tool_protocol: native |
     prompt` в models.yaml; loop детектирует и адаптирует
     shape вызова. См. §3.3.
- **Cost:** medium (отдельный design pass, ~200-300 строк
  ADR + reference implementation).
- **Horizon:** **v0.1 hard requirement**. Без этого Phase M
  не стартует.

### R-2. Новый ADR — «Tool sandbox / path allow-list policy»

- **Why:** §4.1 / §9.3 — UC1 read+edit на локальный диск без
  policy = security-bug. Sandbox-policy сейчас вообще нет.
- **What:** ADR с:
  - **Read allow-list** (директории, которые Coder может
    `read_file`): репо в `~/.fa/repos.toml` + `notes/` + `~/.fa`.
  - **Write allow-list** (для `edit_file`): только `notes/` +
    репо в allow-list. Запрет на ~/.ssh, /etc, /usr, /, любой
    путь начинающийся с `.git/objects/`.
  - **Network allow-list** (если/когда tools начнут делать
    HTTP) — отдельный сегмент.
  - **Failure mode:** tool возвращает structured error,
    inner-loop пишет это как `tool_result.is_error = true`,
    модель видит и пытается другой путь.
- **Cost:** cheap (policy + 30 LoC проверки путей; критично, но
  не сложно).
- **Horizon:** **v0.1 hard requirement**.

### R-3. Pre-implementation fixture — edit_file shape vs ADR-2 modes

- **Why:** §3.2 (tension-1) — string-replace **не валидирован**
  на mid-tier OSS моделях. Ampcode честно предупреждает.
- **What:**
  - 5–10 фикстурных задач: «add a docstring», «rename
    function», «extract method», «fix typo in comment», «add
    import», и т.п.
  - Прогон каждой задачи через каждую модель из ADR-2 routing
    (Planner / Coder / Debug). Замер success-rate
    string-replace vs diff-apply.
  - Если **string-replace проседает на mid-tier**: либо
    переключить Coder на diff-apply primary + string-replace
    fallback, либо upgrade'ить Coder до top-tier для editing.
- **Cost:** medium (10 fixtures × N models × 2 modes; manual
  scoring, но short-cycle).
- **Horizon:** **v0.1 hard gate before R-1's reference
  implementation lands**.

### R-4. ADR-3 amendment — Variant D «Extraction layer» как v0.2 candidate

- **Why:** §5.3 (gap-4), §9.4 — SLIDERS-style extraction —
  правильный v0.2 апгрейд для UC2/UC3 при росте корпуса.
  Vector layer (ADR-3 «reserved slot») не закрывает aggregation
  bottleneck.
- **What:** add к ADR-3 **section** (или к
  `memory-architecture-design-2026-04-26.md` §3 «Ось C» новый
  пункт C5) с:
  - Краткое описание Variant D = «structured extraction over
    filesystem canon, реляционные `entities` / `facts` /
    `provenance` таблицы поверх ADR-4 SQLite, schema induction
    per-topic, SQL-loop QA».
  - Когда триггерится: corpus > ~10 MB текста, или
    aggregation-style вопросы становятся доминирующими.
  - Что **общее** с Variant A: filesystem canon, ADR-4 SQLite,
    chunker.
  - Что **новое**: schema-induction-step, extraction step,
    reconciliation, SQL tool в registry.
  - Не ADR. Просто сильный pointer для v0.2 ADR-author'а.
- **Cost:** cheap (~80-100 строк markdown).
- **Horizon:** **v0.1 nice-to-have, v0.2 important**. Если
  пишем сейчас — экономим время первого v0.2 ADR.

### R-5. ADR-4 / ADR-5 amendment — расширить chunks schema и Chunk dataclass

- **Why:** §6.2 (gap-5), §7.2 (gap-6), §9.1 — line_start /
  line_end уже эмитятся chunker'ом, но не хранятся. Plus
  forward-compat для v0.2 SLIDERS.
- **What:** в обе ADR (`Chunk` dataclass в ADR-5; SQL `chunks`
  table в ADR-4) добавить колонки:
  - `parent_title` (H1 / frontmatter-title / filename),
  - `breadcrumb` (parent section headings, tuple/JSON array),
  - `line_start` / `line_end` (уже в dataclass'е, добавить в SQL),
  - `byte_start` / `byte_end` (новые).
  Frontmatter (optional): `topic:` field для ingestion-tagging.
- **Cost:** cheap (5-10 строк ADR amendment, schema migration
  пока ничего не залито). Точные SQL/dataclass-блоки — в
  самом amendment'е, не здесь.
- **Horizon:** **v0.1**. Если до first chunker PR — миграции 0;
  после — ALTER TABLE + reindex.

### R-6. Frontmatter-extension — `topic:` (optional) для UC3

- **Why:** §5.5, §8.4 — SLIDERS amortizes per-topic. UC3
  ingest на v0.2 может использовать topic-clustered schema.
  И `privacy_tier` для §8.4.
- **What:** в `knowledge/README.md` §Conventions добавить
  optional поля:
  ```yaml
  topic: "finance"           # for v0.2 amortization
  privacy_tier: "public"     # public | local-only | sensitive
  ```
  Парсер уже принимает свободные ключи; никакого breaking
  change.
- **Cost:** cheap (1 PR, ~30 lines в README).
- **Horizon:** **v0.1 cheap forward-compat**. Не блокер.

### R-7. ADR-2 amendment — явное «inner-loop в v0.1 однопроходный»

- **Why:** §3.4 / §8.6 (tension-2) — несогласованность ADR-2
  vs agent-roles.md. Implementer должен знать, что Critic /
  Specifier — deferred.
- **What:** add к ADR-2 §Consequences пункт:
  - «**v0.1 inner-loop** реализует только **роли из routing-
    таблицы**: Planner / Coder / Debug / Eval. **Critic /
    Reflector / Task-Specifier из `agent-roles.md` §5.1 —
    deferred к v0.2** (когда добавим reflexion-цикл и multi-
    role pipeline).»
  - + ссылка на agent-roles.md как «target architecture
    после v0.1».
- **Cost:** cheap (1 PR amendment, 5 строк в ADR-2 + cross-link).
- **Horizon:** **v0.1**. Cheap clarity-fix.

### R-8. `docs/glossary.md` updates — disambiguate terminology

- **Why:** §3.4, §9.2, §9.7 — три ambiguity-points
  (Eval vs Critic, no-auto-escalation vs retry-loop, role
  Coder vs Executor).
- **What:** добавить в `docs/glossary.md`:
  - **Coder** (ADR-2) ≡ **Executor** (agent-roles.md) — same
    role, two names. Standardize on **Coder** in ADR-2 / impl.
  - **Eval** (ADR-2) — offline LLM-as-judge baseline,
    version-pinned. **NOT** per-turn critic.
  - **Critic / Reflector** (agent-roles.md) — per-turn judge.
    **Not** in v0.1 scope; v0.2 candidate.
  - **«No auto-escalation»** (ADR-2) — **no cross-tier**
    escalation (Coder doesn't become Debug). **Within-role
    retry-loops are allowed and expected.**
- **Cost:** cheap (1 PR, ~30 lines).
- **Horizon:** **v0.1**. Mostly cosmetic but high value for
  implementer clarity.

### R-9. v0.2 watch-item — SQLite WAL mode при daemon-watcher

- **Why:** §6.3 — при v0.2 inbox-watcher + extraction +
  read-side concurrency возможны write-amplification issues.
- **What:** **note в HANDOFF.md или в ADR-4 §Future Work**, не
  ADR. Триггер: «когда добавляется второй процесс, пишущий в
  ~/.fa/state/index.sqlite». Mitigation — `PRAGMA journal_mode
  = WAL`.
- **Cost:** cheap.
- **Horizon:** v0.2.

### R-10. Eval gold-set формат — phrase-level с провенансом

- **Why:** §8.5 — SLIDERS-paper §6 Obs.2 предупреждает: gold
  часто неполный. Phrase-level provenance даёт debug-
  возможность.
- **What:** в ADR-1 §Follow-up или в новой research-ноте «Eval
  gold-set design» — формат gold-fixture'а с полями
  `question` / `answer` / `provenance: [{file, span, line}]` (полный
  YAML-пример живёт в самом eval-design документе).
- **Cost:** cheap. **Horizon:** v0.1 nice-to-have.

### R-summary — таблица приоритетов

| # | Recommendation | Cost | Horizon | Blocking? |
|---|---|---|---|---|
| R-1 | ADR «inner-loop + tool contract» | medium | v0.1 | **yes** |
| R-2 | ADR «sandbox / path allow-list» | cheap | v0.1 | **yes** |
| R-3 | Fixture run «edit_file shapes» | medium | v0.1 | **gate before R-1 impl** |
| R-4 | ADR-3 amendment «Variant D» | cheap | v0.1 nice / v0.2 | no |
| R-5 | ADR-4 + ADR-5 amendments (chunk schema) | cheap | **v0.1** | yes if to avoid migration later |
| R-6 | Frontmatter `topic:` + `privacy_tier:` | cheap | v0.1 forward-compat | no |
| R-7 | ADR-2 amendment «v0.1 single-loop» | cheap | v0.1 | recommended |
| R-8 | docs/glossary.md disambiguation | cheap | v0.1 | recommended |
| R-9 | HANDOFF/ADR-4 note WAL | cheap | v0.2 | no |
| R-10 | Eval gold phrase-level | cheap | v0.1 follow-up | no |

**Минимальный «pre-Phase-M» pack:** R-1, R-2, R-3, R-5
(must), R-7, R-8 (recommended). R-4 и R-6 — cheap forward-
compat: можно сразу, можно отложить. R-9 / R-10 — на момент
ev/upgrade.

---

## 11. Открытые вопросы под решение лида

Multiple-choice; ответы войдут в будущие ADR / amendment'ы.

- **Q-1. Inner-loop ADR (R-1) — форма:**  отдельный ADR-6 «inner-
  loop + tool contract».
  > **Status:** superseded by reality 2026-05-01. ADR-6 был
  > занят отдельным sandbox-policy ADR (PR #6, merged
  > 2026-04-29 — см. Q-2 ниже). Inner-loop ADR теперь
  > планируется как **ADR-7** (свободный слот). Решение
  > подтверждено лидом 2026-05-01 (option «accept-as-is» по
  > §11 reconciliation в
  > [`semi-autonomous-agents-cross-reference-2026-05.md`](./semi-autonomous-agents-cross-reference-2026-05.md)
  > §7.2). Ссылочные пункты в других нотах, упоминающие
  > «ADR-6 inner-loop», следует читать как «ADR-7 inner-loop
  > (future)».
- **Q-2. Sandbox policy (R-2) — форма:** amendment к ADR-1 §«Concrete
  v0.1 in-scope list».
  > **Status:** superseded by reality 2026-05-01. Sandbox
  > policy была оформлена как **отдельный ADR-6** (PR #6,
  > merged 2026-04-29) с 350+ строк policy / TOML / globs /
  > audit-log / one-shot bypass — слишком объёмно для
  > §«Concrete v0.1 in-scope list» ADR-1. ADR-1 без
  > amendment по этому пункту. Решение подтверждено лидом
  > 2026-05-01 (option «accept-as-is» по §11 reconciliation
  > в
  > [`semi-autonomous-agents-cross-reference-2026-05.md`](./semi-autonomous-agents-cross-reference-2026-05.md)
  > §7.2 — отдельный ADR подтверждается independent reference
  > goclaw 5-layer permission system).
- **Q-3. Edit-format фикстура (R-3) — формат будущего решения:**
  string-replace OK на всех тиерах → запинить в loop ADR.
- **Q-4. Variant D (R-4) — где живёт:** (a) amendment к ADR-3,
  новый раздел «Future variants/upgrade» .
- **Q-5. `chunks` schema (R-5) — когда:** сейчас, до first
  chunker PR.
- **Q-6. Frontmatter `topic:` / `privacy_tier:` (R-6):** только `topic:`,
  `privacy_tier` premature.
- **Q-7. ADR-2 amendment про single-loop (R-7):**  да.
- **Q-8. Терминология (R-8) — форма:** (a) `docs/glossary.md` + inline в
  каждой ADR кратко.
- **Q-9. Implementation vs ADR-first:** параллельно: R-1+R-2 первым PR, R-5 в
  chunker PR, R-3 фикстуры отдельно
- **Q-10. Кто пишет amendment-PR'ы:** эта сессия после
  ответов.

---

## 12. Что эта нота намеренно НЕ покрывает

Конкретные model-slugs («Note on model slugs» в ADR-2),
собственные perf-benchmark'и, multi-agent / Workforce / handoff
(`agent-roles.md` §3.7-3.10), Variant B (Mem0) аргументы, UC4
+ PDF/DOCX/YouTube extractors, prompt-templates — всё v0.2+
или deferred.

---

## 13. Связь с другими нотами

- [`how-to-build-an-agent-ampcode-2026-04.md`](./how-to-build-an-agent-ampcode-2026-04.md)
  — write-side parent; §3 / §5 — основа R-1.
- [`sliders-structured-reasoning-2026-04.md`](./sliders-structured-reasoning-2026-04.md)
  — read-side parent; §3.1 — основа R-5; §6 — основа R-4.
- [`memory-architecture-design-2026-04-26.md`](./memory-architecture-design-2026-04-26.md)
  — Variant A/B/C, в которые SLIDERS добавляет D (R-4).
- [`chunker-design.md`](./chunker-design.md) — base для ADR-5;
  R-5 расширяет Chunk dataclass.
- [`agent-roles.md`](./agent-roles.md) — Specifier / Planner /
  Executor / Critic; tension с ADR-2 (R-7, R-8).
- [`agentic-memory-supplement.md`](./agentic-memory-supplement.md)
  — Mem0 (Variant B); R-4 mapping.
- [`llm-wiki-critique-first-agent.md`](./llm-wiki-critique-first-agent.md)
  — provenance-frontmatter (T1) — поддерживает R-6.

---

## 14. Ограничения этой ноты

Frontmatter `chain_of_custody` фиксирует single-pass synthesis,
source-pointers на parent-ноты и ADR. Дополнительно: цифры
из parent-нот («400 строк» / «$0.76 / question») не реплицировались;
upstream-репо ампкода и stanford-oval/sliders не клонировались;
second-LLM peer-review нет; edit-shape фикстуры (R-3) не
запускались. Оценка «cheap migration» в R-5 верна только пока
src/fa пуст; после Phase M start — устаревает.

---

## Sources

- ampcode parent-нота:
  [`how-to-build-an-agent-ampcode-2026-04.md`](./how-to-build-an-agent-ampcode-2026-04.md)
  (compiled 2026-04-29).
- SLIDERS parent-нота:
  [`sliders-structured-reasoning-2026-04.md`](./sliders-structured-reasoning-2026-04.md)
  (compiled 2026-04-29).
- ADR-1 [`v01-use-case-scope`](../adr/ADR-1-v01-use-case-scope.md)
  (accepted 2026-04-27).
- ADR-2 [`llm-tiering`](../adr/ADR-2-llm-tiering.md) (accepted
  2026-04-27).
- ADR-3 [`memory-architecture-variant`](../adr/ADR-3-memory-architecture-variant.md)
  (accepted 2026-04-27).
- ADR-4 [`storage-backend`](../adr/ADR-4-storage-backend.md)
  (accepted 2026-04-27).
- ADR-5 [`chunker-tool`](../adr/ADR-5-chunker-tool.md)
  (accepted 2026-04-28).
- [`project-overview.md`](../project-overview.md).
- [`docs/architecture.md`](../../docs/architecture.md).
- [`docs/workflow.md`](../../docs/workflow.md).
- [`agent-roles.md`](./agent-roles.md).
- [`memory-architecture-design-2026-04-26.md`](./memory-architecture-design-2026-04-26.md).
- [`chunker-design.md`](./chunker-design.md).
- [`agentic-memory-supplement.md`](./agentic-memory-supplement.md).
- [`llm-wiki-critique-first-agent.md`](./llm-wiki-critique-first-agent.md).
