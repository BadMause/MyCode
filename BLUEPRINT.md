# BLUEPRINT — Единый Hockey Coach CRM

> Карта проекта для Claude Code. Цель: один offline-файл (`crm_hockey.html`, vanilla JS,
> localStorage `crm_hockey_v1`), 15 блоков, поглощающий скаут-базу и планировщик предсезонки.
> Главные принципы: **один источник истины на сущность**, **одна точка создания на сущность**,
> **двусторонние связи не рвём**. Держи эту карту в контексте при любой правке.
>
> Живая карта логики ведётся в `handoff.md` (по решению заказчика — НЕ как блок в приложении).
> При добавлении/изменении блока обнови §1/§3 здесь и §2/§4 в `handoff.md`.

---

## 0. ИСХОДНОЕ СОСТОЯНИЕ (что объединяем)

| Файл | Роль | Ключи localStorage | Судьба |
|---|---|---|---|
| `crm_hockey.html` | Главный CRM, 15 блоков | `crm_hockey_v1` | **БАЗА.** В неё вливаем остальные. |
| `scout_database_3.html` | Скаут-база МХЛ (grade, skills, рост/вес) | `mhl_scout_players_v1`, `mhl_scout_roster_v1` | Поглощается → блок 02 `scouting` (`scoutDB`). |
| `preseason_planner_v15_2.html` | Планировщик предсезонки (day/week/fatigue/load) | `ps9c/d/f/p` | Поглощается → блок 10 `preseason`. |

Скаут-база и предсезонка **сейчас живут в двух местах** (в CRM и отдельными файлами) — это
корень дублирования. После слияния отдельные файлы становятся read-only архивом, единственный
рабочий файл — CRM.

---

## 1. ГЛАВНЫЙ ПРИНЦИП: ВЛАДЕНИЕ ДАННЫМИ (ownership)

Каждая сущность имеет **ровно одного владельца-модуль**, который её создаёт, редактирует и удаляет.
Остальные модули — **только читатели** (или дописывают свои поля через владельца). Это убирает
конфликт «создать можно из 5 мест».

| Сущность | store-ключ | ВЛАДЕЛЕЦ (CRUD) | ЧИТАТЕЛИ |
|---|---|---|---|
| Игрок команды | `scout[]` | **03 profiles** | dashboard, roster, matches, stats, physical, psycho, medical, analytics |
| Скаут-кандидат | `scoutDB[]` | **02 scouting** | dashboard (алерты целей) |
| Состав/звенья | `roster.lines`, `.special`, `.history[]` | **04 roster** | matches (читает history), dashboard |
| Соперник | `opponents[]` | **07 opponents** | matches (выбор соперника), dashboard, calendar |
| Матч | `matches[]` | **05 matches** | stats, profiles, roster, dashboard, calendar, analytics |
| Состав соперника на матч | `opponents[].lineups[]` | **05 matches** (через `syncOppLineup`) | opponents (частотный анализ) |
| Физтест | `physical[]` | **08 physical** | profiles, analytics |
| Психотест | `psycho[]` | **09 psycho** | profiles, analytics |
| План предсезонки | `preseason{plan,details,fatigue,...}` | **10 preseason** | calendar, dashboard, analytics |
| Событие календаря | `calendar[]` | **11 calendar** | dashboard |
| Травма | `medical[]` | **12 medical** | profiles, dashboard, analytics; **пишет** `scout[].status` |
| Заметка | `notes[]` | **14 notes** | dashboard, поиск ⌘K |
| Настройки/нормативы | `settings{}` | **15 settings** | physical (нормы), экспорт (шапка) |

**Правило:** если модуль не владелец — он не имеет своей кнопки «Создать X». Он даёт ссылку
«перейти к владельцу» или встроенную форму, которая зовёт метод владельца.

---

## 2. ЕДИНСТВЕННАЯ ТОЧКА СОЗДАНИЯ (anti-duplication)

Самая большая боль — «много мест, где создаётся одно и то же». Закрепляем канон:

### 2.1 Матч — 3 входа, 1 конструктор
Сейчас матч создаётся из `matches.add()`, `opponents.createGamePair()` и из roster.
**Канон:** один низкоуровневый конструктор `matches._create(payload)` (единственный, кто делает
`matches.push`). Все три входа — лишь обёртки, собирающие `payload` и зовущие `_create`:
- `matches.add()` — модальная форма (дата, соперник из `opponents`, дом/плей-офф/предсезонка).
- `opponents.createGamePair(opId,...)` — создаёт игру у соперника И зовёт `matches._create`,
  сразу проставляя `oppId`. Не дублирует логику матча.
- из roster — кнопка «создать матч под этот состав» тоже зовёт `matches._create`, затем привязка.

