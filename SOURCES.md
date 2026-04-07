# Sources — Сырые данные из видео и скринов

Исходные материалы из видео по контекстной инженерии. Скрины промптов, транскрипция, реальные артефакты. Используется как reference при доработке визарда и агентской системы.

## Агентская система

### Планируемая структура результата (образец из видео)

Целевая структура `.claude/` после полной реализации визарда:

```
.claude/
├── agents/                          ← субагенты агентской системы (переиспользуемые роли)
│   ├── codebase-researcher.md       ← субагент для research: сканирует код по конкретному направлению
│   ├── architect-reviewer.md        ← ревьюер соответствия архитектуре/дизайну
│   ├── blast-radius-analyzer.md     ← анализатор влияния изменений (какие части системы затронуты)
│   ├── bug-tracer.md                ← трассировщик багов (воспроизведение, root cause)
│   ├── quality-reviewer.md          ← ревьюер качества кода
│   └── security-reviewer.md         ← ревьюер безопасности
│
├── commands/                        ← команды трубы (вызываются разработчиком последовательно)
│   ├── research_codebase.md         ← /research — исследование кодовой базы по задаче
│   ├── design_feature.md            ← /design-feature — архитектура новой фичи
│   ├── design_bugfix.md             ← /design-bugfix — архитектура фикса бага
│   ├── design_frontend_feature.md   ← /design-frontend-feature — архитектура фронтенд-фичи
│   ├── implement_backend.md         ← /implement-backend — имплементация бэкенда
│   ├── implement_frontend.md        ← /implement-frontend — имплементация фронтенда
│   ├── fix_backend_bug.md           ← /fix-backend-bug — фикс бэкенд-бага
│   └── fix_frontend_bug.md          ← /fix-frontend-bug — фикс фронтенд-бага
│
├── skills/                          ← дополнительные скиллы (утилиты)
│
├── claude.md                        ← общие инструкции проекта
└── settings.local.json              ← локальные настройки
```

#### Ключевые паттерны из образца:

**Разделение agents/ и commands/:**
- `agents/` — переиспользуемые роли-субагенты. Не вызываются напрямую разработчиком. Вызываются из commands/ через `subagent_type: "agent-name"`.
- `commands/` — команды трубы. Вызываются разработчиком (`/research`, `/design-feature`). Оркестрируют работу, запускают субагентов из agents/.

**Специализация commands/ по двум осям:**
- Тип задачи: `feature` vs `bugfix`
- Стек: `backend` vs `frontend`
- Это даёт матрицу: `design_feature`, `design_bugfix`, `design_frontend_feature`, `implement_backend`, `implement_frontend`, `fix_backend_bug`, `fix_frontend_bug`

**Паттерны из промпта research_codebase.md (образец):**
- Роль задана жёстко: "YOUR ONLY JOB: DOCUMENT AND EXPLAIN THE CODEBASE AS IT EXISTS TODAY"
- Critical constraints явно запрещают выход за роль: DO NOT suggest improvements, DO NOT critique, DO NOT propose changes, ONLY describe what EXISTS
- Процесс формализован по шагам:
  1. Initial Response — агент отвечает "I'm ready to research" и просит задачу
  2. Decompose the Research Question — декомпозиция на 2-4 направления, создание task list через TaskCreate
  3. Spawn Parallel Research Tasks — запуск субагентов `codebase-researcher` параллельно
- Правила параллелизма субагентов:
  - 2-4 параллельных задач (не больше 4 — context overflow risk)
  - Sequential — когда одна область зависит от находок другой
  - Background — для широких поисков, которые не блокируют основную работу
- Каждый таск для субагента должен включать:
  - Конкретный вопрос, на который нужно ответить
  - Стартовые файлы/пути (если известны)
  - Формат выхода (что именно вернуть)
  - Explicit scope boundaries — что НЕ исследовать
- Пример задач для субагентов:
  - "Trace authentication flow from HTTP handler to DB" → codebase-researcher
  - "Map all event handlers and their triggers" → codebase-researcher
  - "Document repository interfaces and their implementations" → codebase-researcher
- Дополнительные шаги после получения результатов от субагентов:
  4. Merge & Analyze Results — собрать результаты, построить общую картину с cross-references
  5. Identify gaps — найти пробелы, при необходимости запустить follow-up субагентов (max 1 follow-up round)
  6. Gather Metadata — собрать метаданные в YAML frontmatter
  7. Generate Research Document — сгенерировать итоговый документ
- Метаданные research-документа (YAML frontmatter):
  ```yaml
  date: YYYY-MM-DD
  researcher: Claude
  commit: $(git rev-parse --short HEAD)
  branch: $(git branch --show-current)
  research_question: "Original question"
  ```
- Формат итогового research-документа:
  ```markdown
  [YAML frontmatter with metadata]
  ---
  # Research: [Topic]

  ## Summary
  [2-3 paragraph executive summary]

  ## Detailed Findings

  ### 1. [Component/Area Name]
  - **Location**: `path/to/file.go:line-numbers`
  - **Description**: What it does
  - **Dependencies**: What it uses/imports
  - **Data Flow**: Input → Processing → Output

  ### 2. [Next Component]
  ...

  ## Code References
  - `file.go:42` — description
  ```
- Ключевые паттерны формата:
  - Git commit и branch фиксируются — research привязан к конкретному состоянию кода
  - Executive summary 2-3 параграфа (не bullet points) — для быстрого понимания
  - Findings группируются по компонентам/областям, не по файлам
  - Каждый finding имеет Location + Description + Dependencies + Data Flow
  - Code References отдельной секцией — быстрый lookup
  - Дополнительные секции в research-документе:
    ```markdown
    ## Architecture Insights
    - Pattern used: [name]
    - Data flow: A → B → C
    - Key dependencies: ...

    ## Open Questions
    [Anything that needs further investigation]
    ```
  - Architecture Insights — выявленные паттерны, data flow, ключевые зависимости (факты, не мнения)
  - Open Questions — что осталось неясным и требует дополнительного исследования
- Critical Rules (шаг 7 в research_codebase.md):
  1. Always include file:line references — no vague descriptions
  2. Read files COMPLETELY — no limit/offset
  3. Use codebase-researcher subagent for parallel investigation
  4. Max 4 parallel tasks — more causes context overflow
  5. Maintain objectivity — only facts, no opinions
  6. Preserve exact paths — use paths as they exist in the repo
