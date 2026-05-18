# KASTOL

> Минималистичный пульт управления ИИ-агентами, которые работают на твоём ПК и решают любые задачи (код, ресёрч, рутина), а ты можешь подключаться к ним хоть с десктопа, хоть из Телеги, хоть голосом.

Документ — единое ТЗ + roadmap + архитектура. Это исходник, по которому проект будет вайб-кодиться.

---

## 1. Что это и зачем

### Одной фразой
Локальный оркестратор multi-agent ИИ-команд + единая точка управления любым ИИ-сценарием на твоём ПК.

### Сценарий использования
1. Открываешь приложение
2. Описываешь задачу ("сделай 2D-платформер на Pygame")
3. Проходишь визард: провайдер → модели → пресет команды агентов → рабочая папка
4. Лёгкая модель-планер выдаёт **план** — ты его правишь и аппрувишь
5. Жмёшь Run — несколько агентов с разными характерами совещаются, спорят, пишут код
6. В реальном времени видишь:
   - переписку (кто что сказал)
   - что каждый делает прямо сейчас (читает файл, пишет, думает)
   - артефакты (созданные файлы, диффы)
   - стоимость в токенах/$
7. Можешь в любой момент: поставить на паузу, кинуть сообщение, ответить кому-то лично, добавить глобальное правило в живую память миссии, форкнуть ветку диалога
8. На выходе — готовый результат в рабочей папке + полный лог в KASTOL

### Ключевые принципы
1. **Local-first** — всё крутится на твоём ПК, никаких облаков, никакой регистрации
2. **Bring your own keys** — твои API-ключи, лежат в `.env` локально
3. **Один backend → много фронтов** — Web сегодня, TG-бот завтра, голос послезавтра, десктоп потом
4. **Безопасность по умолчанию** — ИИ не может дропнуть систему, опасные действия требуют OK, опционально песочница в Docker
5. **Прозрачность** — видно каждое сообщение, каждый tool call, каждую копейку
6. **Бесплатно и open source** — монетизации нет

### Целевой пользователь
Я (автор) и друзья. Не коммерческий продукт.

---

## 2. Полный скоуп фич

### A. Конфигурация
- **Менеджер провайдеров**: OpenAI, Anthropic, Gemini, Groq, Ollama, LM Studio, OpenRouter — всё через LiteLLM (один интерфейс на ~100 провайдеров)
- **Менеджер агентов**: имя, роль, системный промпт (характер), модель, инструменты, температура
- **Менеджер пресетов команд** — готовые наборы:
  - Code Squad — Architect + Coder + Critic + Tester
  - Debate — Optimist + Skeptic + Moderator (для дизайн-решений)
  - Researcher — Searcher + Analyst + Writer
  - + свои собственные
- **Workflow**: sequential / debate (N раундов) / manager-worker / свой граф
- **Шаринг пресетов** через гитхаб гист — кинул ссылку, KASTOL импортнул

### B. Запуск миссии
- Указание workspace (рабочая папка)
- Описание задачи в свободной форме
- **Plan Review** — отдельный экран перед запуском:
  - Лёгкая модель-планер выдаёт план без ограничения по пунктам
  - Можно редактировать каждый пункт, добавлять/удалять/переставлять
  - Можно помечать пункты "needs human approval" (миссия паузнётся когда дойдёт)
  - Свободный комментарий "учти вот это"
  - План становится живым документом, агенты с ним сверяются
- Live-стрим переписки
- Кнопки: Pause / Resume / Inject message / Stop / Fork (ответвить от точки)
- Реалтайм счётчик токенов и денег

### C. Прозрачность процесса
**ChatStream** — основной чат:
- Каждое сообщение каждого агента в реальном времени
- Цвет/аватар на агента
- Под сообщением — какие tool calls были, можно развернуть и посмотреть аргументы и output
- Бейджи: токены, время, стоимость

**Activity Sidebar** — что происходит прямо сейчас:
- "Architect → пишет ответ..."
- "Coder → редактирует `player.py`"
- "Critic → читает `level.json`"
- Текущий файл подсвечивается, diff в реальном времени

**Artifacts Panel** — все результаты миссии в одном месте: созданные файлы, скриншоты, диффы, ссылки. Кликнул — открыл, не надо рыться в чате.

### D. Вмешательство в диалог
- **Inject message** — кинуть сообщение от своего лица, агенты увидят на следующем шаге
- **Reply to message** — выделил сообщение конкретного агента, ответил лично (остальные тоже видят)
- **Global Rules** — глобальные правки в живой памяти миссии: "ВСЕМ: используем Godot, забыли про Pygame". Все агенты с этого момента видят это правило
- **Pin/Unpin** — пин важных сообщений в верх контекста
- **Edit & branch** — отредактировал чужое сообщение → создаётся форк ветки от этой точки

