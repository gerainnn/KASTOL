# KASTOL

> Минималистичный пульт управления ИИ-агентами, которые работают на твоём ПК и решают любые задачи (код, ресёрч, рутина), а ты можешь подключаться к ним хоть с десктопа, хоть из Телеги, хоть голосом.

Документ — единое ТЗ + roadmap + архитектура + дизайн-система. Это исходник, по которому проект вайб-кодится.

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
3. **Один backend → много фронтов** — Web, TG-бот, голос, десктоп
4. **Безопасность по умолчанию** — ИИ не может дропнуть систему, опасные действия требуют OK, опционально песочница в Docker
5. **Прозрачность** — видно каждое сообщение, каждый tool call, каждую копейку
6. **Бесплатно и open source** — монетизации нет
7. **Расширяемость** — MCP-серверы, плагины, шаблоны и пресеты как граждане первого класса
8. **Долговременный контекст** — KASTOL помнит про тебя и про твои папки между миссиями

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
- **Personality library** — папка `personalities/` с 15-20 готовыми персонажами (Senior Python Dev, Junior Eager, Pragmatic PM, Pedantic Linter и т.д.), юзер комбинирует

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
- Цвет/полоска на агента
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

### E. Разные агенты на одной модели
Чтобы они реально спорили, а не подпевали — пять механик:
1. **Personality engine** — у каждого агента жёсткий характер в system prompt
2. **Forced disagreement protocol** — для роли Critic жёсткое правило: запрещено соглашаться с первой итерацией, минимум 2 раунда возражений
3. **Разные температуры** — Architect 0.3, Critic 0.7, Coder 0.2
4. **Изолированные view контекста** — каждый агент видит не весь диалог, а свою выжимку. Вынуждает мыслить независимо
5. **Devil's Advocate режим** — после каждого консенсуса принудительно вызывается агент-адвокат дьявола

### F. Инструменты для агентов (tools)
- **Файлы**: read_file, write_file, list_dir, edit_file (внутри workspace, наружу нельзя)
- **Шелл**: run_command — whitelist по умолчанию, опасные команды требуют OK
- **Веб**: web_search, fetch_url
- **Код**: run_python, run_tests
- **Память**: запись заметок в общую память миссии
- **Knowledge**: search_knowledge — поиск по локальной KB миссии (RAG, см. раздел S)
- **Зрение** (фаза 4): take_screenshot + vision-модели
- **Управление ПК** (фаза 4): mouse, keyboard, через pyautogui
- **MCP-серверы** (см. раздел O) — любые tools оттуда автоматически доступны

### G. История и replay
- Все запуски в SQLite
- Открыть старый запуск, посмотреть весь диалог
- **Resume / Continue this** — закрыл ноут на середине миссии → утром один клик "продолжить"
- **Fork** — взять старый запуск, поменять промпт на середине, посмотреть альтернативную ветку
- **Mission tree** — визуальный граф форков с превью каждой ветки

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
- **Rate limiter на tool calls** — не больше N команд подряд без OK
- **Diff на global rules** — если агент пытается изменить правило, алерт
- **Auto-snapshot в git** — перед каждым опасным действием автокоммит в скрытую ветку
- **Watchdog на циклы** — 3 одинаковых tool call подряд → пауза + вопрос юзеру
- **Validators** — tool-ответы проходят валидацию схемой, невалид → retry с пояснением

### K. UX-приятности
- **Smart Context** — автосжатие истории, фоновый агент сжимает старые куски в саммари
- **Diff viewer перед записью** — agent собирается перезаписать файл → всплывает дифф → apply / reject / edit
- **Cost guard** — лимит на миссию ($X или Y токенов), пауза перед лимитом
- **Hotkey palette (Cmd+K)** — командная палитра, поиск по миссиям/агентам/файлам/действиям
- **Templates задач с параметрами** — `scaffold {language} {framework} app named {name}`, заполняешь форму, запускаешь
- **Мульти-миссии параллельно** — несколько миссий одновременно, в UI вкладки
- **Notes слой** — твои личные заметки к миссии или сообщению (агенты их не видят)
- **Voice notes для команд** (фаза 3) — запись голосом → Whisper → автоматически становится global rule
- **"Explain this decision"** — кнопка на любом сообщении: "почему ты так решил?"
- **Agent benchmark mode** — даёшь команде задачу-эталон, сравниваешь конфиги
- **Estimated time** — "эта миссия займёт ~7 минут" перед запуском
- **Mood meter** — индикатор расхождения с консенсусом
- **Звуки** — блип когда миссия закончилась, тук-тук когда нужен ответ (по дефолту выкл)
- **System tray** — иконка в углу со статусом, клик → мини-чат
- **Темы** — light/dark (custom через JSON, можно делиться)
- **Inline help** — `?` иконки у каждой настройки, на hover короткое объяснение. `Cmd+/` → глобальная справка
- **Cost dashboard** — долгосрочная аналитика расходов: за месяц, по агентам, самые дорогие миссии
- **Comparison / A/B mode** — запустить ту же миссию двумя командами параллельно, side-by-side сравнение результатов
- **Coach / Meta-agent** (опц.) — наблюдает за командой и шлёт мета-фидбек юзеру ("вы 3 раза переписали один файл, что-то не так")