- Output path: `.thoughts/research/YYYY-MM-DD-topic-name.md`
- Good vs Bad Research (примеры из образца):
  - BAD: "The authentication system is poorly designed."
  - GOOD: "The authentication system uses JWT tokens (`src/auth/jwt.go:42`). Tokens are verified in middleware (`src/auth/middleware.go:89`) before reaching protected routes."
  - BAD: "The code should use async/await instead of callbacks."
  - GOOD: "The database queries use callback-based API (`src/db/users.go:156-178`). Error handling follows the `(err, result)` pattern..."
- Ключевой паттерн: примеры good/bad в промпте — мощный способ задать уровень конкретности и объективности

### Пример реального research-документа (артефакт первой фазы)

Файл: `.thoughts/research/2026-02-19-usr-avatar.md`
Задача: USR-AVATAR — User Avatar Upload

**Структура `.claude/` из образца (полная, с учётом новых папок):**
```
.claude/
├── agent-memory/          ← память агентов (между сессиями?)
├── agents/
├── commands/
├── skills/
├── claude.md
└── settings.local.json
```
Также: `.thoughts/research/` — папка для артефактов research-этапа.

**Качество реального research-документа (образец):**

Summary:
- 2 параграфа, конкретные факты: "user-service currently has **no avatar functionality implemented**. However, there are 4 orphaned error constants..."
- Сразу обозначает архитектуру: "strict Clean Architecture with DDD. Every component uses the Builder pattern with three-phase validation."
- Конкретные цифры: "The User entity has 14 private fields"

Detailed Findings — группировка по компонентам:
1. User Entity:
   - Location: `domain/entity/user/User.go:12-27`
   - Перечислены все 14 полей
   - **No avatar field exists** — явно указано что нет
   - Перечислены mutation methods с точными строками
   - Паттерн: `Every mutating method stamps u.updatedAt = time.Now()`

2. User Entity Builder:
   - Location: `domain/entity/user/Builder.go:13-206`
   - Three-phase: `Build()` → `checkRequiredFields()` → `fillDefaultFields()` → `validateBusinessRules()`
   - Required fields перечислены
   - Defaults перечислены с конкретными значениями
   - **No avatar setter exists** — явно

3. Error Constants (Orphaned — Avatar/Storage):
   - Точные пути и коды ошибок: `domain/entity/user/Error.go:30` — `ErrAvatarKeyIsRequired` (USR-400)
   - "None of these are referenced in any code path" — факт об orphaned коде

**Ключевые паттерны из реального артефакта:**
- Каждый finding начинается с точного Location
- Явно указывает что НЕ существует (no avatar field, no avatar setter) — это критично для фичи
- Orphaned код обнаружен и помечен — предыдущая попытка реализации?
- Уровень детализации: конкретные имена полей, методов, значения defaults, коды ошибок
- Builder pattern трассирован по фазам: Build → check → fill → validate

### Переход между этапами трубы (образец)

**Как вызывается design-этап:**
```
/clear
/design_feature user-avatar .thoughts/research/2026-02-19-usr-avatar.md USR-AVATAR: Аватарка пользователя
```

Ключевые наблюдения:
- `/clear` перед каждым этапом — **контекст сбрасывается**. Каждый этап работает в чистом контексте, без мусора от предыдущего
- Аргументы команды: `<task-slug> <path-to-research> <task-id: описание>`
- Research-документ передаётся явно по пути — агент сам его читает
- Задача описана кратко в аргументах, но детали берутся из research.md

**Что передаёт человек вместе с командой (помимо research):**
Детальное описание задачи от человека — уже с конкретикой:
- Что нужно: POST/DELETE эндпоинты, avatar_urls в GET user
- Ограничения: jpeg/png/webp, до 5MB, квадратная, минимум 256x256
- Ресайз варианты: sm 64x64, md 128x128, lg 256x256, xl 512x512
- Хранение: S3, JPEG качество 85%, оригинал хранится
- Формат ответа API: JSON с avatar_urls (sm, md, lg, xl)
- Права: загружать свою или админ чужую
- Поведение: при перезагрузке — старый файл удалять

Это важно: **задача уточняется человеком на входе в design**, а не только в research. Research дал факты о коде, а человек добавляет бизнес-требования.

**Реакция агента:**
"I'll start by reading the research file and discovering the codebase structure in parallel."
— Агент сразу читает research и параллельно изучает код. Не спрашивает уточнений.

### Образец промпта design_feature.md (второй этап трубы)

**Стандарты проекта загружаются из файлов:**
- Читает ALL files in `/promts/` — Architecture Layers, Clean Architecture, Domain Model, Builder, RepoModel, Go Style, Tests Style, Dom...
- Папка `/promts/` содержит стандарты проекта — архитектурные принципы, стиль кода, паттерны. Это проектная специфика, не универсальные правила.
- Конкретные файлы стандартов из образца (Go-проект):
  - `Architecture Layers.txt`
  - `Builder.txt`
  - `Clean architecture.txt`
  - `Domain Model.txt`
  - `Domain model test.txt`
  - `Go style.txt`
  - `RepoModel.txt`
  - `Tests Style.txt`
- Это отдельные файлы с правилами для каждого аспекта: как строить слои, как использовать builder, как писать domain model, как писать тесты, стиль Go-кода. Визард должен генерировать аналогичные файлы стандартов для конкретного проекта (или как минимум учитывать их в промптах агентской системы).

**Определение необходимости дополнительного research:**
Когда research нужен:
- Current architecture of the affected modules
- Unfamiliar parts of the codebase
- Implementations that might be reusable
- unclear

**Conditional documents — таблица зависимости:**
Design-агент решает, какие **conditional documents** нужны (see Phase 2.3):

| Conditional Document     |                     |
|--------------------------|---------------------|
| events                   | `05-events.md`      |
| entities                 | `06-repo-model.md`  |
| ...standards             | `07-standards.md`   | (частично видно)
| endpoints                | `08-api-contract.md`|

Т.е. design генерирует не один файл, а набор документов в зависимости от задачи. Нумерация `05-`, `06-`, `07-`, `08-` — часть общей серии документов.

Полная таблица conditional documents (Phase 0.5 Determine Feature Context):

