# HANDOFF — Hockey Coach CRM

> Документ для переноса контекста в другой чат. Здесь: что это за проект, карта приложения
> и его логики, что сделано в этой сессии и что осталось. Глубокая спецификация архитектуры —
> в `BLUEPRINT.md` (этот файл — практический «где мы сейчас»).
>
> Дата: 2026-06-14. Репозиторий: **BadMause/MyCode**. Рабочая ветка: **`claude/busy-lamport-7lnt8o`**.

---

## 0. TL;DR

- **Что это:** один автономный офлайн-файл `crm_hockey.html` (~2240 строк, vanilla JS, без сборки,
  Chart.js встроен). CRM для хоккейного тренера: игроки, скаутинг, составы, матчи, статистика,
  соперники, физика/психология/медицина, предсезонка, календарь, аналитика, заметки, настройки.
- **Хранилище:** `localStorage`, ключ **`crm_hockey_v1`** (namespace `CRM`).
- **Как запустить:** открыть `crm_hockey.html` двойным кликом в браузере. Сервер не нужен.
- **В этой сессии сделано (3 коммита):**
  1. Слой расширяемости + живая «карта» в приложении (блок 16 Map) — `bc522ae`.
  2. Фикс краша вкладок на нестроковых датах + английское меню/амплуа/статусы — `5251d9b`.
  3. Перевод хоккейных терминов/сокращений на английский — `c924d75`.
- **Принципы (не нарушать):** один владелец на сущность; одна точка создания; двусторонние связи
  не рвём; даты — строки `YYYY-MM-DD`; все строки через `esc()`; новый блок — один вызов
  `CRM.registerModule(...)`.

---

## 1. Файлы и git

| Файл | Что это |
|---|---|
| `crm_hockey.html` | Само приложение (единственный рабочий файл). |
| `BLUEPRINT.md` | Полная спецификация архитектуры (15 блоков, ownership, инварианты, §11 — расширяемость). |
| `handoff.md` | Этот файл. |
| `.gitignore` | Игнорит `.shots/` (локальные скриншоты). |

**Коммиты (новые → старые):**
```
c924d75  Translate hockey terms & abbreviations to English
5251d9b  Fix tab crash on non-string dates; English menu, positions, statuses
bc522ae  Add safe block-extension layer + live in-app architecture map
34725c0  Add empty-state guards for physical/psycho/medical add forms   (до сессии)
3155554  Add .gitignore for local screenshot artifacts                  (до сессии)
73b435f  Inline Chart.js 4.4.1 for full offline use                      (до сессии)
51c1956  Build offline Hockey Coach CRM (crm_hockey.html)                (до сессии)
```
Всё запушено в `origin/claude/busy-lamport-7lnt8o`. PR **не** создавался.

---

## 2. КАРТА ПРИЛОЖЕНИЯ (логика)

### 2.1 Инфраструктура (namespace `CRM`)
Всё висит на `window.CRM`. Ключевые объекты (номера строк — ориентир, могут смещаться, ищи grep’ом):

| Объект | Строка | Назначение |
|---|---|---|
| `const CRM` | 186 | `{ modules:{}, LSKEY:"crm_hockey_v1" }` |
| `CRM.TABS` | 189 | Массив `[id,num,label,desc]` — порядок и подписи сайдбара. |
| `CRM.ICONS` | ~205 | SVG-иконки по id блока. |
| Хелперы | 229–276 | `uuid, esc, parseLocalDate, localYmd, normYmd, fmtDate, fmtDateShort, num, clamp, avg, round1`. |
| Константы | ~277 | `POS_LABEL, POS_FULL, STATUS_LABEL, STATUS_CLASS` (амплуа/статусы — на английском). |
| `playerById/scoutById/playerName/...` | ~280 | Доступ к игрокам, рендер имени/грейда. |
| `sectionTitle/chartOpts/makeChart` | ~300 | Заголовок раздела, опции и фабрика графиков Chart.js. |
| `CRM.store` | 282 | Единый стор: `load, get(path), set(path,val), save, exportJson, importJson, resetAll, defaults, defaultNorms`. Внутри `migrate()` (add-only миграции + нормализация дат). |
| `CRM.toast` | ~360 | Тосты. |
| `CRM.modal` | ~366 | `open(html), close(), frame(title,body)`. `<script>` внутри modal НЕ исполняется → события только через `onclick`. |
| `CRM.modalNav` | ~385 | Стек «назад» для модалок (`wrap, back, reset`, максимум 12). |
| `CRM.router` | ~400 | `parse(), go(name,params), render()`. Маршрут — из `location.hash`. |
| `CRM.boot` | 439 | Точка входа (на `DOMContentLoaded`): `store.load()` → `registerCustomBlocks()` → строит сайдбар → `render()`. |

