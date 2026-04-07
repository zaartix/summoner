# Wizard — TODO

Потенциальные улучшения, отложенные в пользу простоты первой версии.
Детальные образцы промптов, форматов и примеры — в [SOURCES.md](SOURCES.md).

## Визард

### Контекст и производительность
- [ ] Разделение контекста между фазами визарда — каждая фаза в отдельном субагенте с передачей результата через файлы, чтобы не забивать основной контекст
- [ ] Сохранение project-context.md после фазы 0 (понимание проекта) для переиспользования и экономии контекста
- [ ] Cross-check review (привлечение других моделей через скилл cross-check) на критических этапах принятия решений визардом

### Валидация
- [ ] Итоговая валидация — после генерации всех промптов показать человеку summary и дать возможность скорректировать
- [ ] Dry-run — прогнать сгенерированную агентскую систему на тестовой задаче для проверки качества промптов

### Генерация стандартов проекта
- [ ] Визард генерирует файлы стандартов проекта (аналог `/promts/` из образца) — отдельные файлы по аспектам: архитектура, стиль кода, тестирование, domain model, etc. → [SOURCES.md: "Стандарты проекта загружаются из файлов"](SOURCES.md)

---

## Агентская система

### Структура результата
- [ ] Разделение на `agents/` (роли-субагенты) и `commands/` (труба) — commands вызывают agents через `subagent_type` → [SOURCES.md: "Планируемая структура результата"](SOURCES.md)
- [ ] Папка `agent-memory/` для памяти агентов между сессиями
- [ ] `/clear` между этапами трубы — каждый этап в чистом контексте → [SOURCES.md: "Переход между этапами трубы"](SOURCES.md)

### Специализация по типу задачи
- [ ] Разделение промптов на фича/баг — у бага другой процесс
- [ ] Разделение промптов на бэкенд/фронтенд — разные стеки, разные промпты, разные стандарты (`/promts/be/` vs `/promts/fe/`)
- [ ] Рефакторинг как отдельный тип задачи
- [ ] Матрица команд: тип задачи × стек → [SOURCES.md: "Специализация commands/ по двум осям"](SOURCES.md)

### Research (этап 1)
- [ ] Параллельные субагенты: лид декомпозирует на 2-4 направления, `codebase-researcher` сканирует, лид синтезирует → [SOURCES.md: "Паттерны из промпта research_codebase.md"](SOURCES.md)
- [ ] Max 4 параллельных задач (context overflow risk)
- [ ] Формат выхода: YAML frontmatter (git commit, branch), Summary, Findings по компонентам, Code References → [SOURCES.md: "Формат итогового research-документа"](SOURCES.md)
- [ ] Good/Bad примеры в промпте для задания уровня конкретности → [SOURCES.md: "Good vs Bad Research"](SOURCES.md)
- [ ] Более дешёвые модели для субагентов research

### Design (этап 2)
- [ ] Multi-file by view: 01-architecture, 02-behavior, 03-decisions, 04-testing + conditional 05-08 → [SOURCES.md: "Rules (15 правил design_feature)"](SOURCES.md)
- [ ] Conditional documents по типу задачи (events, entities, standards, api-contract) → [SOURCES.md: "Conditional documents"](SOURCES.md)
- [ ] Design может дозапрашивать research через субагентов (3 типа задач: Architecture Analysis, Pattern Discovery, Integration Analysis) → [SOURCES.md: "Research Task 1/2/3"](SOURCES.md)
- [ ] Два approval gate: design И code plan — оба до имплементации
- [ ] Аргументы формализованы: `slug + service-path + description` → [SOURCES.md: "Phase 0: Understand the Mission"](SOURCES.md)

### Plan (этап 3)
- [ ] Phase Order Strategy: bottom-up / adapter-first / vertical slice → [SOURCES.md: "Phase Order Strategy"](SOURCES.md)
- [ ] plan/README.md с таблицей фаз, зависимостями, статусами → [SOURCES.md: "Формат plan/README.md"](SOURCES.md)
- [ ] Отдельные файлы phase-01.md...phase-NN.md — self-contained, один файл на assignment

### Implement (этап 4)
- [ ] Claude Teams: Lead + Backend Implementer + 4 Reviewer'а (rv-build, rv-arch, rv-sec, rv-plan) → [SOURCES.md: "Образец промпта implement_backend.md"](SOURCES.md)
- [ ] Delegate Mode для Lead — технически запрещает писать код
- [ ] Mob Loop: assign → implement → 4 parallel reviews → approve/reject → next → [SOURCES.md: "The Mob Loop"](SOURCES.md)
- [ ] Task DAG с зависимостями через `addBlockedBy`
- [ ] Final Cross-Phase Review с 8 конкретными чеклист-пунктами → [SOURCES.md: "Phase 3: Final Review"](SOURCES.md)
- [ ] Smoke Test — реальный запуск сервиса → [SOURCES.md: "Phase 4: Smoke Test"](SOURCES.md)
- [ ] Handoff report с таблицей quality gates, rejection log, plan deviations → [SOURCES.md: "Phase 5: Handoff"](SOURCES.md)
- [ ] Shutdown teammates + TeamDelete cleanup
- [ ] Conventional Commits, NO Co-Authored-By, NO push → [SOURCES.md: "Phase 7: Commit"](SOURCES.md)

### Delta Update (Flow B)
- [ ] Механизм обновления агентской системы при обнаружении проблем
- [ ] Анализ проблемы → impact analysis → обновление промптов
- [ ] Changelog изменений в агентской системе

### Quality Gates
- [ ] Автоматический запуск quality gates через CI/CD хуки
- [ ] Кастомные quality gates на основе специфики проекта
- [ ] Метрики эффективности агентской системы