| Condition                        | Conditional Document   |
|----------------------------------|------------------------|
| Feature has domain events        | `05-events.md`         |
| Backend feature with entities    | `06-repo-model.md`     |
| Backend feature                  | `07-standards.md`      |
| Feature exposes REST endpoints   | `08-api-contract.md`   |

**Дополнительный research на этапе design (optional):**
Спавнит 2-3 параллельных субагента `codebase-researcher` для:

Architecture Analysis — анализ {service-path}:
- `service/container.go` — DI container, all registered dependencies
- `init-service/` — scan all init files, map initialization chain order
- Search for `RegisterFiberRoute`, `fiber.Router` — map existing endpoints
- `/entity/` — list all entities, their VOs, builders, state machines
- `/usecase/` — list all use cases, their dependencies
- `adapter/` — controllers, repos, gateways

Т.е. design-агент тоже может запускать research-субагентов если ему не хватает контекста из research.md.

**Research Task 1: Architecture Analysis** (полный текст задачи для субагента):
```
Analyze the architecture of {service-path}:

Start from these entry points:
1. {service-path}/initService/container.go — DI container, all registered dependencies
2. {service-path}/initService/ — scan all init files, map initialization chain order
3. Find router setup (search for RegisterFiberRoute, fiber.Router) — map existing endpoints
4. {service-path}/domain/entity/ — list all entities, their VOs, builders, state machines
5. {service-path}/domain/usecase/ — list all use cases, their dependencies
6. {service-path}/adapter/ — controllers, repos, gateways

For each relevant component, report:
- Layer and package structure
- Builder pattern usage (3-stage order)
- Value object patterns
- Repository mapping patterns (NewFromEntity/ToEntity/Restore)
- Event patterns (if any)

Report file:line references for all findings.
Save findings — facts only, no critique, no suggestions.
```

**Research Task 2: Pattern Discovery** (задача для субагента):
```
Find patterns relevant to "{feature-name}" in {service-path}:

1. Similar features already implemented — find the closest analog and document its full structure
2. Reusable components in domain-common/ — shared primitives, VOs, utilities
3. API patterns — endpoint registration, request/response formats, error response structure
4. Testing patterns — test suite setup, mock patterns, stub patterns
5. Error code ranges — scan ALL error codes to map used ranges and find next available
```

Ключевое: Task 2 ищет **аналоги** — ближайшую похожую фичу как образец для проектирования. Также сканирует error code ranges чтобы не было коллизий.

**Research Task 3: Integration Analysis** (задача для субагента):
```
Analyze integration points for "{feature-name}":

1. External services and gateways — gRPC clients, HTTP clients, LLM pipelines
2. Event-driven communication — event bus, domain events, handlers
3. Shared types and contracts (boundary layer) — interfaces exposed to other modules
4. Database collections affected — existing MongoDB collections, indexes
5. Auth/middleware — how auth is applied to routes

Report file:line references for all findings.
Save findings — facts only.
```

**Save Research — сохранение результатов:**
- После завершения субагентов — `mkdir -p {service-path}/docs/{feature-name}`
- Research сохраняется в: `{service-path}/docs/{feature-name}/research.md`
- Т.е. документы design-фазы хранятся **внутри service-path**, а не в `.thoughts/`
- Это третий вариант хранения: research_codebase → `.thoughts/research/`, design_feature research → `{service-path}/docs/{feature-name}/`

**Формат research-документа из design_feature (Phase 1):**
```markdown
---
date: YYYY-MM-DD
feature: {feature-name}
service: {service-path}
---

# Research: {Feature Name}

## Summary
[2-3 paragraphs: what exists, what's relevant, key patterns found]

## Project Structure (Discovered)
- **Init chain order:** [Config → Repos → Gateways → ... actual order from initService/]
- **Router pattern:** [How routes registered, middleware — reference file:line]
- **Entity pattern:** [VO usage, builder pattern — reference closest analog file:line]
- **Repo pattern:** [NewFromEntity/ToEntity/Restore — reference file:line]
- **Error code ranges in use:** [PRS-001..099 validation, 100..199 state, etc.]
- **Next available error range:** [e.g. PRS-600..699]

## Architecture Overview
[Current architecture of affected modules]

## Existing Patterns
[Patterns found that should be reused — with file:line references]

## Integration Points
[External dependencies, gateways, events]

## Key Files
- `path/to/file.go:line` — description
```

Ключевое: Error code ranges — агент маппит ВСЕ занятые диапазоны и находит следующий свободный. Это уровень детализации, который предотвращает коллизии.

**Phase 2: Design (Multi-File, View-Based)** — генерирует несколько файлов-views (детали Phase 2 между скринами, но структура видна из Rules).

**Артефакты design_feature (конец промпта):**

Artifacts:
- Design: `{service-path}/docs/{feature-name}/` ✅ Approved
- Code Plan: `{service-path}/docs/{feature-name}/plan/` ({N} phase files)

Next step после approval:
- Backend: `/implement_backend {service-path}/docs/{feature-name}/plan/README.md`
- Frontend: `/implement_frontend {service-path}/docs/{feature-name}/plan/README.md`

**Approval flow (human-in-the-loop):**
```
Please review the code plan and:
1. ✅ Approve — ready for implementation
2. 🔄 Request changes — specify adjustments
3. ❗ Questions — ask about specific phases
```
**WAIT for user approval.** — агент останавливается и ждёт.

**Rules (15 правил design_feature — полный список):**
1. **Design before code** — never jump to implementation details in Phase 2
2. **Multi-file by view** — separate structure (01), behavior (02), decisions (03), testing (04). NEVER put everything in one file
3. **Mermaid for all diagrams** — renderable, versionable, diffable
4. **file:line references** — every reference to existing code includes exact location
5. **Facts in research, decisions in design** — research is objective, design is opinionated
6. **Two approval gates** — design approval AND code plan approval before implementation
7. **Read ALL standards AND discover real structure first** — read every file in promts/ AND explore the actual codebase (container.go, init chain, router, entities) before designing
8. **Stop at uncertainty** — ask the user, don't guess architectural decisions
9. **C4 zoom-in narrative** — L1→L2→L3 in one file (01-architecture.md), they tell one continuous story
10. **Conditional files** — only create 05-08 when the feature requires them (see Phase 0.5)
11. **One sequence per use case** — group happy path + error cases + edge cases under one use case section in 02-behavior.md
12. **Exact API contract** — every REST endpoint must have exact JSON request/response shapes with field names and types in 08-api-contract.md
13. **Cross-document consistency** — architect reviewer verifies that all documents reference each other correctly (entities↔test...)
14. **Error code conflict check** — verify new error codes don't conflict with existing ranges before assigning
15. **Match real project patterns** — discovered in Phase 0.3. NEVER use generic/textbook patterns that don't match the codebase

