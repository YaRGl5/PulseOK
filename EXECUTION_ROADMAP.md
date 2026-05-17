# PulseOK · Детальный план выполнения

**Версия документа:** 1.0
**Дата:** 2026-05-17
**Статус:** Утверждён к исполнению

---

## Оглавление

- [0. Контекст и реестр решений](#0-контекст-и-реестр-решений)
- [Stage 0 · Документация (12 правок)](#stage-0--документация-12-правок)
- [Stage 1 · Connectivity (10 шагов)](#stage-1--connectivity-10-шагов)
- [Stage 2 · Engines](#stage-2--engines)
  - [Phase 2.1 · PlayerokAPI fork (полный)](#phase-21--playerokapi-fork-полный)
  - [Phase 3.1 · EventBus (полный)](#phase-31--eventbus-полный)
  - [Phase 3.2 · ModuleLoader](#phase-32--moduleloader)
  - [Phase 4.1 · delivery/ engine](#phase-41--delivery-engine)
  - [Phase 4.2 · Recovery Poller (active mode)](#phase-42--recovery-poller-active-mode)
  - [Phase 4.3 · relist/ engine ⭐](#phase-43--relist-engine-)
  - [Phase 4.4 · relist/ config + UI bindings](#phase-44--relist-config--ui-bindings)
  - [Phase 4.5 · bump/ engine](#phase-45--bump-engine)
  - [Phase 4.6 · stats/ engine ⭐](#phase-46--stats-engine-)
- [Stage 3 · Telegram UI](#stage-3--telegram-ui)
  - [Phase 5.1 · tgbot/bot.py + middleware](#phase-51--tgbotbotpy--middleware)
  - [Phase 5.2 · Setup Wizard](#phase-52--setup-wizard)
  - [Phase 5.3 · Главное меню](#phase-53--главное-меню)
  - [Phase 5.4 · Delivery UI](#phase-54--delivery-ui)
  - [Phase 5.5 · Relist UI](#phase-55--relist-ui)
  - [Phase 5.6 · Bump UI](#phase-56--bump-ui)
  - [Phase 5.7 · Stats UI](#phase-57--stats-ui)
  - [Phase 5.8 · Прочее (chats, deals, fast_replies, system, notifications)](#phase-58--прочее-chats-deals-fast_replies-system-notifications)
- [Stage 4 · Plugin SDK + миграция](#stage-4--plugin-sdk--миграция)
  - [Phase 6 · PulseContext SDK](#phase-6--pulsecontext-sdk)
  - [Phase 7.1 · Перенос steam_rental_pro](#phase-71--перенос-steam_rental_pro)
  - [Phase 7.2 · Перенос auto_smm](#phase-72--перенос-auto_smm)
  - [Phase 7.3 · Перенос kosell_rental](#phase-73--перенос-kosell_rental)
  - [Phase 7.4 · Перенос auto_steam_points](#phase-74--перенос-auto_steam_points)
  - [Phase 7.5 · Перенос auto_delivery](#phase-75--перенос-auto_delivery)
  - [Phase 7.6 · 48ч продакшн-soak с плагинами](#phase-76--48ч-продакшн-soak-с-плагинами)
- [Stage 5 · Релиз v1.0.0](#stage-5--релиз-v100)
- [Приложение A · Реестр событий](#приложение-a--реестр-событий)
- [Приложение B · Полная схема config.json](#приложение-b--полная-схема-configjson)
- [Приложение C · Acceptance criteria релиза v1.0.0](#приложение-c--acceptance-criteria-релиза-v100)

---

## 0. Контекст и реестр решений

### 0.1 Что строим
Глубокий форк `playerok-universal-13.9` для автоматизации Playerok. Цель — стабильное, изолированное, расширяемое ядро с продуманной архитектурой, готовое для платных модулей.

### 0.2 Окружение
| Параметр | Значение |
|---|---|
| ОС | Ubuntu (VPS) + Windows 11 (dev) |
| Python | 3.11 |
| Аккаунт Playerok | боевой `yargl` |
| Telegram | новый отдельный бот |
| Часовой пояс | `Europe/Moscow` (UTC+3) |
| Параллельная работа | PulseOK как **observer-only** на Stage 1 (Universal продолжает работать) |

### 0.3 Реестр решений (Q1–Q12)
| # | Решение |
|---|---|
| Q1 | Stage 1 — observer-only, Universal параллельно |
| Q2 | `silent_mode: false` по умолчанию (тумблер в настройках); `silent_mode=true` блокирует ВСЕ исходящие сообщения и публикации |
| Q3 | Cookie expiry → алерт в TG; за 3 дня до истечения предупреждение (если возможно распарсить) |
| Q4 | Auto-confirm не делаем — Playerok сам подтверждает через 2/7/30 дней по категории; ловим через `itemDealUpdated` |
| Q5 | ROLLED_BACK → алерт админу, ключ НЕ возвращаем в склад |
| Q6 | Auto-bump перенос из `tools/autobump/` AS-IS на Phase 4.5 |
| Q7 | Relist на `itemDealUpdated.status=CONFIRMED` (и `ROLLED_BACK` тоже) |
| Q8 | Cooldown между релистами одного товара — **15 сек** |
| Q9 | API rate limit — **2 req/sec** |
| Q10 | Импорт настроек из Universal через стандартный JSON export/import |
| Q11 | **Все плагины переписываются под PulseContext SDK** (новые мажорные версии) |
| Q12 | Все 4 WS подписки активны: `chatMessageCreated`, `itemDealCreated`, `itemDealUpdated`, `userUpdated`. **Тумблер в настройках** «Уведомлять о смене статуса сделки» — по умолчанию ON. |

### 0.4 Ссылки на ключевые документы
- [`MASTER_PLAN.md`](../MASTER_PLAN.md) — полная архитектура и спецификации
- [`STAGE_1_CONNECTIVITY.md`](../STAGE_1_CONNECTIVITY.md) — детальный Stage 1
- [`PLUGIN_SDK_ADVANTAGES.md`](../PLUGIN_SDK_ADVANTAGES.md) — PulseContext SDK
- [`pulseok_ui_and_flow.md`](../pulseok_ui_and_flow.md) — UI-спецификация

---

## Stage 0 · Документация (12 правок)

**Цель:** синхронизировать `MASTER_PLAN.md` и `pulseok_ui_and_flow.md` с финальными решениями перед стартом кода.

### Правки

| # | Файл | Что добавить |
|---|---|---|
| D1 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.2.1 | Расширенная SQL-схема `sales` (+ `promotion_cost`, `listing_cost`, `bump_cost`, `transfer_fee`, `transfer_fee_pct`, новая формула `net_profit`) |
| D2 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.2.2 | **NEW** Локальная таблица тарифов (`promotion_tariffs.json`) — для прогноза при создании лота |
| D3 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.3 | `import_from_playerok()` + резолв promotion: запрос `get_transactions(item_id=X, type=[ITEM_DEFAULT_PRIORITY, ITEM_PREMIUM_PRIORITY])` |
| D4 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.4 | Лин-меню статистики (4 KPI + 5 графиков + drill-down) |
| D5 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.6 | **NEW** Раздел «Графики»: matplotlib + тёмный пресет + 5 типов графиков |
| D6 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §9.6 | **NEW** Поддержка картинок: уведомления чатов + шаблоны быстрых ответов с фото |
| D7 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §12.2 | Добавить `"timezone": "Europe/Moscow"` в схему `config.json` |
| D8 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §9.2 | Setup Wizard: добавить шаг 5 «Часовой пояс» |
| D9 | [`pulseok_ui_and_flow.md`](../pulseok_ui_and_flow.md) §📊 | Перерисовать экран «Статистика» под лин-версию |
| D10 | [`pulseok_ui_and_flow.md`](../pulseok_ui_and_flow.md) §Setup Wizard | Добавить шаг выбора TZ |
| D11 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §8.6.7 | **NEW** Тренды (`scipy.stats.linregress`) + прогнозы (extrapolation + 95% CI) |
| D12 | [`MASTER_PLAN.md`](../MASTER_PLAN.md) §15.4 | **NEW** Реестр решений Q1–Q12 (silent_mode, ROLLED_BACK, observer Stage 1, plugins на PulseContext, rate limit, cooldowns, и т.д.) |

### Acceptance Stage 0
- [ ] Все 12 правок применены
- [ ] `MASTER_PLAN.md` версия в шапке поднята до v9
- [ ] Внутренние ссылки между документами не сломаны
- [ ] `pulseok_ui_and_flow.md` сохраняет полный набор экранов

---

## Stage 1 · Connectivity (10 шагов)

**Цель:** минимально жизнеспособный бот, держащий WebSocket к Playerok 48+ часов без падений. **Никакой логики выдачи/relist/bump.** Только подключение, наблюдение, наблюдательный recovery, базовый TG.

### 1.1 · Скелет проекта
**Файлы:**
```
pulseok/
├── bot.py                    # точка входа: argparse, asyncio.run
├── requirements.txt          # aiogram==3.x, aiohttp, websockets, pyyaml, pytz
├── PATCHNOTES.md             # v0.0.1 заголовок
├── .gitignore                # data/, logs/, *.pyc, __pycache__
├── install.sh                # apt + pip install
├── install.bat               # для Windows-разработки
├── run.sh                    # screen-обёртка с автоперезапуском
├── README.md                 # минимум: что это, как запустить
└── core/
    └── __init__.py
```
**Acceptance:**
- [ ] `python bot.py --help` показывает usage
- [ ] `pip install -r requirements.txt` проходит чисто
- [ ] `bot.py` падает с понятной ошибкой, если нет `data/config.json`

---

### 1.2 · Logging setup
**Файл:** `pulseok/core/logging_setup.py`

**Что делает:**
- `RotatingFileHandler` 10МБ × 5 файлов → `data/logs/pulseok.log`
- Console handler с цветами (`colorama`)
- TZ-aware форматтер: время в МСК, не UTC
- Формат: `[2026-05-17 10:43:21 МСК] [INFO] [pulseok.listener] WS connected`

**Acceptance:**
- [ ] Логи пишутся в файл и в консоль
- [ ] Время в логах в МСК
- [ ] Ротация работает (тест: 10 МБ → новый файл)
- [ ] Есть метод `get_recent_lines(n=50)` для команды `/logs`

---

### 1.3 · Circuit Breaker
**Файл:** `pulseok/core/circuit_breaker.py`

**Что делает:** реализует паттерн Circuit Breaker (CLOSED → OPEN → HALF_OPEN → CLOSED).

**Параметры (из конфига):**
- `failure_threshold: 5` — сколько подряд ошибок до OPEN
- `recovery_timeout: 60` — секунд в OPEN до перехода в HALF_OPEN
- `half_open_max_calls: 3` — сколько успешных вызовов нужно в HALF_OPEN для CLOSED

**API:**
```python
@cb.protect("playerok_api")
async def make_request(...): ...
```

**Acceptance:**
- [ ] Unit-тест в `tests/test_circuit_breaker.py` покрывает все 3 состояния
- [ ] При OPEN запросы падают с `CircuitBreakerOpenError` без вызова функции
- [ ] Метрики (count_failures, count_successes, current_state) доступны для `/status`

---

### 1.4 · Config Manager
**Файл:** `pulseok/core/config_manager.py`

**Что делает:**
- Читает `data/config.json` с валидацией схемы
- Hot-reload через `watchdog` при изменении файла
- Атомарная запись через `tempfile + os.replace`
- Точечный API: `cfg.get("tg.admin_ids")`, `cfg.set("tg.silent_mode", True)`
- Подписка на изменения: `cfg.on_change(key, callback)`

**Acceptance:**
- [ ] Hot-reload работает (тест: `echo > config.json` → перезагружается без рестарта)
- [ ] Атомарная запись: при `kill -9` во время `set()` файл не повреждён
- [ ] Если `config.json` отсутствует — создаётся из дефолтов
- [ ] Schema-валидация: лишние ключи → warning в лог, отсутствующие обязательные → ошибка

---

### 1.5 · PlayerokAPI fork (минимум для Stage 1)
**Папка:** `pulseok/playerokapi/` (8 файлов из `playerok-universal-13.9/playerokapi/`)

**Что меняем:**
- `account.py` — оборачиваем `_make_request()` в Circuit Breaker, добавляем rate limit 2 req/sec через `asyncio.Semaphore`
- Остальные файлы AS-IS

**Что добавляем для Stage 1:**
- Метод `account.get_profile()` — для проверки cookie на старте (уже есть в Universal)

**Что НЕ добавляем на Stage 1** (отложено в Phase 2.1):
- `get_item()`, `get_all_my_items()` — нужны для relist (Phase 4.3)
- `get_transactions(item_id=...)` — нужно для stats (Phase 4.6)
- `send_message(attachments=[...])` — нужно для шаблонов с фото

**Acceptance:**
- [ ] `from pulseok.playerokapi import Account` работает
- [ ] `account.get_profile()` возвращает данные при валидном cookie
- [ ] Невалидный cookie → понятная ошибка
- [ ] При 5 подряд ошибках API → CB открывается, в логах видно

---

### 1.6 · Listener с exponential backoff
**Файл:** `pulseok/playerokapi/listener/listener.py` (патч поверх форка)

**Что меняем:**
- WS reconnect — exponential backoff: 1с → 2 → 4 → 8 → 16 → 32 → 60 (cap)
- `seen` set регистрируется ДО dispatch (предотвращает повторную обработку при гонке)
- Геттеры: `is_connected: bool`, `uptime_sec: float`, `reconnect_count: int`, `last_event_at: datetime`

**WS подписки (все 4):**
```python
SUBSCRIPTIONS = [
    "chatMessageCreated",
    "itemDealCreated",
    "itemDealUpdated",
    "userUpdated",
]
```

**Acceptance:**
- [ ] Дёрни сетевой кабель → бот переподключается с backoff, не падает
- [ ] При 10 переподключениях за 1 минуту → CB открывается
- [ ] `listener.is_connected` корректно отражает реальность
- [ ] Все 4 подписки активны и приходят на handler

---

### 1.7 · EventBus (минимум для Stage 1)
**Файл:** `pulseok/core/event_bus.py`

**Что делает:**
- Async dispatch: `await event_bus.emit("NEW_MESSAGE", payload)`
- Sync compat: `event_bus.emit_sync("NEW_DEAL", payload)` для legacy
- Изоляция ошибок: ошибка в одном handler не валит остальных
- Backward-compat сигнатур (R4):
  - 1-arg: `def handler(payload)` — для Universal-плагинов
  - 2-arg: `async def handler(payload, ctx: PulseContext)` — для PulseOK
- Tail buffer: `deque(100)` последних событий для команды `/events_tail`

**Список событий Stage 1:**
- `WS_CONNECTED`, `WS_DISCONNECTED`, `WS_RECONNECT_STARTED`
- `NEW_MESSAGE` (из `chatMessageCreated`)
- `NEW_DEAL` (из `itemDealCreated`)
- `DEAL_STATUS_CHANGED` (из `itemDealUpdated`)
- `USER_UPDATED` (из `userUpdated`)
- `RECOVERY_DEAL_FOUND` (из poller)

**Acceptance:**
- [ ] Сабскрайберы регистрируются и получают события
- [ ] Ошибка в одном sub не валит остальных (изоляция через `try/except` per-subscriber)
- [ ] `/events_tail 20` показывает последние 20 событий
- [ ] Backward-compat: handler с 1 аргументом работает

---

### 1.8 · Recovery Poller (наблюдательный режим Stage 1)
**Файл:** `pulseok/core/recovery_poller.py`

**Что делает:**
- Каждые 60 секунд `account.get_deals(statuses=[PAID, PENDING])`
- Сравнивает с локальным `seen_deals: set`
- Если найдена новая сделка которой нет в seen → emit `RECOVERY_DEAL_FOUND`
- На Stage 1 этот event только логирует «нашёл пропущенную сделку», не выдаёт товар

**API:**
```python
poller.seed_seen(deal_ids: list[str])  # инициализация на старте
poller.mark_deal_seen(deal_id)         # вызывается из WS handler
poller.check_now() -> list[Deal]       # ручной вызов для /recovery_check
```

**Acceptance:**
- [ ] Эмитит `RECOVERY_DEAL_FOUND` если WS пропустил сделку
- [ ] НЕ дублирует событие если оно уже было обработано через WS (контракт `mark_deal_seen` ДО emit из listener)
- [ ] В TG приходит уведомление «🛟 Recovery: найдена пропущенная сделка #ABC123»

---

### 1.9 · Telegram (минимум)
**Файлы:**
```
pulseok/tgbot/
├── bot.py                    # инициализация aiogram3 + dispatcher
├── middleware.py             # AdminOnly + CtxInjector
├── setup_wizard.py           # FSM 5 шагов
└── routers/
    ├── system.py             # /start /status /uptime /events_tail /restart /setup /logs
    └── notifications.py      # подписка на NEW_MESSAGE, NEW_DEAL, DEAL_STATUS_CHANGED
```

**Setup Wizard FSM (5 шагов):**
1. Cookie Playerok
2. User-Agent (с дефолтом)
3. TG admin IDs
4. Часовой пояс (Europe/Moscow по умолчанию)
5. Подтверждение → `account.get_profile()` → запись config → restart

**TG-команды Stage 1:**
- `/start` — приветствие + меню (минимум: только статус и настройки)
- `/status` — снапшот: WS up/down, uptime, reconnects, last event time, CB state
- `/uptime` — короткий ответ «Up: 47h 12m»
- `/events_tail [N=20]` — последние N событий
- `/restart` — `os.execv` для рестарта
- `/setup` — запуск Setup Wizard заново
- `/logs` — последние 50 строк лога

**Acceptance:**
- [ ] Setup Wizard проходится с нуля
- [ ] AdminOnly middleware блокирует не-админов
- [ ] При новом сообщении в Playerok → уведомление в TG (с поддержкой картинок!)
- [ ] При новой сделке → уведомление в TG
- [ ] При смене статуса сделки → уведомление в TG **(если включено в настройках, по умолчанию ON)**
- [ ] `/status` показывает актуальное состояние

---

### 1.10 · Heartbeat и финальная сборка
**Файл:** `pulseok/core/heartbeat.py`

**Что делает:**
- Каждые 5 минут проверяет `listener.last_event_at`
- Если > 10 минут без событий → ALERT в TG
- Каждый час пишет в лог: `Heartbeat: WS connected, uptime 12h, 156 events processed`

**Финальная сборка `bot.py`:**
```python
async def main():
    config = ConfigManager(...)
    logging = setup_logging(config)
    cb = CircuitBreaker(config.cb_config)
    account = Account(cookie=config.cookie, cb=cb)
    event_bus = EventBus()
    listener = Listener(account, event_bus)
    poller = RecoveryPoller(account, event_bus)
    heartbeat = Heartbeat(listener, event_bus)
    tg = TelegramBot(config.tg_token, event_bus, account)

    await asyncio.gather(
        listener.run(),
        poller.run(),
        heartbeat.run(),
        tg.run(),
    )
```

**Acceptance:**
- [ ] Бот запускается одной командой `python bot.py`
- [ ] Все 4 корутины работают параллельно
- [ ] Graceful shutdown по SIGTERM/Ctrl+C
- [ ] Если какая-то корутина упала → лог + restart процесса (через `run.sh`)

---

### 1.SOAK · 48-часовой soak-тест на VPS

**Подготовка:**
1. VPS Ubuntu 22.04, Python 3.11
2. `git clone` или `scp` папки `pulseok/`
3. `bash install.sh`
4. Заполнить `data/config.json` (или пройти Setup Wizard через TG)
5. `screen -dmS pulseok bash run.sh`

**Что мониторим:**
- WS uptime (должен быть > 99% за 48ч)
- Количество reconnects (< 50 за 48ч)
- Memory usage (стабильное, без утечек)
- CPU usage (< 5% в среднем)
- 0 unhandled exceptions в логах

**Ежедневный чек:**
- `/status` снапшот в TG в 12:00 МСК
- Просмотр свежих логов

**Definition of Done Stage 1:**
- [ ] 48 часов аптайм без ручных вмешательств
- [ ] WS reconnects прошли успешно (видно в логах)
- [ ] Получено N сообщений / M сделок (записать факт)
- [ ] CB ни разу не открывался по сетевым ошибкам
- [ ] PATCHNOTES.md обновлён до v0.1.0
- [ ] Все 10 шагов acceptance criteria выполнены

---

## Stage 2 · Engines

**Цель:** ядро функционала (delivery, relist, bump, stats) на устойчивом фундаменте Stage 1.

### Phase 2.1 · PlayerokAPI fork (полный)

**Файл:** `pulseok/playerokapi/account.py`

**Что добавляем:**

| Метод | Источник | Зачем |
|---|---|---|
| `get_item(item_id) -> Item \| None` | новый | для relist — получить актуальный item перед публикацией |
| `get_all_my_items(status=None) -> Iterator[Item]` | расширение | курсорный обход всех своих лотов с пагинацией |
| `get_transactions(item_id=X, type=[...])` | расширение фильтра | для stats — резолв promotion-стоимости |
| `send_message(chat_id, text, attachments=[url])` | расширение | для шаблонов с фото |
| `update_deal(deal_id, status, complaint_id=...)` | если нет | для возможной ручной обработки споров |

**Acceptance:**
- [ ] Все новые методы покрыты тестами в `tests/test_account.py`
- [ ] `get_all_my_items()` корректно делает пагинацию (>100 лотов)
- [ ] `send_message` с attachments работает (проверить на тестовом чате)

---

### Phase 3.1 · EventBus (полный)

**Файл:** `pulseok/core/event_bus.py` (расширение Stage 1)

**Что добавляем поверх Stage 1:**
- Полный реестр событий (см. Приложение A)
- Метод `event_bus.emit_with_response()` для синхронных запросов (например, плагин просит у delivery «выдай ключ за меня»)
- Приоритизация подписчиков (`priority: int = 100`, ниже = раньше вызывается)
- Метрики: количество событий каждого типа, время обработки, ошибки

**Acceptance:**
- [ ] Все 18 событий из реестра работают
- [ ] `emit_with_response` возвращает результат от первого подписчика, который вернул не-None
- [ ] `/events_stats` показывает таблицу: тип события / счётчик / средн. время / ошибок

---

### Phase 3.2 · ModuleLoader

**Файл:** `pulseok/core/module_loader.py`

**Что делает:**
- Сканирует `modules/*` директории
- Читает `meta.py` каждого плагина
- Импортирует `__init__.py`, регистрирует event handlers, TG routers, BOT_EVENT_HANDLERS
- API:
  - `loader.list_loaded() -> list[ModuleInfo]`
  - `loader.disable_module(slug)` — без удаления, выключает регистрации
  - `loader.enable_module(slug)`
  - `loader.reload_module(slug)` — для hot-reload во время разработки

**Acceptance:**
- [ ] Загрузка существующих 5 плагинов проходит без ошибок (даже до их миграции под PulseContext — должна работать R4 backward compat)
- [ ] `disable_module()` корректно убирает плагин из event_bus
- [ ] При ошибке в `__init__.py` плагина — лог + плагин помечен как FAILED, ядро живёт

---

### Phase 4.1 · delivery/ engine

**Папка:** `pulseok/delivery/`

```
delivery/
├── __init__.py
├── engine.py          # DeliveryEngine — главный класс
├── storage.py         # JSON storage по cells (ячейки = группы товаров)
├── matcher.py         # match deal.item.name → cell (с _normalize)
├── queue.py           # Queue из MASTER_PLAN (5 стадий с фолбэками)
├── meta.py            # VERSION = "1.0.0"
└── PATCHNOTES.md
```

**Что переносим из `auto_delivery/`:**
- Логика storage AS-IS
- Логика matching, но через `_normalize()` (нормализация эмодзи)
- Очередь — из MASTER_PLAN (паттерн «Очереди выдачи» с 5 стадиями)

**Что меняем:**
- Хендлеры event_bus вместо собственных подписок
- `silent_mode` — если ON, выдача не отправляется (только в очередь и алерт админу)

**Acceptance:**
- [ ] Сделка → match по cell → выдача из stock
- [ ] Если нет stock → алерт + cell.status=`empty`
- [ ] Очередь работает (5 стадий: processing → 10 мин ретраев → алерт → 30 мин редких ретраев → ручной перехват)
- [ ] Telegram-меню «⏳ Очередь / Ошибки выдачи» работает (см. MASTER_PLAN §4.5)

---

### Phase 4.2 · Recovery Poller (active mode)

**Файл:** `pulseok/delivery/background.py`

**Что меняем поверх Stage 1:**
- На событии `RECOVERY_DEAL_FOUND` — теперь не только логирует, но и реально вызывает `delivery.process_deal()`
- Защита от дублей: проверяем что сделка не в `delivery.queue` и не в `delivery.history`

**Acceptance:**
- [ ] Имитируем пропуск WS (отключаем listener) → poller через 60 сек ловит сделку → выдача проходит
- [ ] Двойная обработка не происходит (сделка обработана через WS → poller её скипает)

---

### Phase 4.3 · relist/ engine ⭐ ГЛАВНЫЙ ФИКС

**Папка:** `pulseok/relist/`

```
relist/
├── __init__.py
├── engine.py          # RelistEngine — переписан с нуля
├── matcher.py         # _normalize-based матчинг
├── state.py           # block_keyword, unblock_keyword, cooldown tracking
├── meta.py            # VERSION = "1.0.0"
└── PATCHNOTES.md
```

**4 фикса по сравнению со сломанным `tools/relist/engine.py`:**

| # | Проблема старого | Фикс PulseOK |
|---|---|---|
| F1 | Матчинг по `item.id` (id меняется при каждом publish) | Матчинг по нормализованному имени через `_normalize()` |
| F2 | Нет refresh перед publish | `account.get_item(id)` → проверяем актуальный статус → publish |
| F3 | Только 12 товаров без пагинации | `account.get_all_my_items()` курсорный обход |
| F4 | APPROVED статус попадает в кандидаты | Фильтр: только `SOLD/EXPIRED` |

**Триггеры relist:**
- `DEAL_STATUS_CHANGED` со статусом `CONFIRMED` → relist связанного товара
- `DEAL_STATUS_CHANGED` со статусом `ROLLED_BACK` → relist (товар у нас, можно перевыставить)
- Ручной триггер через TG

**Cooldown между релистами одного товара:** 15 секунд

**API:**
```python
relist.block_keyword(keyword: str)      # запретить relist для товаров с этим словом
relist.unblock_keyword(keyword: str)
relist.relist_now(item_id: str)         # ручной relist
relist.get_state() -> dict              # для UI
```

**Acceptance:**
- [ ] Продали лот «GTA 5 ⭐» → через 15 сек он опубликован заново
- [ ] Если в склад был добавлен новый ключ для блокированного keyword — relist не происходит
- [ ] APPROVED товары не релистятся
- [ ] Все 4 бага из старого `tools/relist` не воспроизводятся

---

### Phase 4.4 · relist/ config + UI bindings

**Файл:** `pulseok/relist/config.py`

**Что хранит:**
```json
{
  "auto_relist": true,
  "cooldown_sec": 15,
  "blocked_keywords": ["тест", "пример"],
  "bindings": [
    { "item_pattern": "GTA 5 ⭐", "stock_cell": "gta5_keys" }
  ]
}
```

**Acceptance:**
- [ ] Конфиг hot-reload работает
- [ ] Блокировки сохраняются между перезапусками

---

### Phase 4.5 · bump/ engine

**Папка:** `pulseok/bump/`

**Что переносим:** `tools/autobump/` AS-IS (только адаптировать под PulseContext и новые названия модулей)

**Что меняем:**
- Использует `account.get_all_my_items()` вместо локальной пагинации
- Конфиг через `ConfigManager`

**Acceptance:**
- [ ] Каждые 4 часа все лоты в `included_folder` поднимаются (`increase_item_priority_status`)
- [ ] `excluded_folder` исключается
- [ ] Лог автоподнятий доступен

---

### Phase 4.6 · stats/ engine ⭐

**Папка:** `pulseok/stats/`

```
stats/
├── __init__.py
├── engine.py          # StatsEngine: live recording, queries
├── importer.py        # import_from_playerok с резолвом promotion
├── tariffs.py         # promotion_tariffs.json (для прогноза)
├── charts.py          # matplotlib рендер 5 графиков
├── trends.py          # linregress + forecast с 95% CI
├── queries.py         # SQL-агрегации
├── meta.py            # VERSION = "1.0.0"
├── data/
│   ├── sales.sqlite           # SQLite база
│   └── promotion_tariffs.json # таблица тарифов
└── PATCHNOTES.md
```

**SQL-схема `sales`:**
```sql
CREATE TABLE sales (
    deal_id          TEXT PRIMARY KEY,
    item_id          TEXT,
    item_name        TEXT,
    category_id      TEXT,
    category_name    TEXT,
    buyer_id         TEXT,
    buyer_username   TEXT,
    gross_revenue    REAL NOT NULL,
    fee_multiplier   REAL NOT NULL,
    commission       REAL NOT NULL,
    listing_cost     REAL DEFAULT 0,
    bump_cost        REAL DEFAULT 0,
    promotion_cost   REAL DEFAULT 0,
    transfer_fee_pct REAL DEFAULT 0.06,
    transfer_fee     REAL DEFAULT 0,
    cost             REAL DEFAULT 0,
    net_profit       REAL,
    status           TEXT,
    source           TEXT,
    created_at       TEXT NOT NULL,
    updated_at       TEXT
);
CREATE INDEX idx_sales_created ON sales(created_at);
CREATE INDEX idx_sales_category ON sales(category_id);
```

**Live-запись:**
- На `DEAL_STATUS_CHANGED` со статусом `CONFIRMED` → INSERT/UPDATE row
- Резолв promotion-стоимости через `get_transactions(item_id=X, type=[ITEM_DEFAULT_PRIORITY, ITEM_PREMIUM_PRIORITY])`

**Import from Playerok:**
- Курсорный обход `get_deals(statuses=[CONFIRMED, ROLLED_BACK])`
- Для каждой сделки: `get_transactions(item_id=..., type=[promotion types])`
- Идемпотентность через `INSERT OR REPLACE`

**Графики (matplotlib + тёмный пресет):**

| # | Название | Тип | Тренд | Прогноз |
|---|---|---|---|---|
| 1 | 📈 Чистая прибыль | line | ✅ | ✅ |
| 2 | 📊 Кол-во продаж | bar | ✅ | ✅ |
| 3 | 💰 Декомпозиция | stacked bar | ✅ | ✅ |
| 4 | 🏆 Топ-10 товаров | h-bar | — | — |
| 5 | 🥧 Структура комиссии | donut | — | — |

**Период-свитчер:** `7д / 30д / 90д / Всё время` (4 кнопки под каждым графиком)

**Тренды:** `scipy.stats.linregress` поверх данных, пунктирная жёлтая линия + текст «↗ +12.4% (рост 3.2 ₽/день)»

**Прогнозы:** экстраполяция на 3/7/14/30 дней (зависит от периода) + закрашенная зона ±1.96σ (95% CI)

**Acceptance:**
- [ ] Live-запись работает
- [ ] Импорт за 5000 сделок занимает < 5 минут
- [ ] Идемпотентность импорта
- [ ] Все 5 графиков рендерятся и шлются в TG как PNG
- [ ] Тренд + прогноз отображаются на графиках 1-3
- [ ] CSV-экспорт включает все колонки
- [ ] Период-свитчер работает через `edit_message_media`

---

## Stage 3 · Telegram UI

**Цель:** полный Telegram-интерфейс по [`pulseok_ui_and_flow.md`](../pulseok_ui_and_flow.md).

### Phase 5.1 · tgbot/bot.py + middleware

**Что делает:**
- aiogram3 Bot + Dispatcher с MemoryStorage
- AdminMiddleware (фильтр не-админов)
- CtxMiddleware (инжектит `ctx: PulseContext` во все handlers)

**Acceptance:** все handlers получают `ctx`, не-админы получают «Доступ запрещён».

---

### Phase 5.2 · Setup Wizard

**FSM 5 шагов** (Stage 1 уже сделал минимум; здесь финализируем):
1. Cookie
2. User-Agent
3. TG admin IDs
4. Часовой пояс
5. Подтверждение → restart

**Acceptance:** wizard проходится с нуля, ошибки показываются понятно.

---

### Phase 5.3 · Главное меню

**Жёстко по UI-спеке:**
```
📦 Автовыдача
♻️ Перевыставление
⬆️ Автоподнятие
📊 Статистика
🔌 Модули

[ярлыки до 4 плагинов 2x2]

🤖 Консультант  💳 Вывод
💬 Чаты         🛍 Товары
📋 Сделки       ⭐ Отзывы
💸 Транзакции   📝 Логи
⚙️ Настройки    🔄 Рестарт
```

---

### Phase 5.4 · Delivery UI

**Экраны** (см. UI-спеку `📦 Автовыдача`):
- Список ячеек, добавление/удаление
- Просмотр stock
- Очередь / Ошибки выдачи (5 стадий)
- **БЕЗ карточек товаров с превью** (по решению пользователя)

---

### Phase 5.5 · Relist UI

**Экраны:**
- Включён/выключен
- Cooldown
- Заблокированные ключевые слова
- Лог релистов (200 последних)
- Кнопка «🧪 Beta: API test relisting»

---

### Phase 5.6 · Bump UI

**Экраны:**
- Включён/выключен
- Интервал (default 4ч)
- Included / Excluded folders
- Лог автоподнятий

---

### Phase 5.7 · Stats UI

**Главный экран:**
```
┌──────────────────────────────────────┐
│ 📊 Статистика · [7д ▼] · МСК         │
├──────────────────────────────────────┤
│ 💵 Чистая прибыль   4 121 ₽          │
│ 📦 Сделок           23               │
│ 🧾 Комиссий          1 086 ₽         │
│ 📈 Средний чек      236 ₽            │
├──────────────────────────────────────┤
│ 🏆 Топ-3 товаров                     │
│ 1. GTA 5      · 12 шт · 1 440 ₽     │
│ 2. CS2        · 6 шт  · 900 ₽       │
│ 3. Red Dead   · 5 шт  · 750 ₽       │
├──────────────────────────────────────┤
│ [📈 Прибыль] [📊 Кол-во]             │
│ [💰 Декомпозиция] [🏆 Топ-10]        │
│ [🥧 Комиссии] [🔍 Подробно]          │
│ [📤 CSV]    [📥 Импорт]              │
│        ◀️ Назад                       │
└──────────────────────────────────────┘
```

**[🔍 Подробно]** — drill-down с разбивкой комиссий/promo/transfer fee/cost/net.

**[📥 Импорт]** — запускает `stats.import_from_playerok()` с прогрессом в TG.

---

### Phase 5.8 · Прочее

**fast_replies.py:**
- CRUD шаблонов: текст ИЛИ картинка+текст (новинка)
- Хранится в `config.json` под `fast_replies: list[Reply]`
- При нажатии «📋 Шаблоны» в уведомлении NEW_MESSAGE → показывается список → отправка через `account.send_message(attachments=[url])`

**chats.py / deals.py:**
- Форк из Universal с обновлённым дизайном
- В чатах: рендер картинок (если в сообщении есть `attachments`)

**system.py:**
- `/restart`, `/logs`, `/uptime`, `/events_tail`

**notifications.py:**
- `NEW_MESSAGE` → уведомление с поддержкой картинок (`send_photo`)
- `NEW_DEAL` → уведомление
- `DEAL_STATUS_CHANGED` → уведомление **(если включено в настройках)**
- `ON_STOCK_EMPTY` → алерт
- `ON_RELIST_DONE` → уведомление

**Тумблер «Уведомлять о смене статуса сделки»:**
- В `⚙️ Настройки → Уведомления → Статус сделок`
- По умолчанию: ON
- Хранится в `config.notifications.deal_status_changed: bool`

---

## Stage 4 · Plugin SDK + миграция

**Цель:** `PulseContext` SDK + перевод всех 5 плагинов на новый API + 48ч продакшн.

### Phase 6 · PulseContext SDK

**Файл:** `pulseok/core/pulse_context.py`

```python
@dataclass
class PulseContext:
    delivery:   DeliveryEngine
    relist:     RelistEngine
    bump:       BumpEngine
    stats:      StatsEngine
    config:     ConfigManager
    event_bus:  EventBus
    account:    PlayerokAccount
    tg_bot:     Bot | None
    modules:    ModuleLoader
    storage:    PluginStorage   # каждый плагин получает свою партицию
    timezone:   pytz.tzinfo

    async def notify(self, text: str, admin_only: bool = True) -> None
    async def send_message(self, chat_id: str, text: str, attachments: list[str] = None) -> None
    def logger(self, name: str) -> logging.Logger
    def disable_module(self, slug: str) -> None
    def enable_module(self, slug: str) -> None
```

**PluginStorage:**
- Каждый плагин получает свою папку в `data/modules/<slug>/`
- API: `storage.save("rentals.json", data)`, `storage.load("rentals.json")`
- Атомарная запись через `tempfile + os.replace`

**Acceptance:**
- [ ] Все engines доступны через `ctx.*`
- [ ] PluginStorage не позволяет плагину писать вне своей папки
- [ ] Backward-compat: 1-arg handlers Universal-плагинов работают

---

### Phase 7.1 · Перенос steam_rental_pro → v1.0.0

**Что меняется:**
- Папка `pulseok/modules/steam_rental_pro/`
- Все вызовы `bot.send_message()` → `ctx.send_message()`
- Хранилище паролей/аккаунтов → `ctx.storage.save("accounts.json", ...)`
- Логи стат продаж → `ctx.stats.record_event(...)`
- TG-callback префикс `srp:` (как сейчас)
- `meta.py` — `VERSION = "1.0.0"`, `SDK = "pulseok"`

**Acceptance:**
- [ ] Все функции v0.5.x работают на новом ядре
- [ ] Аренда Steam-аккаунтов проходит полный цикл
- [ ] Возврат пароля работает
- [ ] Стат-учёт через `ctx.stats` агрегируется на главном экране

---

### Phase 7.2 · Перенос auto_smm → v2.0.0

**Что меняется:**
- Замена прямых вызовов на `ctx.*`
- Карточки активных заказов используют `ctx.notify`
- Очередь использует `ctx.event_bus.emit(...)`
- TG-префикс `smm:`

---

### Phase 7.3 · Перенос kosell_rental → v2.0.0

**Что меняется:**
- Привязки лотов хранятся через `ctx.storage`
- TG-префикс `kr:`

---

### Phase 7.4 · Перенос auto_steam_points → v2.0.0

**Что меняется:**
- API-клиент Dessly Hub использует общий `aiohttp.ClientSession` из `ctx`
- Очередь интегрирована с `ctx.delivery.queue`
- TG-префикс `asp:`

---

### Phase 7.5 · Перенос auto_delivery → v2.0.0

**Важно:** `auto_delivery` функционал теперь — часть ядра (`pulseok/delivery/`). Существующий плагин:
- (a) **Удалить** — функционал в ядре, плагин больше не нужен
- (b) **Переделать в "плагин-импортёр"** — оставить только `import_legacy_history.py` как тулзу для импорта старых backup'ов
- (c) **Переделать в плагин расширений** — добавляет custom-выдачу для специфичных категорий

Рекомендую (b) — минимум кода, максимум совместимости.

---

### Phase 7.6 · 48ч продакшн-soak с плагинами

**Что включаем:**
- Все 5 переписанных плагинов (или 4, если auto_delivery удалён)
- На боевом yargl
- `silent_mode: false` — бот реально отвечает покупателям

**Что мониторим:**
- 0 unhandled exceptions
- 0 потерянных выдач
- Стабильное memory usage
- Корректная stat-агрегация

**Definition of Done Stage 4:**
- [ ] 48 часов аптайма с реальной нагрузкой
- [ ] Все плагины работают без сбоев
- [ ] Конверсия выдачи 100% (нет зависших в Очереди)

---

## Stage 5 · Релиз v1.0.0

### 5.1 · Документация

**Файлы:**
- `README.md` — что это, как установить, как настроить
- `docs/INSTALLATION.md` — детальная инструкция для VPS Ubuntu/Debian
- `docs/MIGRATION_FROM_UNIVERSAL.md` — как переехать с Universal
- `docs/PLUGIN_DEVELOPMENT.md` — туториал «как написать плагин под PulseOK SDK»
- `docs/TROUBLESHOOTING.md` — типовые проблемы

### 5.2 · Docker (опционально)

**Файлы:**
- `Dockerfile` — Python 3.11-slim base
- `docker-compose.yml` — pulseok + volume для `data/`

### 5.3 · Архив для распространения

**Структура архива `pulseok-v1.0.0.zip`:**
```
pulseok/                  # код
docs/                     # документация
modules/                  # 5 переписанных платных плагинов (если включены в дистрибутив)
README.md
LICENSE
install.sh
install.bat
run.sh
requirements.txt
```

### Acceptance Stage 5
- [ ] README с инструкцией по запуску
- [ ] Docker-образ собирается и запускается
- [ ] Архив распакован на чистом VPS → Setup Wizard → бот работает
- [ ] PATCHNOTES.md финализирован до v1.0.0

---

## Приложение A · Реестр событий

| # | Событие | Триггер | Payload | Подписчики |
|---|---|---|---|---|
| 1 | `WS_CONNECTED` | listener подключился | `{ts}` | heartbeat, notifications |
| 2 | `WS_DISCONNECTED` | listener отвалился | `{ts, reason}` | heartbeat, notifications |
| 3 | `WS_RECONNECT_STARTED` | начался reconnect | `{ts, attempt}` | heartbeat |
| 4 | `NEW_MESSAGE` | `chatMessageCreated` | `{message, chat, user}` | notifications, plugins |
| 5 | `NEW_DEAL` | `itemDealCreated` | `{deal, item, buyer}` | delivery, notifications, stats, plugins |
| 6 | `DEAL_STATUS_CHANGED` | `itemDealUpdated` | `{deal, prev_status, new_status}` | relist, stats, notifications, plugins |
| 7 | `USER_UPDATED` | `userUpdated` | `{unread_count}` | notifications |
| 8 | `RECOVERY_DEAL_FOUND` | poller нашёл пропуск | `{deal}` | delivery |
| 9 | `ITEM_PAID` | sub-event NEW_DEAL/PAID | `{deal, item}` | delivery, stats |
| 10 | `ON_STOCK_EMPTY` | delivery нашёл пустой cell | `{cell, item_name}` | notifications, relist (block) |
| 11 | `ON_STOCK_REFILLED` | админ долил stock | `{cell, count}` | relist (unblock + trigger) |
| 12 | `ON_RELIST_DONE` | relist опубликовал | `{old_id, new_id, item_name}` | stats, notifications |
| 13 | `ON_RELIST_FAILED` | relist упал | `{item_name, error}` | notifications |
| 14 | `ON_BUMP_DONE` | bump поднял лот | `{item_id, item_name}` | stats |
| 15 | `ON_DELIVERY_DONE` | выдача успешна | `{deal, item, code}` | stats, notifications |
| 16 | `ON_DELIVERY_FAILED` | выдача в Очередь с ошибкой | `{deal, item, error}` | notifications |
| 17 | `ON_DEAL_ROLLED_BACK` | sub-event DEAL_STATUS_CHANGED | `{deal}` | notifications (alert) |
| 18 | `ON_RENTAL_FINISHED` | (из плагинов) аренда закончилась | `{rental_id, item_id}` | relist (auto-relist) |

---

## Приложение B · Полная схема config.json

```json
{
  "version": "1.0.0",
  "timezone": "Europe/Moscow",

  "playerok": {
    "cookie": "...",
    "user_agent": "Mozilla/5.0 ...",
    "rate_limit_rps": 2,
    "circuit_breaker": {
      "failure_threshold": 5,
      "recovery_timeout": 60,
      "half_open_max_calls": 3
    }
  },

  "tg": {
    "token": "...",
    "admin_ids": [123456789],
    "silent_mode": false
  },

  "notifications": {
    "new_message": true,
    "new_deal": true,
    "deal_status_changed": true,
    "stock_empty": true,
    "relist_done": false,
    "rolled_back_alert": true
  },

  "delivery": {
    "queue_retry_intervals_sec": [120, 120, 120, 120, 120],
    "queue_late_retry_interval_sec": 300,
    "queue_late_retry_count": 6
  },

  "relist": {
    "auto_enabled": true,
    "cooldown_sec": 15,
    "blocked_keywords": []
  },

  "bump": {
    "enabled": true,
    "interval_sec": 14400,
    "included_folder": "default",
    "excluded_keywords": []
  },

  "stats": {
    "transfer_fee_pct": 0.06,
    "chart_engine": "matplotlib",
    "chart_dark_theme": true
  },

  "fast_replies": [
    { "type": "text", "body": "Здравствуйте!" },
    { "type": "photo", "url": "https://...", "caption": "Инструкция" }
  ]
}
```

---

## Приложение C · Acceptance criteria релиза v1.0.0

### Стабильность
- [ ] 7-дневный аптайм на боевом yargl без ручных вмешательств
- [ ] WS reconnects работают плавно (exponential backoff видно в логах)
- [ ] Circuit Breaker корректно открывается и закрывается
- [ ] Memory usage стабильный (нет утечек за 7 дней)
- [ ] CPU usage < 5% в среднем

### Функционал
- [ ] Setup Wizard проходится с нуля за < 5 минут
- [ ] Auto-delivery: конверсия 100% (нет зависших в очереди дольше 40 минут)
- [ ] Auto-relist: 4 фикса работают, лоты переоткрываются за 15 секунд
- [ ] Auto-bump: лоты поднимаются каждые 4ч
- [ ] Stats: live-запись + import + 5 графиков с трендом и прогнозом

### Плагины
- [ ] Все 5 (или 4) плагинов работают на новом SDK
- [ ] Plugin storage изолирован
- [ ] Disable/enable модулей работает

### UI
- [ ] Все экраны UI-спеки реализованы
- [ ] Картинки в чатах рендерятся
- [ ] Шаблоны быстрых ответов поддерживают фото
- [ ] Период-свитчер графиков работает

### Документация
- [ ] README + INSTALLATION + MIGRATION + PLUGIN_DEV + TROUBLESHOOTING написаны
- [ ] PATCHNOTES.md ведётся для каждого этапа
- [ ] Все meta.py имеют корректные версии

---

**Конец документа.**