### L. Что НЕ делаем
- Свой LLM не тренируем
- Облачную версию не пилим
- Регистрацию/мульти-юзеров не делаем
- Не пытаемся быть Cursor'ом — это не редактор, это **дирижёр агентов**
- Мобильного приложения нет — мобилка через TG-бот

---

## 3. Onboarding (первый запуск)

Что юзер видит когда первый раз открыл KASTOL:

1. **Welcome screen** — короткое описание, одна кнопка "Get started"
2. **Подключение провайдера** — форма с автодетектом (Ollama/LM Studio пингуются локально), вставка API-ключа для облачных. Минимум один провайдер обязателен.
3. **Заполнение User Profile** (опц., можно скипнуть) — короткая форма "кто ты, что любишь, что не любишь" → пишется в глобальный контекст (см. раздел Q)
4. **Демо-миссия** — заранее заготовленная "напиши hello-world на python в /tmp/kastol-demo". Юзер нажимает Run, видит как 2 агента работают, файл создаётся. Цель: показать ВСЁ за 30 секунд.
5. **Готов** — попадает на дашборд, дальше свободное плавание. Сверху подсказка "это твой первый раз → почитай Inline help (`Cmd+/`) или иди создавай свою миссию"

Onboarding можно повторить из настроек (`Help → Re-run onboarding`).

---

## 4. Архитектура

### Принцип №1: Один backend = ядро. Все клиенты — снаружи.

```
              ┌──────────────────────────────┐
              │        KASTOL Backend        │
              │  FastAPI + LangGraph + WS    │
              │  ┌────────────────────────┐  │
              │  │  Event Bus (pub/sub)   │  │
              │  └────────────────────────┘  │
              └──────────────────────────────┘
                    ▲   ▲   ▲   ▲   ▲   ▲
                    │   │   │   │   │   │
        ┌───────────┘   │   │   │   │   └──────────┐
        │               │   │   │   │              │
   ┌────────┐    ┌────────┐ │ ┌─────────┐  ┌──────────┐  ┌──────┐
   │  Web   │    │Tauri   │ │ │ TG bot  │  │ Overlay  │  │ CLI  │
   │ (Next) │    │desktop │ │ │ aiogram │  │ + voice  │  │      │
   └────────┘    └────────┘ │ └─────────┘  └──────────┘  └──────┘
                            │
                       ┌─────────┐
                       │ Public  │
                       │   API   │
                       └─────────┘
```

Ключ: **WebSocket-канал событий** — единый. Любой клиент подключается, видит миссию вживую, может слать команды.

### Принцип №2: Каждая фича = новый файл, не правка старого
Под вайб-кодинг:
- Tools — каждый в своём файле, регистрация через `@tool` декоратор
- Agents — JSON-файл характера + опц. кастомный класс
- Workflows — отдельный файл-граф
- Providers — через LiteLLM, ничего писать не надо
- MCP-серверы — JSON-конфиг + KASTOL сам их подцепит
- Plugins — папка с manifest, KASTOL сканирует
- Конфиги — JSON/YAML, без рекомпиляции
- Логи — с первого коммита, подробные

Когда захочешь "добавь агента-ресёрчера" — создаёшь один JSON-файл. Не 5 правок в 5 местах.

### Принцип №3: Persistent state — миссия переживает всё
Состояние миссии сохраняется каждые 5 секунд. Backend упал, ноут закрылся, питание выключили — открыл KASTOL заново → миссия там же где была, кнопка `Resume`.

### Принцип №4: Расширяемость как граждан 1-го класса
- **MCP** — не плагин, а нативная фича (см. раздел O)
- **Plugins** — стандартная спека (см. раздел P)
- **Public API + CLI** — встроены изначально (см. раздел R)

### Стек

| Слой | Технология | Почему |
|---|---|---|
| Backend | Python 3.11 + FastAPI + WebSockets | Лучшая экосистема для LLM |
| Оркестрация | LangGraph | Граф-ориентированная (агенты = узлы), контролируемее CrewAI/AutoGen |
| LLM-клиент | LiteLLM | Один интерфейс на ~100 провайдеров |
| MCP | официальный Python SDK (`mcp`) | Стандарт Anthropic, поддержан всеми |
| Frontend | Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui + Zustand | Минималистично, выглядит дорого |
| Хранилище | SQLite через SQLModel | Файлик, ноль настройки |
| RAG / KB | Chroma (embedded) | Локально, без серверов |
| Эмбеддинги | sentence-transformers (локально) или OpenAI (если есть ключ) | Local-first по умолчанию |
| Песочница | Docker (опц.) + dry-run + whitelist | Не сжигаем твой пк |
| TG-бот (фаза 3) | aiogram | Подцепится к тому же FastAPI |
| Голос (фаза 3) | Whisper (локально) | Никаких облаков |
| Десктоп (фаза 4) | Tauri | Один лёгкий экзешник, обернёт Next + Python sidecar |
| i18n | next-intl | EN дефолт, RU локаль с первого дня |