### E. Разные агенты на одной модели (важно!)
Чтобы они реально спорили а не подпевали — пять механик:
1. **Personality engine** — у каждого агента жёсткий характер в system prompt
   - Architect: опытный сварливый сеньор, любит чистую архитектуру, ненавидит over-engineering
   - Critic: параноик, обязан найти минимум 3 проблемы, никогда не соглашается сразу
   - Coder: прагматик, важно чтобы работало, теории не любит
2. **Forced disagreement protocol** — для роли Critic жёсткое правило: запрещено соглашаться с первой итерацией, минимум 2 раунда возражений
3. **Разные температуры** — Architect 0.3, Critic 0.7, Coder 0.2
4. **Изолированные view контекста** — каждый агент видит не весь диалог, а свою выжимку (Coder — код + последний план; Critic — только результат). Вынуждает мыслить независимо
5. **Devil's Advocate режим** — после каждого консенсуса принудительно вызывается агент-адвокат дьявола: "найди слабое место в этом решении"

### F. Инструменты для агентов (tools)
- **Файлы**: read_file, write_file, list_dir, edit_file (внутри workspace, наружу — нельзя)
- **Шелл**: run_command — whitelist по умолчанию, опасные команды требуют OK
- **Веб**: web_search, fetch_url
- **Код**: run_python, run_tests
- **Память**: запись заметок в общую память миссии
- **Зрение** (фаза 4): take_screenshot + vision-модели
- **Управление ПК** (фаза 4): mouse, keyboard, через pyautogui

### G. История и replay
- Все запуски в SQLite
- Открыть старый запуск, посмотреть весь диалог
- **Resume / Continue this** — закрыл ноут на середине миссии → утром один клик "продолжить"
- **Fork** — взять старый запуск, поменять промпт на середине, посмотреть альтернативную ветку

### H. Удалёнка (фаза 3)
- TG-бот: команда "сделай X" — KASTOL запускает миссию на твоём ПК, шлёт прогресс в чат, при необходимости спрашивает подтверждение
- Discord-бот (опц.)
- Голосовой ввод через локальный Whisper

### I. Quick-call оверлей (фаза 3)
- Глобальный хоткей (например `Ctrl+Shift+Space`)
- Всплывает мини-окошко (как Spotlight на маке)
- Голос или текст
- Выбираешь "к кому": конкретному агенту из активной миссии / быстрому помощнику / новой мини-миссии
- Ответ голосом (TTS) или текстом
- Закрылось → исчезло, не мешает

### J. Безопасность
- Whitelist/blacklist команд
- Dry-run режим (агенты пишут план, не исполняют)
- Approval-mode (каждое действие требует "да")
- Опциональная Docker-песочница для шелла
- Полный аудит-лог всех действий
- **Ghost mode** — агенты только читают, ничего не пишут
- **Rate limiter на tool calls** — не больше N команд подряд без OK (защита от бесконечного цикла)
- **Diff на global rules** — если агент пытается изменить правило, алерт юзеру
- **Auto-snapshot в git** — перед каждым опасным действием автокоммит в скрытую ветку, легко откатить

### K. UX-приятности
- **Smart Context** — автосжатие истории, фоновый агент сжимает старые куски в саммари, важное пинит
- **Diff viewer перед записью** — agent собирается перезаписать файл → всплывает дифф → apply / reject / edit
- **Cost guard** — лимит на миссию ($X или Y токенов), подходит к лимиту → пауза → "продолжаем?"
- **Hotkey palette (Cmd+K)** — командная палитра, поиск по миссиям/агентам/файлам/действиям
- **Templates задач** — часто запускаемые штуки сохраняешь как шаблон, в следующий раз 1 клик
- **Мульти-миссии параллельно** — несколько миссий одновременно, в UI вкладки
- **Notes слой** — твои личные заметки к миссии или сообщению (агенты их не видят)
- **Voice notes для команд** — запись голосом → Whisper → автоматически становится global rule миссии
- **"Explain this decision"** — кнопка на любом сообщении: "почему ты так решил?" → мини-запрос к этому же агенту
- **Agent benchmark mode** — даёшь команде задачу-эталон, сравниваешь конфиги команд по времени/деньгам/качеству
- **Estimated time** — "эта миссия займёт ~7 минут" перед запуском
- **Mood meter** — индикатор расхождения с консенсусом ("Critic в ярости", "Coder спокоен")
- **Звуки** — блип когда миссия закончилась, тук-тук когда нужен ответ
- **System tray** — иконка в углу со статусом, клик → мини-чат
- **Темы** — dark/light/custom (JSON с цветами, можно делиться)