### 2.2 Слой расширяемости (добавлен в этой сессии) — секция «EXTENSIBILITY LAYER», стр. 1872+
| Объект | Строка | Назначение |
|---|---|---|
| `CRM.ui` | 1886 | Диалоги вместо нативных: `confirm({title,message,danger,okLabel,onOk})`, `formModal({title,fields,onSubmit,submitLabel,note})`, `ok/cancel/submit`. Свой оверлей `#dialog` поверх `#modal`. Типы полей: `text/number/date/textarea/select/checkbox`. |
| `CRM.registry` | 1938 | Единый источник истины для карты: `id → {id,num,label,desc,owns[],creates[],reads[],writes[],readonly,kind}`. `kind ∈ core|meta|custom`. |
| `CRM.CORE_META` | 1942 | Метаданные 15 канонических блоков (ownership/reads/writes) — сидируются в registry при загрузке. |
| `CRM.ownerOf(entity)` | ~1960 | Возвращает id блока-владельца сущности (по базовому ключу). |
| `CRM.registerModule(def)` | 1966 | **Единственный способ добавить блок.** Проверяет id/формат/уникальность; **гарантирует «один владелец на сущность»**; регистрирует в `modules+TABS+ICONS+registry`. `def.replace:true` — обновить. |
| `CRM.unregisterModule(id)` | ~1995 | Удаляет блок (системные `core/meta` удалять нельзя). |
| `CRM.customBlocks` | 2005 | Пользовательские блоки: `PAL` (иконки), `iconSvg, iconKeys, makeModule(def), register(def,replace)`. Генерик-рендер списка/текста + CRUD. |
| `CRM.registerCustomBlocks()` | 2076 | Вызывается из `boot` после `store.load`; перерегистрирует сохранённые кастом-блоки (идемпотентно, чистит «осиротевшие»). |
| Блок **16 Map** | 2083 | `CRM.registerModule({id:'map',...})` — рендерит архитектуру ЖИВЬЁМ из `CRM.registry`: список блоков, владение, граф зависимостей, инварианты, how-to и менеджер кастом-блоков (кнопка «+ Новый блок»). |

### 2.3 Блоки (16 + пользовательские)
Строки — начало `CRM.modules.X={...}`.

| # | id | Строка | Владеет (store) | Создаёт |
|---|---|---|---|---|
| 01 | dashboard | 452 | — (read-only) | — |
| 02 | scouting | 516 | `scoutDB, scoutTarget, scoutPriorities` | скаут-кандидата |
| 03 | profiles | 634 | `scout[]` | игрока команды |
| 04 | roster | 1057 | `roster.*` | состав/звено/историю |
| 05 | matches | 751 | `matches[]` (+ пишет `opponents[].lineups`) | матч |
| 06 | stats | 1201 | — | — |
| 07 | opponents | 1250 | `opponents[], playoffBracket` | соперника, его игру |
| 08 | physical | 1440 | `physical[]` | физтест |
| 09 | psycho | 1482 | `psycho[]` | психотест |
| 10 | preseason | 1564 | `preseason{}` | план/детали/усталость |
| 11 | calendar | 1699 | `calendar[]` | событие |
| 12 | medical | 1513 | `medical[]` (+ пишет `scout[].status`) | травму |
| 13 | analytics | 1740 | — | — |
| 14 | notes | 1767 | `notes[]` | заметку |
| 15 | settings | 1798 | `settings{}` | — (нормативы, экспорт/импорт JSON) |
| 16 | map | 2083 | — (meta) | пользовательские блоки |
| 17.. | `cb_*` | runtime | `custom.<id>` | записи списка / текст |

### 2.4 Владение и потоки данных (стрелка = «пишет в»)
```
opponents ──► matches[]              (createGamePair → matches._create*)
matches   ──► opponents[].lineups    (syncOppLineup*)
matches  ◄──► roster.history.matchId / matches.lineupId   (двусторонняя — чистить обе стороны)
medical   ──► scout[].status
scouting  ──► scout[]                (promoteToTeam* → profiles._create*)
profiles  ──► scout[]                (владелец)
ЧИТАТЕЛИ (никуда не пишут): dashboard, stats, analytics, calendar
custom-блок ──► custom.<id>          (изолирован, не трогает чужие сущности)
```
`*` — методы, помеченные в BLUEPRINT как канон, но в коде ещё НЕ внедрены (см. §4 ниже).