### Доменная модель

```
User Profile (глобальный, один на инсталляцию)
└── about_me, preferences, dont_likes, default_provider, ...

Workspace Memory (на каждую папку, обновляется агентами)
└── conventions, structure, dependencies, recent_changes

Mission (миссия = одна задача)
├── Workspace (ссылка)
├── Providers[] (ссылки)
├── MCP servers[] (подключённые)
├── Knowledge base (vectorstore миссии)
├── Agents[]
│   ├── role (architect, coder, critic, tester...)
│   ├── model
│   ├── system_prompt (личность)
│   ├── tools[] 
│   └── temperature
├── Plan (живой документ)
├── Workflow (sequential / debate / manager-worker)
├── Rules[] (global rules в живой памяти)
├── Run[] 
│   ├── Messages[] 
│   ├── ToolCalls[]
│   ├── Artifacts[]
│   ├── Cost
│   └── State snapshots
├── Notes[] (твои личные заметки)
└── Parent run (для форков)

Trigger (фаза 2+)
└── cron / file_watch / webhook → mission_template
```

### Структура backend

```
backend/app/
├── main.py
├── core/
│   ├── config.py
│   ├── db.py
│   ├── events.py
│   ├── logging.py
│   └── persistence.py     # snapshot / restore
├── models/                # все SQLModel схемы
├── api/
│   ├── missions.py
│   ├── plans.py
│   ├── agents.py
│   ├── providers.py
│   ├── presets.py
│   ├── notes.py
│   ├── profile.py
│   ├── triggers.py
│   ├── plugins.py
│   ├── public.py          # внешний API
│   └── ws.py
├── llm/
│   ├── client.py          # litellm wrapper
│   ├── budget.py          # cost guard
│   └── retry.py           # error recovery
├── agents/
│   ├── base.py
│   ├── runner.py          # tool-loop
│   ├── coach.py           # meta-agent (опц.)
│   └── personalities/     # JSON: 15-20 готовых
├── workflows/
│   ├── sequential.py
│   ├── debate.py
│   ├── manager_worker.py
│   └── comparison.py      # A/B mode
├── tools/
│   ├── registry.py
│   ├── files.py
│   ├── shell.py
│   ├── web.py
│   ├── git.py
│   ├── knowledge.py       # RAG search
│   └── vision.py          # phase 4
├── mcp/                    # MCP support
│   ├── manager.py         # подключения серверов
│   └── adapter.py         # MCP tools → tool registry
├── plugins/
│   ├── loader.py          # сканер плагинов
│   └── spec.py            # manifest schema
├── memory/
│   ├── context.py         # smart auto-compress
│   ├── rules.py           # global rules
│   ├── profile.py         # User Profile
│   ├── workspace.py       # Workspace Memory
│   └── kb.py              # knowledge base / RAG
├── planner/
│   └── service.py
├── safety/
│   ├── whitelist.py
│   ├── rate_limit.py
│   ├── confirm.py
│   ├── watchdog.py        # анти-зацикливания
│   └── validators.py      # tool ответы
├── triggers/
│   ├── scheduler.py       # cron
│   ├── file_watch.py
│   └── webhook.py
├── analytics/
│   └── cost.py            # долгосрочная аналитика
├── export/
│   └── backup.py          # export/import
└── integrations/
    ├── telegram.py
    └── overlay.py
```

### Структура frontend

```
frontend/
├── app/
│   ├── page.tsx                    # dashboard
│   ├── onboarding/page.tsx
│   ├── missions/[id]/page.tsx
│   ├── missions/[id]/tree/page.tsx # mission tree
│   ├── missions/new/page.tsx
│   ├── compare/page.tsx            # A/B mode
│   ├── agents/page.tsx
│   ├── providers/page.tsx
│   ├── presets/page.tsx
│   ├── mcp/page.tsx                # MCP servers
│   ├── plugins/page.tsx
│   ├── triggers/page.tsx
│   ├── analytics/page.tsx          # cost dashboard
│   ├── profile/page.tsx
│   ├── help/page.tsx
│   └── settings/page.tsx
├── components/
│   ├── mission/
│   │   ├── ChatStream.tsx
│   │   ├── ActivitySidebar.tsx
│   │   ├── ArtifactsPanel.tsx
│   │   ├── DiffViewer.tsx
│   │   ├── PlanReview.tsx
│   │   ├── RulesEditor.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── MissionTree.tsx
│   │   └── ComparisonView.tsx
│   ├── ui/                         # shadcn
│   ├── HotkeyPalette.tsx
│   ├── CostBadge.tsx
│   ├── HelpPopover.tsx
│   ├── ThemeProvider.tsx
│   └── I18nProvider.tsx
├── lib/
│   ├── api.ts
│   ├── ws.ts
│   ├── stores/                     # zustand
│   ├── i18n/                       # en, ru
│   └── types.ts                    # автогенерация из OpenAPI
└── public/sounds/
```