**Ключевые паттерны из Rules:**
- Design файлы нумерованы по views: 01-architecture.md, 02-behavior.md, 03-decisions.md, 04-testing.md + conditional 05-08
- Два gate approval: design И code plan — оба до имплементации
- Rule 5 — принципиальное различие: research объективен (факты), design субъективен (решения)
- Rule 15 — НИКОГДА не использовать generic/textbook паттерны, только реальные паттерны проекта
- Rule 9 — C4 как zoom-in narrative: L1→L2→L3 в одном файле, непрерывная история
- Plan хранится в `{service-path}/docs/{feature-name}/plan/` с README.md и phase files
- Implement вызывается с путём к plan/README.md

**Реальная структура docs/{feature-name}/ (из file tree):**
```
docs/
└── user-avatar/
    ├── plan/
    │   ├── README.md
    │   ├── 01-architecture.md    ← phase file (не design view!)
    │   ├── 02-behavior.md
    │   ├── 03-decisions.md
    │   ├── 04-testing.md
    │   ├── 06-repo-model.md      ← conditional
    │   ├── 07-standards.md       ← conditional
    │   └── 08-api-contract.md    ← conditional
    └── [design views: 01-architecture.md, 02-behavior.md, 03-decisions.md, 04-testing.md, etc.]
```
Примечание: в file tree видны design views как отдельные табы (01-architecture.md, 02-behavior.md, 03-decisions.md, 04-testing.md, 06-repo-model.md, 07-standards.md, 08-api-contract.md). Plan файлы хранятся в подпапке `plan/`.

**Design approval flow (подробнее):**
- **WAIT for user approval.** Do NOT proceed to code plan without explicit approval.
- Если user запрашивает изменения:
  1. Update the **specific file** with feedback (not all files)
  2. Re-run architect review if changes are significant
  3. Present again for approval

**Phase 5: Code Plan (4th C — Code Level)**

После design approval — создание implementation plan как **separate files per phase** в `plan/` директории.

### Phase Order Strategy:
Выбор порядка фаз на основе типа фичи:

**Option A: Bottom-up (default for most features)**
Domain → Use Case → Adapter → Controller → DI

**Option B: Adapter-first (for features extending existing entities with new persistence)**
Adapter (repo) → Domain → Use Case → Controller → DI
Advantage: repo model validates data model assumptions early

**Option C: Vertical slice (for features with independent endpoints)**
All layers for Endpoint 1 → All layers for Endpoint 2 → DI
Advantage: shippable increment per phase

Выбранная стратегия и rationale документируются в plan/README.md.

### Output Directory:
```
{service-path}/docs/{feature-name}/plan/
├── README.md      — Overview, file map, DI, error codes, success criteria
├── phase-01.md    — First implementation phase
├── phase-02.md    — Second implementation phase
└── phase-NN.md    — One file per phase
```

### Why separate files?
- Each phase is **self-contained** — implementer reads ONE file per assignment
- Phases can be reviewed/approved **independently**
- Lead can hand a single file to the implementer agent without noise
- Progress tracking: checkmark in README.md, phase file stays as reference

**Формат plan/README.md:**
```markdown
---
date: YYYY-MM-DD
feature: {feature-name}
service: {service-path}
design: ../README.md
status: draft | approved
---

# Code Plan: {Feature Name}

## Overview
[Summary: what will be implemented, referencing design docs for architecture]

## Phase Strategy
[Bottom-up / Adapter-first / Vertical slice — and WHY]

## Phases

| # | Phase   | Layer      | Dependencies | Status |
|---|---------|---------|--------------|--------|
| 1 | [Name]  | Domain    | —            | ⬜     |
| 2 | [Name]  | Use Case  | Phase 1      | ⬜     |
| N | [Name]  | [Layer]   | Phase X      | ⬜     |

## File Map
[видимо далее — маппинг файлов]
```

**Ключевые паттерны:**
- YAML frontmatter: ссылка на design (`design: ../README.md`), статус (`draft | approved`)
- Phase Strategy с обоснованием WHY — не просто порядок, а аргументация
- Таблица фаз: номер, имя, слой архитектуры, зависимости между фазами, статус (⬜ чекбокс)
- Зависимости между фазами явные (Phase 1 → Phase 2) — нельзя начать фазу пока зависимость не завершена
- Status tracking прямо в README.md — ⬜ → ✅

### Образец промпта implement_backend.md (четвёртый этап трубы)

**Заголовок:** Implement Backend — Go Agent Team

**Роль:** "You are the **Lead** of a mob programming team implementing a Go backend plan. You orchestrate, you never write implementation code."

**Core principle:** No phase is complete until the quality gate passes. No exceptions.

**Phase 0: Understand the Mission**

0.1 Read the Plan:
- Read the ENTIRE plan at `$ARGUMENTS[0]` (путь к plan/README.md)
- Что читать:
  - All phases, their order, dependencies
  - Existing checkmarks (✅) — skip completed phases
  - Verification steps per phase
  - Acceptance criteria
  - Related design document (read it too for architectural context)

0.2 Read the Design Document:
- If the plan references a design.md:
  - Understand the C4 architecture decisions
  - Note the data flow and sequence diagrams
  - These are your architectural constraints

0.3 Analyze Phases:
- For each phase determine:
  - Which domain entities are affected?
  - Which layers are touched (domain, usecase, repo, controller)?
  - What are the dependencies between phases?
  - Are there integration points with existing modules?

**Phase 1: Create the Agent Team**

1.1 TeamCreate:
- Use the **TeamCreate** tool to create the team:
  - `team_name`: derive from feature name, e.g. `{feature-slug}-impl`
  - `description`: "Implementing {feature name} — Go backend"

**Phase 1 (подробно): Create the Agent Team**

1.1 TeamCreate:
- `team_name`: `{feature-slug}-impl`
- `description`: "Implementing {feature name} — Go backend"
- Создаёт:
  - Team config at `~/.claude/teams/{team-name}/config.json`
  - Shared task list at `~/.claude/tasks/{team-name}/`