Никто, кроме `matches._create`, не пишет в `matches[]`.

### 2.2 Игрок команды vs скаут-кандидат — не путать
- `scout[]` (команда) создаётся **только** в `profiles`.
- `scoutDB[]` (кандидаты) создаётся **только** в `scouting`.
- Переход кандидата в команду = явный метод `scouting.promoteToTeam(sdId)` → создаёт запись в
  `scout[]` через `profiles._create(...)` и помечает кандидата. Это единственный мост.

### 2.3 Состав соперника
Пишется **только** через `matches.syncOppLineup()` (upsert в `opponents[].lineups`). Модуль
`opponents` его лишь читает/анализирует. Нет второй точки записи.

### 2.4 Связь матч ↔ состав (двусторонняя, не рвать)
`roster.history[].matchId` ↔ `matches[].lineupId`. При удалении любой стороны — отвязывать вторую
(`roster.delFromModal` чистит `matchId`; удаление матча чистит `lineupId`). Это инвариант №1.

---

## 3. КАРТА БЛОКОВ (15 блоков)

| # | name | Владеет | Создаёт | Главные методы |
|---|---|---|---|---|
| 01 | dashboard | — | ничего (read-only) | KPI, алерты травм, форма |
| 02 | scouting | `scoutDB`, `scoutTarget`, `scoutPriorities` | скаут-кандидата | грейды, радар, `promoteToTeam` |
| 03 | profiles | `scout[]` | игрока команды | `open(id)`, OVERVIEW(6 KPI), `_create` |
| 04 | roster | `roster.*` | состав/звено/историю | `createLineupForm→savePending`, `openHistory`, PP/PK hints |
| 05 | matches | `matches[]`, пишет `opp.lineups` | матч (`_create`) | подготовка/матч-день/итоги/отчёт, `playerMatchHistory`, `finishMatch` |
| 06 | stats | — | ничего | `matchPass`, `collect` (читает `playerMatchHistory`) |
| 07 | opponents | `opponents[]`, `playoffBracket` | соперника, его игру | `createGamePair`, `delGame`, `paneLineups`, preseason |
| 08 | physical | `physical[]` | физтест | нормы из `settings.norms` |
| 09 | psycho | `psycho[]` | психотест | — |
| 10 | preseason | `preseason{}` | план/детали/усталость | поглощает planner (см. §4) |
| 11 | calendar | `calendar[]` | событие | ссылается на matches, не дублирует |
| 12 | medical | `medical[]`, пишет `scout[].status` | травму | синхрон статуса игрока |
| 13 | analytics | — | ничего | 4 графика на РЕАЛЬНЫХ данных |
| 14 | notes | `notes[]` | заметку | поиск ⌘K |
| 15 | settings | `settings{}` | — | нормативы, экспорт/импорт JSON |

> 15 блоков задаются статически: `CRM.modules.x`, строка в `CRM.TABS`, иконка в `CRM.ICONS`.
> Блок «Map» и слой расширяемости удалены (сессия 2) — карта логики ведётся в `handoff.md`.

---

## 4. ПЛАН ПОГЛОЩЕНИЯ ОТДЕЛЬНЫХ ФАЙЛОВ

### 4.1 scout_database_3 → блок 02 `scouting`
- Источник: `mhl_scout_players_v1` (массив игроков с grade/position/skills/height/weight/team),
  `mhl_scout_roster_v1` (целевой состав), `mhl_compare_sel` (выбор для сравнения — runtime, не мигрируем).
- Маппинг полей в `scoutDB[]`:
  `name→firstName/surname`, `position→pos`, `height→h`, `weight→w`, `shot→shot`,
  `grade→grade`, `skills{skating,iq,puck,shot,...}→skills`, `team→team`.
- Импорт: разовый — кнопка «Импорт из старой скаут-базы (JSON)» в `scouting`, читает старый JSON,
  upsert по (name+birth) чтобы не плодить дубли.
- Фича сравнения игроков (`mhl_compare_sel`) — портировать как режим внутри `scouting`.

### 4.2 preseason_planner_v15_2 → блок 10 `preseason`
- Источник: `ps9p` (plan), `ps9d` (details), `ps9f` (fatigue), `ps9c` (cal: start/end).
- Маппинг в `preseason{ plan, details, fatigue, cal:{start,end,seasonEnd}, seasonPlan, seasonDetails }`.
- День/неделя/нагрузка/интенсивность — сохранить как в planner; усталость `fatigue[week]`.
- Импорт: кнопка «Импорт плана предсезонки (JSON)» в `preseason`.

После слияния старые ключи не использовать. Конвертер — отдельный одноразовый метод, не трогающий
`store.load()`.

---

## 5. ПОТОКИ ДАННЫХ (data flow)