### Контракт событий (Event Bus)

```
mission.started / paused / resumed / finished / restored
plan.generated / updated / approved
agent.thinking / message / tool_call / tool_result
artifact.created / updated
cost.update
rule.added / removed / changed
confirmation.requested / resolved
mcp.connected / disconnected / error
plugin.installed / removed
trigger.fired
watchdog.alert
state.snapshot
```

---

## 5. MCP support (раздел O)

**MCP (Model Context Protocol)** — стандарт Anthropic для подключения tools/контекста к LLM. Используется Cursor, Claude Desktop, OpenAI и др.

### Как работает в KASTOL
- В UI: страница `MCP Servers` — список + кнопка `+ Add MCP server`
- Поддерживаются оба транспорта: **stdio** (локальный процесс) и **HTTP** (удалённый)
- Конфиг сервера — JSON (`name`, `command`, `args`, `env` или `url`)
- При подключении KASTOL опрашивает сервер через `mcp` SDK, получает список tools
- Все tools оттуда **автоматически попадают в registry** и могут быть назначены агентам как обычные
- Каждый MCP-server можно включать/отключать на уровне миссии

### Готовые серверы которые поддерживаем "из коробки" (предсконфигурированные шаблоны)
- Filesystem (официальный)
- GitHub
- Linear
- Slack
- Brave Search
- Postgres
- Память (memory MCP)

Юзер: `+ Add → выбрать из шаблонов → ввести креды → готово`.

---

## 6. Plugin spec (раздел P)

Плагин = папка с `manifest.json` + Python модуль. Расширяет KASTOL новыми tools, workflows, personalities, UI-панелями.

### Структура
```
~/.kastol/plugins/<plugin-name>/
├── manifest.json
├── __init__.py
├── tools/         (опц.)
├── workflows/     (опц.)
├── personalities/ (опц.)
└── ui/            (опц., React компонент через iframe)
```

### manifest.json
```json
{
  "name": "kastol-jira",
  "version": "0.1.0",
  "author": "...",
  "description": "...",
  "kastol_min_version": "0.4.0",
  "exports": {
    "tools": ["create_issue", "search_issues"],
    "workflows": [],
    "personalities": ["jira-pm"]
  },
  "config_schema": {
    "JIRA_URL": "string",
    "JIRA_TOKEN": "secret"
  }
}
```

### Установка
- Локально: положил папку в `~/.kastol/plugins/` → перезапустил → готово
- Из гитхаба: `kastol install user/repo` → KASTOL клонирует, проверяет manifest, активирует
- В UI: страница `Plugins` со списком, on/off, редактированием конфига

### Безопасность
- Плагины запускаются в том же процессе backend (доверие)
- В будущем (фаза 4+) — sandbox через Docker
- `kastol install` показывает diff manifest и спрашивает подтверждение перед установкой

---

## 7. User Profile + Workspace Memory (раздел Q)

### User Profile (глобальный)
Хранится в `~/.kastol/profile.yaml`. Заполняется при онбординге, редактируется руками или через UI.

```yaml
about_me: |
  Python backend разраб, иногда фронт на Next.js. 
  Работаю один.
preferences:
  - "Использую uv вместо pip"
  - "Пишу тесты на pytest"
  - "Английский в коде, комментарии на русском"
dont_likes:
  - "Tabs (только spaces)"
  - "Over-engineering"
default_provider: anthropic
default_team: code-squad
```

Профиль автоматически инжектится в system prompt каждого агента (как "User context").

### Workspace Memory (на каждую папку)
Хранится в `<workspace>/.kastol/memory.yaml`. Обновляется агентами по ходу миссий.

```yaml
project_type: "python game"
conventions:
  - "Использовать pygame 2.5+"
  - "Графика в /assets/sprites"
structure: |
  src/main.py      — entry
  src/player.py    — player class
  src/levels/      — JSON уровни
recent_decisions:
  - "Решили использовать Tiled для редактора уровней"
```

При первом запуске миссии в новой папке — KASTOL предлагает запустить **онбордер-агента**: он обходит проект и пишет начальный `memory.yaml`. Юзер ревьюит.

---

## 8. Knowledge base / RAG (раздел S)

### Что
Загрузил доку библиотеки / PDF / readme / скрин архитектуры → агенты могут к ней обращаться через `search_knowledge` tool.