1.2 Create Tasks:
- TaskCreate для каждой фазы из плана:
  - `subject`: "Phase N: {Phase Name}"
  - `description`: paste the FULL phase details from the plan — files to create/modify, key decisions, verification criteria. Task must be **self-contained** — a teammate reads ONLY this task description.
  - `activeForm`: "Implementing Phase N: {Name}"
- TaskUpdate для зависимостей:
  - `addBlockedBy`: link each task to the phases it depends on (e.g. Phase 2 blocked by Phase 1)
- ONE additional task в конце:
  - `subject`: "Final Cross-Phase Review"
  - `description`: "Review all phases together for cross-cutting concerns: unique error codes, no orphaned code, naming consistency, DI chain correctness, Value Objects everywhere."
  - `activeForm`: "Running final cross-phase review"
  - `addBlockedBy`: all implementation phase task IDs

1.3 Enable Delegate Mode:
- Enter **Delegate Mode** (Shift+Tab). В delegate mode Lead ограничен coordination-only tools — spawning, messaging, task management. Lead CANNOT write code. Это гарантирует разделение ответственности.

1.4 Spawn Teammates:
- Use the **Task** tool with `team_name` parameter to spawn each teammate. Each teammate is a persistent Claude Code session that stays alive, claims tasks, and communicates.

**Ключевые паттерны:**
- Lead НИКОГДА не пишет код сам — только оркестрирует. Delegate Mode это enforce'ит технически
- Пропускает уже завершённые фазы (✅ checkmarks)
- Design document читается для архитектурного контекста — это constraints
- **TeamCreate** — использует Claude Teams для создания команды агентов
- Teammates — persistent Claude Code sessions через Task tool

**Spawn: Backend Implementer**

Task tool parameters:
```
name: "backend"
team_name: "{team-name}"
subagent_type: "general-purpose"
mode: "bypassPermissions"
prompt: [see below]
```

Prompt for backend implementer:
```
You are the **backend** implementer in a mob programming team for Go microservices.

## Your Role
- You IMPLEMENT Go code for the task assigned by the Lead
- You run build and tests before reporting done
- You do NOT move to the next task without Lead's approval

## Team Coordination
- Check **TaskList** after completing each task to find your next assignment
- Use **TaskUpdate** to mark tasks `in_progress` when starting and `completed` when done
- Use **SendMessage** (type: "message", recipient: "lead-name") to report status to the Lead
- If you need the reviewer, message the Lead — do NOT message the reviewer directly

## Ownership
- You work within the service directory specified by the Lead
- Typical structure: `microservices/{service}/domain/`, `adapter/`, `initService/`

## MANDATORY: Read Standards Before Coding
Before writing ANY code, read ALL files in `/promts/be/`:
- `Architecture Layers.txt` — layer separation, dependency direction
- `Clean architecture.txt` — UseCase → Entity → Repository boundaries
- `Domain Model.txt` — private fields, getters, invariants, Value Objects for ALL fields
- `Builder.txt` — 3-stage: checkRequired → fillDefault → validateBusiness
- `RepoModel.txt` — model → entity mapping, round-trip preservation
- `Tests Style.txt` — Test Suite, t.Parallel(), AAA, single assertion per test
- `Domain model test.txt` — entity test patterns, stub organization
- `Go style.txt` — naming, typed errors, no comments, no magic strings
- `Base Mock.txt` — BaseMock library for test mocks
- `LLM Integration.txt` — if touching LLM/AI features
```

Ключевые паттерны teammate:
- `mode: "bypassPermissions"` — teammate не спрашивает разрешения на каждое действие
- Teammate коммуницирует ТОЛЬКО через Lead (не напрямую с reviewer)
- Standards в подпапке `/promts/be/` (backend) — есть разделение be/fe для стандартов
- Каждый стандарт с кратким описанием — implementer знает зачем читать
- MANDATORY: прочитать стандарты ДО написания любого кода

**1.5 Review Protocol**

Когда implementer сообщает что фаза готова, Lead отправляет **все 4 review агента одновременно** через SendMessage. Они persistent teammates — spawned once, reused for every review round.

4 параллельных сообщения:
```
SendMessage to "rv-build": "Review Phase N. Service: {service-path}"
SendMessage to "rv-arch": "Review Phase N. Changed files: [list]. Service: {service-path}"
SendMessage to "rv-sec": "Review Phase N. Changed files: [list]. Service: {service-path}"
SendMessage to "rv-plan": "Review Phase N. Changed files: [list]. Plan: [phase-file-path]. Design: [design-dir-path]. Service: {service-path}"
```

Aggregate verdict после ответа всех 4:
```
AGGREGATE VERDICT:
- rv-build: ✅/❌
- rv-arch: ✅/❌
- rv-sec: ✅/❌
- rv-plan: ✅/❌

ALL ✅ → Phase APPROVED
ANY ❌ → Phase REJECTED — combine ALL findings into one message to implementer
```

Формат rejection message (группировка по reviewer):
```
## ❌ Phase [N] REJECTED

### Build/Test/Lint (rv-build)
[findings]

### Architecture + Standards (rv-arch)
[findings]

### Security (rv-sec)
[findings]

### Completeness (rv-plan)
[findings]

Fix ALL findings and re-report.
```

4 ревьюера:
- **rv-build** — build, tests, lint
- **rv-arch** — architecture + standards compliance
- **rv-sec** — security
- **rv-plan** — completeness (соответствие плану и дизайну)

**Phase 2: Execute — Phase by Phase**

The Mob Loop (полная диаграмма):
```
LEAD: Assign task via SendMessage to implementer (task details from TaskGet)
    ▼
IMPLEMENTER:
    • TaskUpdate → in_progress
    • Read standards
    • Read files FULLY
    • Implement
    • go build + test + lint
    • SendMessage → Lead
    ▼
LEAD: SendMessage to ALL 4 reviewers IN PARALLEL (see §1.5 Review Protocol)
    rv-build, rv-arch, rv-sec, rv-plan
    ▼
ALL 4 REVIEWERS respond, Lead AGGREGATES verdicts
    ▼
❌ ANY FAILED              ✅ ALL PASSED
    ▼                          ▼
Lead: SendMessage          Lead: TaskUpdate → completed
to implementer             SendMessage next task
with ALL findings
from ALL agents
```