### 2.5 Схема localStorage (`store.defaults()`)
```js
{
  scout:[], scoutDB:[],
  roster:{ lines:{}, special:{}, history:[], showSpecial:false },
  matches:[], opponents:[],
  playoffBracket:{r8:[],r4:[],r2:[],f:[]},
  physical:[], psycho:[], practice:[], calendar:[], medical:[],
  preseason:{ plan:[], details:[], fatigue:[], seasonPlan:'', seasonDetails:'',
              cal:{start:'',end:'',seasonEnd:''} },
  notes:[],
  customBlocks:[],          // [{id,num,label,desc,icon,kind:'list'|'text',fields:[{key,label,type}],createdAt}]
  custom:{},                // { <blockId>: [...rows] }  или  { <blockId>: {text:''} }
  settings:{ coach, team, league, season, norms:[{id,name,lower,gold,silver,bronze}] },
  scoutTarget:{}, scoutPriorities:{},
  state:{ seedOpp10:false }
}
```

### 2.6 Инварианты (проверять перед сдачей)
1. `roster.history[].matchId` ⇄ `matches[].lineupId` согласованы; удаление чистит обе стороны.
2. `matches[]` пополняется только через `matches._create` (канон BLUEPRINT).
3. `scout[].status` пишет только `medical`.
4. `scoutDB`(кандидаты) ↔ `scout`(команда) — мост только `promoteToTeam`.
5. Статистика читается только из `result.finished===true`.
6. Даты — `YYYY-MM-DD` строки (через `parseLocalDate`); все строки через `esc()`.
7. Новый блок-владелец не может забрать чужую сущность (проверка в `registerModule`).
8. Пользовательский блок владеет только `custom.<id>`.

### 2.7 Конвенции
- Один HTML-файл, без сборки. Новый блок = **один** `CRM.registerModule(...)`.
- События — только `onclick`-атрибуты (внутри `modal.open()` `<script>` не исполняется).
- Данные — только через `CRM.store.get/set`.
- Миграции — add-only (новые поля с дефолтами), пользовательские данные не перезаписывать.
- Никакого нативного `prompt/confirm` — только `CRM.ui.confirm/formModal`.

---

## 3. ЧТО СДЕЛАНО В ЭТОЙ СЕССИИ

### 3.1 Слой расширяемости + живая карта (commit `bc522ae`)
- **`CRM.registerModule` / `CRM.registry` / `CRM.CORE_META` / `CRM.ownerOf` / `CRM.unregisterModule`** —
  декларативная регистрация блоков с гард-проверкой «один владелец на сущность» и авто-нумерацией.
- **`CRM.ui.confirm` / `CRM.ui.formModal`** (оверлей `#dialog`) — заменили **все 14** нативных `confirm()`.
- **Пользовательские блоки в рантайме** (`CRM.customBlocks` + `registerCustomBlocks`): из UI создаётся
  блок типа «список» (колонки `имя:тип`) или «текст»; данные изолированы в `custom.<id>`; переживают
  перезагрузку и экспорт/импорт.
- **Блок 16 «Map»** — живая карта из `CRM.registry`: блоки, владение, граф, инварианты, how-to,
  менеджер кастом-блоков.
- В стор добавлены `customBlocks:[]`, `custom:{}` (+ миграция). В `BLUEPRINT.md` добавлен **§11**
  (модель расширяемости) и обновлены §1/§3/§7/§8/§10.

### 3.2 Фикс краша + английское меню/амплуа/статусы (commit `5251d9b`)
- **Баг:** вкладки Fitness/Medical/Notes падали с `(b.date||'').localeCompare is not a function` —
  в `localStorage` от старой версии лежали даты не-строками (число/таймстамп/объект), сортировка по
  дате крашилась.
- **Фикс:** глобальный `normYmd(v)` + нормализация дат в `store.migrate()` для
  `physical/psycho/medical/notes/calendar/matches/opponents.games/roster.history` → всё приводится к
  `YYYY-MM-DD` при загрузке (инвариант 2.6.6). Данные чинятся сами при следующей загрузке.
- **Английский (через центральные константы — пропагируется везде):** сайдбар `CRM.TABS`, `POS_LABEL`,
  `POS_FULL`, `STATUS_LABEL`, `tournLabel`, фильтр позиций «All», метка блока Map.