### Как работает
- На уровне миссии — папка `<mission>/knowledge/` с эмбеддингами в Chroma (embedded)
- Источники:
  - Локальные файлы (drag-n-drop в UI)
  - URL (KASTOL скачает, парсит)
  - Папки (рекурсивно индексирует код-проект)
- Эмбеддинги — sentence-transformers локально (по дефолту), или OpenAI/Cohere если хочется качество
- Tool `search_knowledge(query)` → top-K кусков с цитатами

### Workspace KB (общая для папки)
Дополнительно — папка `<workspace>/.kastol/kb/` с эмбеддингами твоего кода. Любая миссия в этой папке имеет доступ. Переиндексация по cron или вручную.

---

## 9. External API + CLI (раздел R)

### Public API
REST на `/api/v1/` (отдельный неймспейс от внутреннего API). Документирован OpenAPI.

Эндпоинты:
- `POST /v1/missions` — создать миссию (с шаблоном или из сырого описания)
- `GET /v1/missions/{id}` — статус
- `POST /v1/missions/{id}/inject` — кинуть сообщение
- `POST /v1/missions/{id}/rule` — добавить global rule
- `GET /v1/missions/{id}/stream` — SSE-стрим событий
- `POST /v1/triggers/webhook/{trigger_id}` — внешний триггер

Аутентификация — token из настроек.

### CLI клиент
Отдельный пакет `kastol-cli` (или встроенный `python -m kastol`).

```bash
kastol run "напиши hello-world на python" --workspace ~/test
kastol missions
kastol mission show <id>
kastol install user/some-plugin
kastol export > backup.tar.gz
kastol import backup.tar.gz
kastol update
```

CLI — тонкий клиент к Public API. Если backend не запущен — поднимает его сам.

---

## 10. Triggers и Scheduling

### Виды
- **Cron** — "каждое утро в 9:00 запусти миссию-шаблон X с переменной date={today}"
- **File watch** — "когда в `~/inbox/` появится `.pdf` → запусти миссию-обработчик"
- **Webhook** — "когда придёт POST на `/v1/triggers/webhook/abc123` с payload — запусти миссию"

### Конфигурация
В UI — страница `Triggers`. Каждый триггер ссылается на **mission template** (см. раздел K — templates с параметрами).

Все триггеры выключаются одним рубильником в настройках (`Pause all triggers`).

---

## 11. Backup / Export-Import (раздел T)

### Что входит в бэкап
- Все миссии и runs
- Пресеты, агенты, personalities
- User Profile, Workspace Memory (опц., можно исключить)
- Конфиги MCP / plugins (без секретов по дефолту)
- Заметки

### Что НЕ входит
- API-ключи провайдеров (отдельным флагом, с шифрованием)
- Содержимое workspace (это твой код, его бэкапит git)

### Команды
```bash
kastol export --output backup.tar.gz
kastol export --include-secrets --password XXX
kastol import backup.tar.gz
```

В UI — кнопки `Export` / `Import` в настройках.

---

## 12. Error recovery & Persistent state

### Persistent state
- Каждые 5 секунд — снапшот состояния миссии в SQLite (messages, tool_calls, current step)
- При старте backend — сканит "не завершённые" миссии, пишет в UI "у тебя есть 2 миссии в паузе после рестарта, восстановить?"

### Retry & validators
- LLM API упал → exponential backoff, до 5 попыток, потом пауза миссии и запрос юзеру
- Tool-ответ невалидный (не подходит под схему) → агенту приходит сообщение "твой tool call неверный, исправь" + retry, до 3 раз → пауза
- Рейт-лимит → wait + retry

### Watchdog
- Если агент сделал 3 одинаковых tool_call подряд (одни и те же аргументы) → пауза + alert "агент Х зациклился, помоги"
- Если 10 шагов подряд без `write_*` действий → "вы много обсуждаете, не хотите ли начать делать?"
- Триггеры настраиваются в `safety/watchdog.py`

---

## 13. Roadmap по фазам (обновлённый)

### Phase 1 — Foundation MVP (расширенный)
**Цель:** ты пишешь "сделай платформер на pygame", смотришь как 3 агента спорят и пишут код, на выходе работающий проект. И ничего не падает.