### L. Что НЕ делаем
- Свой LLM не тренируем
- Облачную версию не пилим
- Регистрацию/мульти-юзеров не делаем
- Не пытаемся быть Cursor'ом — это не редактор, это **дирижёр агентов**
- Мобильного приложения нет — мобилка через TG-бот

---

## 3. Архитектура

### Принцип №1: Один backend = ядро. Все клиенты — снаружи.

```
              ┌──────────────────────────────┐
              │        KASTOL Backend        │
              │  FastAPI + LangGraph + WS    │
              │  ┌────────────────────────┐  │
              │  │  Event Bus (pub/sub)   │  │
              │  └────────────────────────┘  │
              └──────────────────────────────┘
                    ▲   ▲   ▲   ▲   ▲
                    │   │   │   │   │
        ┌───────────┘   │   │   │   └──────────┐
        │               │   │   │              │
   ┌────────┐    ┌────────┐ │ ┌─────────┐  ┌──────────┐
   │  Web   │    │Tauri   │ │ │ TG bot  │  │ Overlay  │
   │ (Next) │    │desktop │ │ │ aiogram │  │ + voice  │
   └────────┘    └────────┘ │ └─────────┘  └──────────┘
                            │
                       ┌─────────┐
                       │ CLI/API │
                       └─────────┘
```

Ключ: **WebSocket-канал событий** — единый. Любой клиент подключается, видит миссию вживую, может слать команды.

### Принцип №2: Каждая фича = новый файл, не правка старого
Под вайб-кодинг:
- Tools — каждый в своём файле, регистрация через `@tool` декоратор
- Agents — JSON-файл характера + опц. кастомный класс
- Workflows — отдельный файл-граф
- Providers — через LiteLLM, ничего писать не надо
- Конфиги — JSON/YAML, без рекомпиляции
- Логи — с первого коммита, подробные

Когда захочешь "добавь агента-ресёрчера" — создаёшь один JSON-файл. Не 5 правок в 5 местах.

### Стек

| Слой | Технология | Почему |
|---|---|---|
| Backend | Python 3.11 + FastAPI + WebSockets | Лучшая экосистема для LLM |
| Оркестрация | LangGraph | Граф-ориентированная (агенты = узлы), контролируемее CrewAI/AutoGen |
| LLM-клиент | LiteLLM | Один интерфейс на ~100 провайдеров |
| Frontend | Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui + Zustand | Минималистично, дорого выглядит без дизайнера |
| Хранилище | SQLite через SQLModel | Файлик, ноль настройки |
| Песочница | Docker (опц.) + dry-run + whitelist | Не сжигаем твой пк |
| TG-бот (фаза 3) | aiogram | Подцепится к тому же FastAPI |
| Голос (фаза 3) | Whisper (локально) | Никаких облаков |
| Десктоп (фаза 4) | Tauri | Один лёгкий экзешник, обернёт Next + Python sidecar |

### Доменная модель

```
Mission (миссия = одна задача)
├── Workspace (папка где живёт код/файлы)
├── Providers[] (ключи к провайдерам)
├── Agents[]
│   ├── role (architect, coder, critic, tester...)
│   ├── model (gpt-4o, claude-sonnet, qwen-local...)
│   ├── system_prompt (личность)
│   ├── tools[] (read_file, write_file, run_shell...)
│   └── temperature
├── Plan (живой документ, ревьювится перед запуском)
├── Workflow (sequential / debate / manager-worker)
├── Rules[] (global rules в живой памяти миссии)
├── Run[] (запуски — лог сообщений, артефакты, стоимость)
│   ├── Messages[] 
│   ├── ToolCalls[]
│   ├── Artifacts[]
│   └── Cost
└── Notes[] (твои личные заметки, агенты не видят)
```

### Структура backend