### 5.1 Жизненный цикл матча
`matches._create` → `open(id)`:
ПОДГОТОВКА (привязать наш состав `attachLineup`; собрать состав соперника `updOppSlot→syncOppLineup`)
→ ИТОГИ (`saveResult`: счёт, `playerStats`, PP/PK, 2 вратаря)
→ `finishMatch()` (валидация + попытки PP/PK) → `finished=true` → `matchReport()`.

### 5.2 Источник истины «сыграл ли игрок»
**Только** `matches.playerMatchHistory(pid)`, читает завершённые матчи. Нигде не дублируется.
Потребители: profiles OVERVIEW, profilePane, stats.collect, roster.pp/pkHints.
Кэш: `CRM._pmhCache{pid→result}`, сброс по версии-счётчику при saveResult/finishMatch/удалении.

### 5.3 Статус игрока
`medical` — единственный, кто пишет `scout[].status`. profiles/dashboard только читают.

---

## 6. ГРАФ ЗАВИСИМОСТЕЙ (стрелка = «пишет в»)

```
opponents ──► matches[]            (createGamePair зовёт matches._create)
matches   ──► opponents.lineups    (syncOppLineup)
matches   ──► roster.history.matchId   ◄──► roster ──► matches.lineupId  (двусторонняя!)
medical   ──► scout.status
scouting  ──► scout[]              (promoteToTeam → profiles._create)
profiles  ──► scout[]              (владелец)
roster ──► matches[].lineupId
ЧИТАТЕЛИ (никуда не пишут): dashboard, stats, analytics, calendar, profiles(OVERVIEW)
ОБЩИЕ: playerById, scoutById, esc(), parseLocalDate/localYmd, CRM.store, CRM.modal, modalNav
```

Тот же граф поддерживается вручную в `handoff.md` §2.3 (блок «Map» удалён).

---

## 7. ИНВАРИАНТЫ ЦЕЛОСТНОСТИ (проверять перед сдачей)

1. `roster.history[].matchId` ↔ `matches[].lineupId` всегда согласованы; удаление чистит обе стороны.
2. Игрок не может быть в двух спецбригадах одновременно (валидация в roster).
3. `matches[]` пополняется ТОЛЬКО через `matches._create`.
4. `scout[].status` пишет ТОЛЬКО medical.
5. `scoutDB[]` (кандидаты) и `scout[]` (команда) не смешиваются; мост — только `promoteToTeam`.
6. Статистика читается ТОЛЬКО из `result.finished===true`.
7. Даты — строки `YYYY-MM-DD`, сравнение через `parseLocalDate` (локальная TZ).
8. Все пользовательские строки — через `esc()`.
9. Турнир кодируется двумя флагами: `playoff`, `preseason` (оба false = сезон).
10. Вратари (`pos==='G'`) — отдельная статистика и правило «сыграл».

---

## 8. КОНВЕНЦИИ (НЕ ЛОМАТЬ)

1. Один HTML-файл, без сборки. Новый блок = `CRM.modules.x` + строка в `CRM.TABS` + иконка в
   `CRM.ICONS` (три места). Описать блок в §1/§3 и в `handoff.md`.
2. События — только `onclick`-атрибуты; `<script>` внутри `modal.open()` не исполняется.
3. Данные — только через `CRM.store.get/set` (точечные пути).
4. Миграции — только добавление полей с дефолтами; данные пользователя не перезаписывать.
5. Переходы «модалка→модалка» не зовут `close()` перед `open()` (иначе рвётся navback-стек).
6. Никакого нативного `prompt()/confirm()` — только `CRM.ui.confirm(...)` и `CRM.ui.formModal(...)`
   (отдельный оверлей `#dialog`, поверх обычной модалки).

---

## 9. НАВИГАЦИЯ (вторая боль)

- Единый сайдбар из `CRM.TABS` — единственная точка перехода между блоками.
- Внутри блоков — модалки + `modalNav` (стек «назад», макс 12, дедуп по модуль+первый-аргумент).
- Кросс-ссылки вместо дублей: из матча → профиль игрока, из профиля → его матчи (`profilePane`),
  из соперника → наши игры с ним. Везде ссылка на владельца, не копия данных.
- Глобальный поиск ⌘K расширить на матчи и заметки (сейчас не ищет).

---

## 10. ПОРЯДОК РАБОТ ДЛЯ CLAUDE CODE

1. Закрепить ownership (§1) и единые конструкторы `_create` (§2) — рефактор без новых фич.
2. Поглощение скаут-базы (§4.1) и планировщика (§4.2) с разовыми импортёрами.
3. ~~Убрать все `prompt/confirm` → модалки (§8.6).~~ **Сделано:** `CRM.ui.confirm/formModal`.
4. Реальная аналитика (блок 13) на `playerMatchHistory`/`physical`/`medical`.
5. Экспорт PDF/CSV всех таблиц + `@media print`.
6. Кэш `playerMatchHistory` (§5.2) — последним.