| # | Фича |
|---|---|
| 1.1 | Скаффолд (repo, env, README, .kiro/steering, LICENSE-MIT, CONTRIBUTING.md) |
| 1.2 | Backend core (FastAPI, WS, SQLite, event bus, persistence) |
| 1.3 | Provider manager через LiteLLM |
| 1.4 | Tool registry + file tools |
| 1.5 | Agent runner + 3 характера с разными темпами и изолированными view |
| 1.6 | Sequential workflow на LangGraph |
| 1.7 | Planner шаг + Plan Review screen |
| 1.8 | ChatStream + ActivitySidebar + Artifacts panel |
| 1.9 | Diff viewer перед записью |
| 1.10 | Inject message + Global rules editor |
| 1.11 | Cost guard + Auto-snapshot в git |
| 1.12 | Resume / Continue this + Persistent state каждые 5s |
| 1.13 | Hotkey palette (Cmd+K) базовый |
| 1.14 | Темы (dark/light) |
| 1.15 | **Onboarding flow** (welcome → provider → profile → demo mission) |
| 1.16 | **User Profile** (глобальный контекст) |
| 1.17 | **Error recovery** (retry, validators, watchdog) |
| 1.18 | **Public API + базовый CLI** (`kastol run`, `kastol missions`) |
| 1.19 | **i18n каркас** (EN дефолт, RU добавится позже) |
| 1.20 | **Inline help** (`?` подсказки + `Cmd+/`) |
| 1.21 | **Personality library** (10+ готовых персонажей в `personalities/`) |

### Phase 2 — Глубина команды + расширяемость
- 2.1 Debate workflow + Devil's Advocate
- 2.2 Manager-worker workflow
- 2.3 Comparison / A/B mode
- 2.4 Reply to message, Pin, Edit & branch
- 2.5 Шелл tool с подтверждением + safety whitelist
- 2.6 Web search/fetch tools
- 2.7 Smart context auto-compress
- 2.8 Templates задач с параметрами (jinja-style)
- 2.9 Notes слой
- 2.10 "Explain this decision" кнопка
- 2.11 Ghost mode + Rate limiter + Rule diff alert
- 2.12 Preset marketplace через гист-шаринг
- 2.13 Estimated time
- 2.14 **MCP support** (подключение серверов, шаблоны для популярных)
- 2.15 **Workspace Memory** (`.kastol/memory.yaml` + онбордер-агент)
- 2.16 **Knowledge base / RAG** (Chroma + search_knowledge tool)
- 2.17 **Plugin spec + loader** (без UI пока, JSON + папка)
- 2.18 **Mission tree visualization**
- 2.19 **Cost dashboard** (долгосрочная аналитика)
- 2.20 **Backup / Export-Import**
- 2.21 **Triggers** (cron + file watch + webhook)
- 2.22 **Coach / Meta-agent** (опц.)
- 2.23 RU локаль

### Phase 3 — Удалёнка и быстрый доступ
- 3.1 TG-бот (запуск миссий, прогресс, подтверждения)
- 3.2 Quick-call оверлей + хоткей
- 3.3 Voice input (Whisper локально) + Voice notes → global rules
- 3.4 TTS ответы
- 3.5 System tray иконка с мини-чатом
- 3.6 Звуки уведомлений
- 3.7 Discord-бот (опц.)
- 3.8 Plugins UI (страница установки/настройки)
- 3.9 Self-update (`kastol update`)

### Phase 4 — Десктоп и зрение
- 4.1 Tauri обёртка → один экзешник
- 4.2 Vision tools (screenshot, vision модели через LiteLLM)
- 4.3 Computer Use агент (мышь/клава, kill-switch, scoped окна)
- 4.4 Docker sandbox для шелла + плагинов
- 4.5 Параллельные миссии — UI вкладок
- 4.6 Agent benchmark mode
- 4.7 Mood meter
- 4.8 Session sharing (read-only public link)

---

## 14. План работ — Phase 1 разбит на PR

| PR | Что входит | Когда работает |
|---|---|---|
| **PR1: Skeleton** | репо, .env.example, docker-compose, .kiro/steering, README, LICENSE-MIT, CONTRIBUTING, скрипты запуска | `make dev` поднимает backend + frontend пустыми |
| **PR2: Backend core** | FastAPI, SQLite, models, event bus, persistence, provider manager, /api/health | Можно вбить ключ, увидеть провайдера в БД |
| **PR3: Agents + tools** | tool registry, file tools, agent runner с tool-loop, 10+ характеров в JSON, validators | CLI: запустить одного агента с задачей "запиши hello.txt" |
| **PR4: Workflow + Planner + Recovery** | Sequential workflow, planner service, REST для миссий, watchdog, retry | API: POST /missions → план → run → файл создан, миссия переживает рестарт |
| **PR5: Public API + CLI** | `/api/v1/`, `kastol-cli` базовый | Можно дёргать KASTOL снаружи |
| **PR6: Frontend skeleton + i18n** | Next.js + shadcn, страницы провайдеров/агентов/миссий, WS-клиент, i18n каркас | UI с пустыми экранами и работающей навигацией |
| **PR7: Mission live UI** | ChatStream, ActivitySidebar, ArtifactsPanel, Plan Review, DiffViewer | Полный сценарий: создал миссию в UI → смотришь как агенты работают |
| **PR8: Control & safety UI** | Inject, Global rules editor, Cost guard, Auto-snapshot git, Resume UI | Можешь вмешиваться, не страшно жать Run |
| **PR9: Onboarding + Profile + Help** | Welcome flow, демо-миссия, User Profile editor, Inline help (`?`, `Cmd+/`) | Первый запуск — не пустой экран, дружелюбно |
| **PR10: Polish** | Hotkey palette, темы, мелкая шлифовка | Phase 1 готов, можно дог-фудить |