```
backend/app/
├── main.py
├── core/
│   ├── config.py          # pydantic-settings, .env
│   ├── db.py              # SQLModel + SQLite
│   ├── events.py          # внутренний event bus → WS
│   └── logging.py
├── models/                # Mission, Run, Message, Agent, Provider, 
│                          # Plan, Artifact, Note, Rule, Preset
├── api/
│   ├── missions.py
│   ├── plans.py
│   ├── agents.py
│   ├── providers.py
│   ├── presets.py
│   ├── notes.py
│   └── ws.py              # WebSocket
├── llm/
│   ├── client.py          # litellm wrapper
│   └── budget.py          # cost guard
├── agents/
│   ├── base.py
│   ├── runner.py          # tool-loop
│   └── personalities/     # JSON: architect.json, critic.json, ...
├── workflows/             # LangGraph графы
│   ├── sequential.py
│   ├── debate.py
│   └── manager_worker.py
├── tools/                 # каждый tool — свой файл
│   ├── registry.py
│   ├── files.py
│   ├── shell.py           # phase 2
│   ├── web.py             # phase 2
│   ├── git.py             # auto-snapshot
│   └── vision.py          # phase 4
├── planner/
│   └── service.py
├── memory/
│   ├── context.py         # smart auto-compress
│   └── rules.py           # global rules
├── safety/
│   ├── whitelist.py
│   ├── rate_limit.py
│   └── confirm.py
└── integrations/          # phase 3+
    ├── telegram.py
    └── overlay.py
```

### Структура frontend

```
frontend/
├── app/                   # Next.js App Router
│   ├── page.tsx                    # dashboard
│   ├── missions/[id]/page.tsx
│   ├── missions/new/page.tsx       # визард + Plan Review
│   ├── agents/page.tsx
│   ├── providers/page.tsx
│   └── presets/page.tsx
├── components/
│   ├── mission/
│   │   ├── ChatStream.tsx
│   │   ├── ActivitySidebar.tsx
│   │   ├── ArtifactsPanel.tsx
│   │   ├── DiffViewer.tsx
│   │   ├── PlanReview.tsx
│   │   ├── RulesEditor.tsx
│   │   └── ConfirmDialog.tsx
│   ├── ui/                          # shadcn
│   ├── HotkeyPalette.tsx           # Cmd+K
│   ├── CostBadge.tsx
│   └── ThemeProvider.tsx
├── lib/
│   ├── api.ts
│   ├── ws.ts                        # подписка на события
│   ├── stores/                      # zustand
│   └── types.ts                     # автогенерация из OpenAPI
└── public/sounds/
```

### Контракт событий (Event Bus)

Всё что происходит на бэке — улетает в шину. Любой клиент слушает.

```
mission.started / paused / resumed / finished
plan.generated / updated / approved
agent.thinking / message / tool_call / tool_result
artifact.created / updated
cost.update
rule.added / removed / changed
confirmation.requested / resolved
```

---

## 4. Roadmap по фазам

### Phase 1 — Foundation MVP
Цель: ты пишешь "сделай платформер на pygame", смотришь как 3 агента спорят и пишут код, на выходе работающий проект.

| # | Фича |
|---|---|
| 1.1 | Скаффолд (repo, env, README, .kiro/steering) |
| 1.2 | Backend core (FastAPI, WS, SQLite, event bus) |
| 1.3 | Provider manager через LiteLLM |
| 1.4 | Tool registry + file tools (read/write/list/edit) |
| 1.5 | Agent runner + 3 характера (Architect / Coder / Critic) с разными темпами и изолированными view контекста |
| 1.6 | Sequential workflow на LangGraph |
| 1.7 | Planner шаг + Plan Review screen |
| 1.8 | ChatStream (видно каждое сообщение, tool calls, токены) |
| 1.9 | ActivitySidebar (что делает каждый агент сейчас) |
| 1.10 | Artifacts panel |
| 1.11 | Diff viewer перед записью |
| 1.12 | Inject message + Global rules editor |
| 1.13 | Cost guard |
| 1.14 | Auto-snapshot в git |
| 1.15 | Resume / Continue this |
| 1.16 | Hotkey palette (Cmd+K) базовый |
| 1.17 | Темы (dark/light) |

### Phase 2 — Глубина команды
Цель: агенты реально срутся, ты можешь вмешиваться хирургически.

- 2.1 Debate workflow + Devil's Advocate
- 2.2 Manager-worker workflow
- 2.3 Reply to message, Pin, Edit & branch
- 2.4 Шелл tool с подтверждением + safety whitelist
- 2.5 Web search/fetch tools
- 2.6 Smart context auto-compress
- 2.7 Templates задач
- 2.8 Notes слой
- 2.9 "Explain this decision" кнопка
- 2.10 Ghost mode + Rate limiter + Rule diff alert
- 2.11 Preset marketplace через гист-шаринг
- 2.12 Estimated time

### Phase 3 — Удалёнка и быстрый доступ
Цель: ты не за компом, но KASTOL делает что нужно.