Цикл повторяется: если FAILED — implementer исправляет, снова review. Если PASSED — Lead назначает следующий task.

**Lead: Assigning a Task (формат сообщения):**
```
type: "message"
recipient: "backend"
content: |
    ## Phase [N]: [Name]

    **Service:** [service path]
    **Layer(s):** domain / usecase / repository / controller
    **Task ID:** [id from TaskCreate]

    **What to implement:**
    [Paste relevant section from plan]

    **Read these files first (FULLY):**
    — [list from plan]

    **Key standards for this phase:**
    — [specific files from /promts/be/ that are most relevant]

    **Phase acceptance criteria:**
    — [from plan]
summary: "Assigned Phase N to backend"
```

Ключевое: Lead вставляет в сообщение ВСЁ что нужно implementer'у — секцию плана, файлы для чтения, релевантные стандарты, acceptance criteria. Implementer не должен сам искать контекст.

**Lead: After Implementer Reports Done:**
1. SendMessage to all 4 persistent reviewers in parallel:
   - rv-build — provide service path
   - rv-arch — provide list of changed files
   - rv-sec — provide list of changed files
   - rv-plan — provide plan phase file path AND design docs path
2. Wait for ALL 4 to respond
3. Aggregate verdicts: if ANY agent reports ❌ → phase is REJECTED
4. If REJECTED → SendMessage combined findings from ALL agents to implementer, loop
5. If ALL PASSED → TaskUpdate task status to `completed`, then SendMessage next task assignment to implementer

**Phase 3: Final Review**

После завершения всех phase tasks (check with TaskList) — запуск Final Cross-Phase Review. Те же 4 ревьюера параллельно, но с cross-phase scope:

- **Build+Test+Lint** — full service build/test/lint (same as per-phase)
- **Architecture+Standards** — read ALL new files across ALL phases, check cross-phase consistency: unique error codes, no orphaned code, naming consistency, DI chain correctness, all VOs
- **Security** — read ALL new files across ALL phases, full security audit
- **Completeness** — read the FULL plan README.md (all phases), verify ALL phases implemented, ALL acceptance criteria met, ALL success criteria from plan/README.md checked

Additional cross-phase checks for Architecture+Standards agent:
```
- [ ] Error codes are unique across the ENTIRE module (not just this phase)
- [ ] No orphaned code from earlier iterations
- [ ] No TODO/FIXME left behind
- [ ] Naming is consistent across all new files
- [ ] All entity fields have round-trip repo tests
- [ ] No circular dependencies between new components
- [ ] DI chain is correct (init order matches dependencies)
- [ ] All Value Objects — no raw types leaked
```

Mark the final review task as `completed` after all 4 agents pass.

**Phase 4: Smoke Test**

После прохождения cross-phase review — SendMessage to implementer для smoke tests:

4.1 Start the Service:
```bash
go run ./{service-path}/cmd/... &
sleep 3
```
If service requires dependencies (MongoDB, Redis, etc.), check if docker-compose is available:
```bash
docker-compose up -d
```

**Phase 5: Handoff**

Итоговый отчёт о реализации:
```markdown
## ✅ Implementation Complete: [Feature Name]

### Phases
- ✅ Phase 1: [summary]
- ✅ Phase 2: [summary]
- ...

### Files Changed
- `path/to/file.go` — new/modified: [what]

### Quality Gates (4 Parallel Review Agents)
| Phase | Build+Test+Lint | Arch+Standards | Security | Completeness | Verdict | Rejections |
|-------|-----------------|----------------|----------|--------------|---------|------------|
| 1     | ✅              | ✅             | ✅       | ✅           | ✅      | 0          |

### Final Review
✅ Cross-phase review passed

### Smoke Test
✅ All endpoints tested — [N] passed, 0 failed

### Rejection Log
- Phase 2, attempt 1: Builder order wrong in OrderEntity → fixed

### Verification
✅ go build ./...
✅ go test ./... -count=1    — [N] tests
✅ golangci-lint run ./...    — 0 issues
✅ Smoke test                 — [N] endpoints verified

### Notes
- Plan deviations: [none / list]
- All acceptance criteria met: yes
```

Ключевые паттерны handoff:
- Таблица quality gates по каждой фазе — видно сколько rejections было
- Rejection Log — история исправлений (что было не так и что починено)
- Plan deviations — явно фиксируются отклонения от плана
- Verification — конкретные команды и их результаты

**Phase 6: Shutdown & Cleanup**

6.1 Shutdown Teammates:
```
SendMessage:
  type: "shutdown_request"
  recipient: "backend"
  content: "All phases complete. Please shut down."
```
(Repeat for each additional implementer if multiple were spawned.)
Repeat for each reviewer: rv-build, rv-arch, rv-sec, rv-plan.
Wait for each teammate to respond with `shutdown_response` (approve: true). If a teammate rejects, check why — they may still have work in progress.

6.2 Clean Up Team:
- After ALL teammates have shut down, use **TeamDelete** to clean up:
  - Removes `~/.claude/teams/{team-name}/`
  - Removes `~/.claude/tasks/{team-name}/`
- Do NOT call TeamDelete while teammates are still active — it will fail.

**Phase 7: Commit**

After ALL reviews pass (quality gates + cross-phase + smoke test) and team is shut down, create a **single local commit**.

Commit Rules:
- **Conventional Commits** format: `feat:`, `fix:`, `refactor:`, `test:`, `chore:`
- 🚫 NEVER add `Co-Authored-By` lines — STRICTLY PROHIBITED. No co-authorship, no attribution lines, no `Signed-off-by`. Commit message must contain ONLY description and summary.
- **NO push** — commit stays local until user explicitly pushes
- **Scope** in parentheses for the module: `feat(parsons-gym): add adaptive hints`

Commit Process:
1. Stage only the files changed by this feature: `git add {list of changed files}`
2. Create commit with conventional message:
```
git commit -m "feat({module}): {short description}

- {Phase 1 summary}
- {Phase 2 summary}
- {Phase N summary}

Quality: {N} phases, {N} tests, 0 lint issues
Smoke: {N} endpoints verified"
```

Commit Type Selection:

| Type     | When                                      |
|----------|-------------------------------------------|
| feat     | New feature or capability                 |
| fix      | Bug fix                                   |
| refactor | Code restructuring without behavior change|
| test     | Adding/updating tests only                |
| chore    | Build, DI, config, infrastructure changes |