После каждого PR — пишу "готов PR такой-то", даёшь добро или поправки.

---

## 15. Правила проекта (steering)

В `.kiro/steering/project.md` сразу будут:
- **Стиль кода**: Python — ruff + black; TS — prettier + eslint
- **Архитектурные правила**: новый tool = новый файл с `@tool` декоратором; новый агент = JSON-файл в `personalities/`; новый workflow = отдельный файл-граф; новый MCP-сервер = JSON в конфиге; новый плагин = папка в `plugins/`
- Не делаем тестов без явной просьбы (пользователь — вайб-кодер, тесты — оверкилл на старте)
- Каждая фича = отдельная ветка + PR
- Имена коммитов: `feat:`, `fix:`, `chore:`, `docs:` (Conventional Commits)
- Local-first, никакой телеметрии
- Логи с первого коммита, подробные
- Всё конфигурируется через JSON/YAML, без рекомпиляции
- Любая внешняя зависимость — обоснованная (минимум третьих сторон)

### Open source гигиена
- `LICENSE` — **MIT**
- `README.md` — описание + быстрый старт + скриншот
- `CONTRIBUTING.md` — как помочь
- `CODE_OF_CONDUCT.md` — стандартный Contributor Covenant
- `.github/ISSUE_TEMPLATE/` — bug, feature, question
- `.github/PULL_REQUEST_TEMPLATE.md` — чеклист

---

## 16. Что НЕ изобретаем (не путаемся с чужим)

| Сравнение | Что у них | Почему мы не они |
|---|---|---|
| CrewAI / AutoGen / LangGraph | Фреймворки оркестрации | Используем как библиотеку (LangGraph), не пытаемся переписать |
| Aider / Cursor / Cline | AI пишет код в редакторе | Мы не редактор, мы дирижёр команд агентов |
| Open Interpreter / Computer Use | AI рулит компом | Берём в фазу 4 как один из режимов |
| ChatGPT Desktop / Claude Desktop | Один чат | У нас команды агентов спорят между собой |
| MCP экосистема | Серверы tools | Используем как клиент, не делаем своих |

Наша **уникальная ниша** — UX-обёртка с минималистичным визардом, прозрачностью спора агентов, удалёнкой через мессенджер, Quick-call оверлеем и MCP-первоклассной поддержкой.

---

## 17. Дизайн и ощущения

### 17.1 Философия

**"Тихая элегантность".** Чёрный текст на белом, белый на чёрном. Никакого цветового шума. Минимум элементов на экране — но каждый элемент сделан на высшем уровне (типографика, отступы, тайминги). Эстетика **iA Writer × Things 3 × Linear settings × Rauno Freiberg**. Продукт должен выглядеть так, будто его сделал один очень упрямый человек с хорошим вкусом.

Главные правила:
- Цвет — это **смысл**, не украшение. Если цвет не несёт информации — его нет.
- Граница важнее тени. Тонкая линия 1px вместо drop-shadow.
- Воздух важнее контента. Лучше меньше элементов с большими отступами.
- Анимация — это **ответ системы пользователю**, не аттракцион.
- Шрифт делает 80% впечатления. Разные веса (regular / medium / semibold) — наша основная "палитра".

### 17.2 Палитра

**Light theme (основная):**
```
background       #FFFFFF
surface          #FAFAFA
border           #E5E5E5
border-subtle    #F0F0F0
text-primary     #0A0A0A
text-secondary   #525252
text-muted       #A3A3A3
accent           #0A0A0A
danger           #DC2626
```

**Dark theme:**
```
background       #0A0A0A
surface          #141414
border           #262626
border-subtle    #1F1F1F
text-primary     #FAFAFA
text-secondary   #A3A3A3
text-muted       #525252
accent           #FAFAFA
danger           #EF4444
```

**Палитра агентов** — единственное цветовое исключение, только в полоске 2px слева от сообщения и точке перед именем:
```
Architect   #4F46E5   indigo
Coder       #16A34A   green
Critic      #DC2626   red
Tester      #CA8A04   amber
Researcher  #7C3AED   violet
Planner     #525252   neutral
Manager     #0891B2   cyan
Devil's     #EA580C   orange
You         #0A0A0A   pure
System      #A3A3A3   muted
```

### 17.3 Типографика
- Один шрифт на UI — **Inter**
- Моноширинный — **JetBrains Mono** (только для кода в диффах и tool call аргументов)
- Шкала: 11/13/14/16/20/28px
- Веса: 400 / 500 / 600. Никаких bold/black, никакого italic
- Tracking `-0.01em` на крупных размерах