- 3.1 TG-бот (запуск миссий, прогресс, подтверждения)
- 3.2 Quick-call оверлей + хоткей
- 3.3 Voice input (Whisper локально) + Voice notes → global rules
- 3.4 TTS ответы
- 3.5 System tray иконка с мини-чатом
- 3.6 Звуки уведомлений
- 3.7 Discord-бот (опц.)

### Phase 4 — Десктоп и зрение
Цель: агенты видят экран и могут управлять пк.

- 4.1 Tauri обёртка → один экзешник
- 4.2 Vision tools (screenshot, vision модели через LiteLLM)
- 4.3 Computer Use агент (мышь/клава, kill-switch, scoped окна)
- 4.4 Docker sandbox для шелла
- 4.5 Параллельные миссии — UI вкладок (бэк уже умеет с фазы 1)
- 4.6 Agent benchmark mode
- 4.7 Mood meter

---

## 5. План работ — Phase 1 разбит на PR

Делать буду серией маленьких PR-ов в отдельные ветки, чтобы можно было смотреть и тыкать постепенно.

| PR | Что входит | Когда работает |
|---|---|---|
| **PR1: Skeleton** | репо, .env.example, docker-compose, .kiro/steering, README, скрипты запуска | `make dev` поднимает backend + frontend пустыми |
| **PR2: Backend core** | FastAPI, SQLite, models, event bus, provider manager, /api/health | Можно вбить ключ, увидеть провайдера в БД |
| **PR3: Agents + tools** | tool registry, file tools, agent runner с tool-loop, 3 характера в JSON | CLI: запустить одного агента с задачей "запиши hello.txt" |
| **PR4: Workflow + Planner** | Sequential workflow, planner service, REST для миссий | API: POST /missions → план → run → файл создан |
| **PR5: Frontend skeleton** | Next.js + shadcn, страницы провайдеров/агентов/миссий, WS-клиент | UI с пустыми экранами и работающей навигацией |
| **PR6: Mission live UI** | ChatStream, ActivitySidebar, ArtifactsPanel, Plan Review, DiffViewer | Полный сценарий: создал миссию в UI → смотришь как агенты работают |
| **PR7: Control & safety** | Inject, Global rules editor, Cost guard, Auto-snapshot git, Resume | Можешь вмешиваться, не страшно жать Run |
| **PR8: Polish** | Hotkey palette, темы, мелкая шлифовка | Phase 1 готов, можно дог-фудить |

После каждого PR — пишу "готов PR такой-то, посмотри / запусти у себя", даёшь добро или поправки.

---

## 6. Правила проекта (steering)

В `.kiro/steering/project.md` сразу будут:
- **Стиль кода**: Python — ruff + black; TS — prettier + eslint
- **Архитектурные правила**: новый tool = новый файл с `@tool` декоратором; новый агент = JSON-файл в `personalities/`; новый workflow = отдельный файл-граф
- Не делаем тестов без явной просьбы
- Каждая фича = отдельная ветка + PR
- Имена коммитов: `feat:`, `fix:`, `chore:` (Conventional Commits)
- Local-first, никакой телеметрии
- Логи с первого коммита, подробные
- Всё конфигурируется через JSON/YAML, без рекомпиляции

---

## 7. Что НЕ изобретаем (не путаемся с чужим)

| Сравнение | Что у них | Почему мы не они |
|---|---|---|
| CrewAI / AutoGen / LangGraph | Фреймворки оркестрации | Используем как библиотеку (LangGraph), не пытаемся переписать |
| Aider / Cursor / Cline | AI пишет код в редакторе | Мы не редактор, мы дирижёр команд агентов |
| Open Interpreter / Computer Use | AI рулит компом | Берём в фазу 4 как один из режимов, не центральная фича |
| ChatGPT Desktop / Claude Desktop | Один чат | У нас команды агентов спорят между собой |

Наша **уникальная ниша** — UX-обёртка с минималистичным визардом, прозрачностью спора агентов, удалёнкой через мессенджер и Quick-call оверлеем.

---

## 8. Финальные вопросы перед стартом кода

1. **Имя — `KASTOL` финал или меняем?**
2. **Какой провайдер у тебя реально подключен?** (OpenAI / Anthropic / OpenRouter / Ollama / нет пока) — нужно для дефолтов в `.env.example`
3. **OK что я начну в одну ветку `feat/skeleton`** и буду пушить по PR-ам как описано выше?

Когда даёшь добро — стартую с **PR1: Skeleton**.