### 3.3 Перевод хоккейных терминов (commit `c924d75`)
Переведены статы и сленг (описательная проза оставлена на русском — по решению пользователя):
- Заголовки таблиц: `G/A/P/GP/PIM/Rtg/PPG/PPA/Avg`, `+/−`.
- KPI: `Goals/Assists/Points/Games played/Avg rating/Missed`, `Goals for/against`.
- `Special teams`, `Power-play goals`, `Opponent goals`; результаты `Win/Loss/Tie`.
- `Goalies/Goalie/Goalie notes/"Goalie was changed"`.
- Турниры `season/preseason/playoff` (плашки, селекты, инлайн); `Venue` + `Home/Away`; `Tournament`.
- `Position`, `Shot (Left/Right)`; навыки `Skating/Size/Hockey IQ/Puck/Shot/Offense/Defense`.
- `Lines/Special teams`, `Playoff bracket`, вкладки соперника (`Roster/Leaders/Tactics/Our games/
  Lineups/Preseason`), `Preseason games`, `Champion/Winner`, рост/вес `cm/kg`.

**Осталось на русском (намеренно):** подзаголовки разделов, кнопки (Сохранить/Отмена/Удалить/Открыть),
общие подписи полей (Дата, Игрок, Имя, Команда, Возр.), пустые состояния, внутренняя документация
блока Map.

---

## 4. ЧТО ОСТАЛОСЬ / СЛЕДУЮЩИЕ ШАГИ

1. **Канон «одна точка создания» (BLUEPRINT §2/§10.1) ещё НЕ внедрён в коде:**
   - нет `matches._create` (сейчас матч пушится напрямую в `matches.add()`/`createGamePair`);
   - нет `scouting.promoteToTeam` (моста кандидат→команда);
   - `syncOppLineup` — заглушка (см. ~стр. 859, `/* placeholder */`).
   Это отдельный рефактор; слой расширяемости и карта к нему готовы.
2. **Полный английский интерфейс** (если захотят): кнопки, подзаголовки, общие подписи полей, тексты
   модалок, пустые состояния, тело блока Map. Сейчас переведены только хоккейные термины + меню.
3. **BLUEPRINT §10 хвост:** реальная аналитика на `playerMatchHistory`, экспорт PDF/CSV + `@media print`,
   кэш `playerMatchHistory` (`CRM._pmhCache`).
4. **⌘K-поиск** расширить на матчи и заметки (BLUEPRINT §9).

---

## 5. КАК ТЕСТИРОВАЛИ (воспроизводимо)

Браузера нет в окружении, поэтому проверяли через Node:
1. **Синтаксис:** выдрать `<script>`-блок приложения (тот, что содержит `const CRM = window.CRM`) в
   `/tmp/app.js` и `node --check /tmp/app.js`.
2. **Рантайм (headless-харнесс):** замокать `window/document/localStorage/Chart/location`, выполнить
   скрипт через indirect-eval, затем:
   - `CRM.boot()`, прогнать `CRM.router.render()` по всем `CRM.TABS` — нет `Ошибка модуля`;
   - засеять `localStorage` записями с «битыми» датами (number/timestamp/{}), убедиться, что
     `migrate` нормализует и вкладки не падают;
   - проверить `registerModule` (гард владельца, дубликаты, авто-num), кастом-блок (create/render/
     delete), экспорт/импорт;
   - проверить, что в отрендеренном HTML нет хоккейного русского и есть `Goals/Assists/Points`, амплуа.

Последние прогоны: синтаксис OK; все 16 вкладок + профиль + отчёт матча рендерятся без ошибок.

---

## 6. БЫСТРЫЙ СТАРТ ДЛЯ НОВОГО ЧАТА (можно вставить как первый промпт)

> Контекст: офлайн-CRM `crm_hockey.html` в репо BadMause/MyCode, ветка
> `claude/busy-lamport-7lnt8o`. Архитектура и правила — в `BLUEPRINT.md` (особенно §11) и
> `handoff.md`. Принципы: один владелец на сущность; новый блок — один вызов
> `CRM.registerModule(...)`; данные через `CRM.store.get/set`; даты `YYYY-MM-DD`; строки через
> `esc()`; диалоги через `CRM.ui.confirm/formModal` (не нативные). Прочитай `handoff.md` и
> `BLUEPRINT.md`, держи карту в контексте. Задача: <…>.

Перед сдачей любой правки: прогнать инварианты §2.6 и (если правил JS) синтаксис/рендер по §5.