### 17.4 Spacing
- Шкала кратна 4px: 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64
- Внутри карточек — 24px базовый, 16px компактный
- Между секциями — 48px
- Между однотипными элементами — 8px
- Радиусы: 0 / 4 / 6 / 8
- Границы только 1px

### 17.5 Иконки
**Lucide**, размеры 14/16/20px, stroke-width `1.5`. Цвет `text-secondary` по дефолту.

### 17.6 Layout
- **Topbar** 44px (только: брейдкрамб, стоимость, pause/resume, меню)
- **Sidebar** 220px (collapse в 56px по `Cmd+\`)
- **Chat** — без рамок, max-width 720px, центрирован
- **Artifacts panel** 320px справа (collapse по `Cmd+]`)
- **Activity bar** 32px снизу
- Все панели resizable, состояние в localStorage

### 17.7 Сообщения в чате
- Без пузырей, без аватаров-картинок
- Полоска 2px слева — единственный цвет агента
- Точка перед именем того же цвета
- Имя semibold, чёрное; мета (время, модель, токены) `xs muted`
- Tool calls — компактные строки моно-шрифтом, раскрываются кликом
- Стриминг — текст печатается по токенам, мигает курсор `▍`
- Hover — тулбар иконок справа (Reply / Pin / Explain / Edit & Branch / Copy)

### 17.8 Анимации
- **Тайминги:** instant 80ms, fast 150ms, normal 220ms, slow 400ms
- **Easing один:** `cubic-bezier(0.32, 0.72, 0, 1)` (без овершута)
- Появление сообщения — имя мгновенно, текст печатается, полоска растёт за 150ms
- Tool call — раскрытие height 0→auto за 200ms
- Activity bar — cross-fade 150ms (не флешит)
- Diff Viewer — слайдит снизу до 60% за 220ms
- Plan Review — карточки стаггером по 30ms
- Cmd+K palette — затемнение фона 40% + scale 0.96→1 за 150ms
- **Никакого:** bounce, spring, scale на hover кнопок, confetti, parallax, spinners

### 17.9 Кнопки
Три типа:
1. **Primary** — фон `text-primary`, текст инверсный, без рамки
2. **Secondary** — transparent, рамка 1px
3. **Ghost** — без фона и рамки

Размеры: 28 / 32 / 36px. Радиус 6px. Без градиентов и теней.

### 17.10 Inputs
32-36px, рамка 1px (на focus — `text-primary`), радиус 4px, лейбл сверху `sm medium`.

### 17.11 Звуки
По дефолту выкл. Опц.: tick (5% volume), done, nudge.

### 17.12 Микро-детали
- Mono-цифры для всех счётчиков
- Стоимость обновляется раз в 200ms
- Хоткеи в углах кнопок (`⌘K`, `Esc`, `⏎` мелким `xs muted`)
- Sticky-scroll в чате с кнопкой `↓ Jump to live`
- Каретка `▍` мигает в стриме
- Tooltips — задержка 600ms, без стрелок
- Таймстемпы относительные (`2m ago`), на hover — абсолютные
- Pause — баннер сверху чата, не сереет
- Devil's Advocate — оранжевый лейбл с `[devil's advocate]` в мете
- Confirmation toast — снизу справа, 220ms

### 17.13 Стек дизайна
| Что | Чем |
|---|---|
| Базовые компоненты | shadcn/ui на Radix |
| Анимации | Framer Motion + CSS transitions |
| Иконки | Lucide (stroke 1.5) |
| Шрифты | Inter + JetBrains Mono через next/font |
| Темы | CSS variables + next-themes |
| Tailwind | расширенный config со всеми токенами |

### 17.14 Резюме
Ты открываешь KASTOL, и оно **тихое**. Ничего не прыгает. Шрифт чёткий. Отступы дышат. Цвет появляется только когда что-то значит. Анимация мягко объясняет что произошло. Агенты пишут. Ты читаешь. Жмёшь Esc — всё закрылось мгновенно. Это не "приложение", это **инструмент мастера**.

---

## 18. Финальные вопросы перед стартом кода

1. **Имя — `KASTOL` финал?**
2. **Какой провайдер у тебя реально подключен?** (OpenAI / Anthropic / OpenRouter / Ollama / нет пока) — нужно для дефолтов в `.env.example` и онбординга
3. **OK план PR-ов** (10 PR-ов в Phase 1)?

Когда даёшь добро — стартует **PR1: Skeleton**.

---

## 19. Дополнительный артефакт

Рядом с этим файлом в репо лежит [`AI_SESSION_PROMPT.md`](./AI_SESSION_PROMPT.md) — готовый промт для запуска сессии в любой ИИ-системе (Claude / ChatGPT / Cursor / любой другой). Содержит контекст проекта, текущий статус, правила работы. Скопировал-вставил → ИИ сразу в курсе.