### Важные уточнения из монолога автора

**Процесс в целом:**
- Весь pipeline tool-agnostic: можно Claude Code, OpenCoder, Windsurf, любой инструмент для работы с LLM. Можно даже свои модели на своём сервере (OpenRouter + свои GPU).
- Для демо использовали Opus 4.6 + Claude Teams (свежие фичи на момент видео).
- Команды изначально живут привязанными ко всему проекту, но для демо перенесены в конкретный микросервис для удобства.

**Research:**
- Субагенты research запускаются с **более слабыми моделями** — чтобы не так быстро расходовать лимиты. Не только экономия контекста, но и экономия средств.
- Для демо был один research, но в реальности их может быть несколько параллельных.

**Design:**
- Ревью дизайна — "невероятно увлекательный момент". Раньше на кодирование уходило много времени, а сейчас можно погрузиться в красивую архитектуру.
- LLM рисует sequence, C4, DFD невероятно эффективно — раньше на это уходило очень много времени вручную.
- Ревью дизайна может занимать часы, можно обсуждать на звонке — это design sessions перед фичей.
- Автор нашёл недочёты в дизайне:
  - Плохой нейминг (AvatarKey)
  - Только S3 без CDN — нужен CDN для эффективной отдачи контента
  - Ресайзинг синхронный — нужен асинхронный (оригинал на S3, потом async resize)
- Два варианта исправления: руками менять документы или просить LLM. Автор иногда делает руками, иногда просит. Важно: исправляя одно, LLM может сломать другое (редко, зависит от модели).

**Plan:**
- Автор заметил, что LLM создал `boundary service`, хотя service не прописан ни в каких промтах — пример недетерминированности. LLM решил "я же senior-architect, я знаю лучше". Именно поэтому ревью плана критично.
- Ревью каждой фазы отдельно: какие файлы создаст, какие поля добавит, как будет выглядеть.

**Implement:**
- Claude Teams позволяют действовать не последовательно, а параллельно — каждый teammate в отдельном контекстном окне.
- Lead (лид) не обязательно нуждается во всех детальных инструкциях в промпте — Claude и так будет создавать агентов и передавать задачи. Но автор добавляет дополнительные ограничения для контроля.

**Результаты демо:**
- 32 минуты на всю имплементацию (8 фаз + все ревью)
- Контекстное окно лида: 76% использовано (без compaction)
- Backend implementer: 96% использовано (4% осталось до compaction) — самый нагруженный
- Все остальные teammates тоже без compaction
- Если фаза не влезает в одно контекстное окно — можно запускать каждую фазу в отдельном teammate

**После имплементации:**
- Commit без push — чтобы можно было поревьювать
- "Ревью до отправки очень важно, LLM может галлюцинировать на любой фазе" — несмотря на все проверки
- Рекомендация: жёсткий CI с линтерами, тестами, smoke-tests, security, CVE checks
- Можно добавить в commit все сгенерированные docs — история архитектурных решений рядом с кодом
- ADR для сложных моментов — что и почему было принято

### Реальный скрин работающей агентской системы (этап implement)

**Окна tmux — 6 teammates:**
- `@main` — Lead
- `@backend` — Backend Implementer
- `@rv-arch` — Architecture Reviewer
- `@rv-build` — Build/Test/Lint Reviewer
- `@rv-plan` — Plan Completeness Reviewer
- `@rv-sec` — Security Reviewer

View teammates: `tmux -L claude-swarm-12209 a`

**Реальный flow Phase 1 → Phase 2:**

Lead (левая панель):
1. `(rv-plan) Phase 1 re-review: COMPLETE`
2. `3/4 — rv-plan ✅ COMPLETE. Just waiting for rv-build.`
3. `Waiting for rv-build.`
4. `(rv-build) Phase 1 re-review: all gates passed`
5. `All 4 reviewers passed on re-review:` rv-build ✅, rv-arch ✅, rv-sec ✅, rv-plan ✅
6. `Phase 1 APPROVED. Marking complete and assigning Phase 2.`
7. `Phase 2 is now unblocked. Also Phase 6 is unblocked (depends only on Phase 1). Assigning Phase 2 first (it unblocks Phases 3, 4, 5).`
8. `Phase 2 assigned. Waiting for backend to implement boundary interfaces and models.`
9. `Reviewers idle — expected. Waiting for backend on Phase 2.`
10. `(backend) Phase 2 done — Boundary interfaces & models`
11. `Phase 2 reported done. Sending all 4 reviewers in parallel.`
12. `All 4 reviewers dispatched for Phase 2. Waiting for verdicts.`
13. `Backend idle — expected, awaiting reviewer verdicts on Phase 2.`

Правая панель — 4 reviewer'а работают параллельно над Phase 2:
- Phase 2 Review (rv-build) — timeout 2m
- Phase 2 Architecture Review (rv-arch) — Reading 8 files
- Phase 2 Security Review (rv-sec) — Reading 8 files
- Phase 2 Review (rv-plan) — Reading 9 files

**Task list видимый в Lead:**
```
● Implementing Phase 2: Boundary Interfaces & Models. (9m 37s · 7.5k tokens)
├ 🟥 Phase 2: Boundary Interfaces & Models (@backend)
├ □ Phase 6: RepoModel + Response Update
├ □ Phase 3: ImageProcessor Adapter > blocked by #2
├ □ Phase 4: S3 FileStorage Adapter > blocked by #2
├ □ Phase 5: UseCase — UploadAvatar, DeleteAvatar, GetAvatarURLs > blocked by #2
├ □ Phase 7: Controller — Upload & Delete Handlers > blocked by #5, #6
├ □ Phase 8: Init/DI + Routing > blocked by #3, #4, #5, #7
├ □ Final Cross-Phase Review > blocked by #2, #3, #4, #5, #6, #7, #8
✓ Phase 1: AvatarKey VO + Entity Extension (@backend)
```

**Ключевые наблюдения из реального запуска:**
- "re-review" — Phase 1 была rejected первый раз, исправлена, прошла re-review
- Lead понимает зависимости: "Phase 2 unblocks Phases 3, 4, 5" — выбирает оптимальный порядок
- Lead ждёт завершения всех 4 reviewer'ов перед следующим шагом
- Ревьюеры работают реально параллельно в отдельных tmux окнах
- Task list с зависимостями (`blocked by #2`, `blocked by #5, #6`) — реальный DAG
- 7.5k tokens за 9m37s на Phase 2 — метрики видны
- Phase 6 не blocked by Phase 2 — может выполняться параллельно (зависит только от Phase 1)