После каждой задачи прогнать инварианты §7 на существующих данных.

> Текущий статус кода: `matches._create`, `promoteToTeam`, реальный `syncOppLineup` ещё НЕ внедрены
> — это рефактор по §2/§10.1. Из слоя расширяемости оставлен только `CRM.ui` (диалоги); блок «Map»,
> `registry`/`registerModule`/пользовательские блоки удалены (сессия 2). Большое ТЗ заказчика — в
> `handoff.md` §4.

---

## 11. СЛОЙ РАСШИРЯЕМОСТИ — УДАЛЁН (сессия 2)

> ⚠️ Слой расширяемости (`CRM.registry` / `registerModule` / `ownerOf` / `CORE_META` /
> `customBlocks` / блок «Map») **удалён** по запросу заказчика. В коде оставлен только `CRM.ui`
> (диалоги — §11.3 ниже актуальна). Карта логики ведётся в `handoff.md`. Подразделы 11.1, 11.2,
> 11.4, 11.5 ниже **неактуальны** и оставлены лишь как исторический след.

Цель: добавлять новые блоки **одним вызовом**, не ломая инварианты. Всё живёт в `crm_hockey.html`
в секции «EXTENSIBILITY LAYER» (рядом с `=== MODULES ANCHOR ===`).

### 11.1 `CRM.registry` — единый источник истины для карты
`id → { id, num, label, desc, owns[], creates[], reads[], writes[], readonly, kind }`.
- `kind`: `core` (15 канон), `meta` (16 Карта), `custom` (пользовательские).
- Канон-метаданные лежат в `CRM.CORE_META` и сидируются в registry при загрузке.
- Блок 16 «Карта» строит таблицы (блоки, владение, граф зависимостей) **из этого реестра** —
  поэтому карта всегда актуальна.

### 11.2 `CRM.registerModule(def)` — единственный способ добавить блок
```js
CRM.registerModule({
  id:'training',                 // [a-z][a-z0-9_]*, уникальный
  num:'17',                      // необязательно — назначится сам (max+1)
  label:'Тренировки', desc:'Журнал тренировок',
  icon:'<svg ...>...</svg>',     // 18×18, как в CRM.ICONS
  owns:['training'],             // что блок ВЛАДЕЕТ
  creates:['тренировку'],
  reads:['scout'],               // что только читает
  writes:[{to:'scout',via:'load'}],  // куда пишет (рисуется в графе)
  readonly:false, kind:'core',
  module:{ render(el){ /* обязателен */ }, add(){ /* методы для onclick */ } }
});
```
Что делает: проверяет id/формат/уникальность; **проверяет «один владелец на сущность»**
(`CRM.ownerOf`); регистрирует `CRM.modules[id]`, `CRM.ICONS[id]`, строку в `CRM.TABS`
и метаданные в `CRM.registry`. `def.replace:true` — обновить существующий блок.
`CRM.unregisterModule(id)` удаляет блок (системные `core/meta` удалять нельзя).

### 11.3 `CRM.ui` — диалоги вместо нативных (§8.6)
- `CRM.ui.confirm({title,message,danger,okLabel,onOk})` — подтверждение.
- `CRM.ui.formModal({title,fields:[{key,label,type,value,required,options,hint}],onSubmit})` —
  форма; `type ∈ text|number|date|textarea|select|checkbox`. Свой оверлей `#dialog`.

### 11.4 Пользовательские блоки (без кода, из UI)
Кнопка «+ Новый блок» в блоке 16. Два вида:
- `list` — таблица с колонками (формат `имя:тип`, типы `text|number|date|textarea`);
- `text` — один большой текст.

Определения хранятся в `store.customBlocks[]`, **данные — в `store.custom.<id>`** (своё
пространство → изоляция, инварианты не задеваются). При загрузке `CRM.boot()` зовёт
`CRM.registerCustomBlocks()`, который заново регистрирует их (идемпотентно) и чистит «осиротевшие»
после импорта. Удаление блока стирает и определение, и данные `custom.<id>`.

### 11.5 Чек-лист при добавлении блока
1. Решить, чем блок **владеет** (новый store-ключ) или он только **читатель**.
2. Если владеет — ключ не должен принадлежать другому блоку (registerModule проверит).
3. Добавить дефолт ключа в `store.defaults()` и в `migrate()` (add-only, §8.4).
4. Зарегистрировать через `CRM.registerModule(...)` рядом с `=== MODULES ANCHOR ===`.
5. Обновить §1/§3 этой карты и `CRM.CORE_META` (для core-блоков).
6. Прогнать инварианты §7.