- Реальный план для user-avatar: 8 фаз (phase-01.md ... phase-08.md)
- Реальная структура docs/user-avatar/:
  ```
  docs/user-avatar/
  ├── plan/
  │   ├── README.md
  │   ├── phase-01.md ... phase-08.md
  ├── 01-architecture.md     ← design views
  ├── 02-behavior.md
  ├── 03-decisions.md
  ├── 04-testing.md
  ├── 06-repo-model.md       ← conditional
  ├── 07-standards.md        ← conditional
  ├── 08-api-contract.md     ← conditional
  └── README.md              ← design overview
  ```

**Ключевые паттерны:**
- Стандарты проекта в отдельной папке `/promts/` — агент загружает их все
- Conditional documents — не фиксированный набор выходов, а зависит от задачи
- Нумерованная серия документов (05-, 06-, 07-, 08-) — единая система
- Design может дозапрашивать research если нужно
- Конкретные точки входа для architecture analysis (DI container, init files, routes, entities, usecases, adapters)

**Начало промпта design_feature.md (полнее):**

Заголовок: `# Design Feature — C4 + DFD + Sequence → Code Plan`

Роль: "You are an expert software architect designing a feature using the C4 model with human-in-the-loop approval."

**Core principle:** Design WHAT and WHY before HOW. No code planning until architecture is approved.

**Phase 0: Understand the Mission**

0.1 Parse Arguments:
- `$ARGUMENTS[0]` — feature name (slug, used for directory name)
- `$ARGUMENTS[1]` — service path (e.g. `microservices/name` or `front`)
- `$ARGUMENTS[2+]` — feature description, requirements, or ticket link (everything after service path)
- Если аргументы отсутствуют — спросить:
  1. Feature name (slug)
  2. Service path
  3. Feature description or ticket link
- Если только name и path без description — спросить описание перед продолжением

0.2 Read the Feature Request:
- Read the feature description/ticket provided by the user
- Understand the **business goal** — what problem does this solve?
- Identify acceptance criteria
- Determine if this touches Backend, Frontend, or infrastructure

0.3 Read Project Standards & Discover Codebase Structure:
- Read ALL project standards (all files in `/promts/`)
- Backend (Go): Architecture Layers, Clean Architecture, Domain Model, Builder, RepoModel, Go Style...

0.4 Decide if Research is Needed (то что было на предыдущем скрине)

**Ключевые паттерны:**
- Аргументы формализованы: slug + service path + description — позволяет вызывать одной строкой
- Graceful degradation: если аргументы пропущены — спрашивает, не падает
- Business goal первичен: "what problem does this solve?" — не "what to build"
- Acceptance criteria выявляются на фазе 0 — до любого проектирования
- Backend/Frontend/infra определяется сразу — влияет на дальнейший процесс
- "No code planning until architecture is approved" — жёсткое ограничение

### Специализация по типу задачи
- [ ] Разделение промптов на фича/баг — у бага другой процесс: воспроизведение, root cause, минимальное исправление
- [ ] Разделение промптов на бэкенд/фронтенд — разные стеки, разная архитектура, разные подходы к ресёрчу и дизайну
- [ ] Рефакторинг как отдельный тип задачи
- [ ] Матрица команд: тип задачи × стек (как в образце)

### Субагенты на этапах агентской системы
- [ ] Разделение на agents/ (роли) и commands/ (труба) — commands вызывают agents через subagent_type
- [ ] Research: лид-команда декомпозирует задачу на 2-4 направления, субагенты (`codebase-researcher`) сканируют параллельно, лид синтезирует
- [ ] Правила параллелизма: max 4 параллельных (context overflow), sequential при зависимостях, background для широких поисков
- [ ] Каждый таск субагенту: конкретный вопрос + стартовые файлы + формат выхода + scope boundaries (что НЕ делать)
- [ ] Implement: команда агентов — лид, кодер, ревьюер, тестировщик с разделением ответственности
- [ ] blast-radius-analyzer — субагент для оценки влияния изменений перед имплементацией
- [ ] bug-tracer — субагент для трассировки багов (воспроизведение, поиск root cause)
- [ ] Использование более дешёвых моделей для субагентов, которые только ищут/открывают файлы

### Образец промпта субагента codebase-researcher (из agents/)

Это промпт для субагента, который вызывается из research_codebase.md. Считаем образцом правильного подхода.

**Роль (жёсткая):**
"You are a codebase research specialist. Your job is to find facts, trace code paths, and document what exists — nothing more."

**Rules:**
- ONLY describe what EXISTS in the code. No suggestions, no critique, no improvements, no subjective opinions.
- Every claim must include exact `file_path:line_number` references.
- Read files COMPLETELY — never use limit/offset.
- When unsure, read more code. Never guess.

**Research Process (формализован):**
1. Start from the entry point (file, function, or concept given to you)
2. Trace dependencies outward — imports, interfaces, implementations
3. Map the data flow: input → processing → output
4. Identify patterns: what conventions does the code follow?
5. Document your findings with exact references

**Output Format (структурирован):**

```
### Summary
2-3 sentences describing what you found.

### Findings
For each component/area:
- **Location**: `path/to/file.go:42-89`
- **What it does**: factual description
- **Key dependencies**: what it imports/uses
- **Patterns**: conventions observed

### Code References
Bullet list of `file:line` — description pairs.
```

**Ключевые паттерны:**
- Субагент начинает с entry point и трассирует наружу (а не пытается покрыть всё)
- Обязательные `file:line` ссылки в каждом утверждении
- Файлы читаются ЦЕЛИКОМ (без limit/offset) — чтобы не пропустить контекст
- Если не уверен — читай больше кода, не угадывай
- Выходной формат жёстко задан: Summary → Findings → Code References

### Delta Update (Flow B)
- [ ] Механизм обновления агентской системы при обнаружении проблем
- [ ] Анализ проблемы → impact analysis → обновление промптов
- [ ] Changelog изменений в агентской системе

### Quality Gates
- [ ] Автоматический запуск quality gates через CI/CD хуки
- [ ] Кастомные quality gates на основе специфики проекта
- [ ] Метрики эффективности агентской системы
