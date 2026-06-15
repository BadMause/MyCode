# HANDOFF — Hockey Coach CRM

> Полный конспект для переноса в другой чат: что за проект, карта логики, что сделано, что
> обсуждалось, что осталось. Самодостаточный — новый чат может работать только по нему + по коду.
>
> **Дата:** 2026-06-15 · **Репозиторий:** `BadMause/MyCode` · **Ветка:** `claude/admiring-carson-kv5qmv`
> · **PR:** #1 (https://github.com/BadMause/MyCode/pull/1 — пуши в ветку обновляют его, новый PR не создавать).
> Глубокая спецификация — в `BLUEPRINT.md`. Этот файл — практический «где мы сейчас».

---

## 0. TL;DR

- **Что это:** один автономный офлайн-файл **`crm_hockey.html`** (~2300 строк, vanilla JS, без сборки,
  Chart.js встроен). CRM для хоккейного тренера.
- **Хранилище:** `localStorage`, ключ **`crm_hockey_v1`**, namespace `window.CRM`.
- **Запуск:** открыть `crm_hockey.html` двойным кликом в браузере. Сервер не нужен. Бэкап — в
  Settings → экспорт/импорт JSON.
- **Состояние:** всё ТЗ заказчика (милстоуны A–I) **выполнено** в этой сессии. 15 блоков. Блок «Map»
  удалён (карта логики ведётся здесь, а не в приложении).
- **Принципы (не нарушать):** один владелец на сущность; данные только через `CRM.store.get/set`;
  даты — строки `YYYY-MM-DD`; все строки через `esc()`; события — только `onclick`/`ondrag*` атрибуты
  (внутри модалок `<script>` НЕ исполняется); диалоги — только `CRM.ui.confirm/formModal` (не нативные);
  новый блок = `CRM.modules.x` + строка в `CRM.TABS` + иконка в `CRM.ICONS`.

---

## 1. ФАЙЛЫ И GIT

| Файл | Что это |
|---|---|
| `crm_hockey.html` | Само приложение (единственный рабочий файл). |
| `BLUEPRINT.md` | Спецификация архитектуры (ownership, инварианты, потоки данных). |
| `handoff.md` | Этот файл — статус + живая карта логики. |
| `.gitignore` | Игнорит `.shots/`. |

**Коммиты сессии 2 (новые → старые):**
```
de8a64a  Periodization: rename + key milestones + planner JSON import (G)
2c9e8b6  Calendar: monthly grid + day detail + Organizational Schedule (H)
7d9124c  Physical Assessments, Mental Fitness Evaluation, Medical color-coding (E/F/I)
245557d  Match: goal-zone tagging on clickable rink + tag stats (D2)
3ce2307  Match: card previews, pre-match planning, period/team stats, shootout (D1)
230eba9  Roster: interactive drag-drop lineup board, tactics/staff/history tabs (C)
36e2cf3  Scouting: position-specific evaluation engine, color grades, priorities, compare (A+B)
5bdbd48  Remove in-app Map/extensibility block; keep logic map in handoff.md
fa5817e  Add handoff.md (база сессии 2)
```
**До сессии 2:** `c924d75` (перевод хоккейных терминов), `5251d9b` (фикс краша дат + англ. меню),
`bc522ae` (слой расширяемости + Map — позже откатан), `34725c0`/`3155554`/`73b435f`/`51c1956` (старт).
Всё запушено в `origin/claude/admiring-carson-kv5qmv` → PR #1.

---

## 2. КАРТА ПРИЛОЖЕНИЯ (логика)

### 2.1 Инфраструктура (namespace `CRM`)
Номера строк — ориентир (могут смещаться, ищи grep’ом).

| Объект | Строка | Назначение |
|---|---|---|
| `const CRM` | 215 | `{ modules:{}, LSKEY:"crm_hockey_v1" }` |
| `CRM.TABS` | 218 | `[id,num,label,desc]` × 15 — сайдбар. |
| `CRM.ICONS` | ~235 | SVG-иконки блоков. |
| Хелперы | ~245 | `uuid, esc, parseLocalDate, localYmd, normYmd, fmtDate, fmtDateShort, num, clamp, avg, round1`. |
| `MONTHS_RU`, `monthMatrix`, `monthGridHtml` | ~262 | Генерик месячная сетка (исп. в Physical и Calendar). |
| `POS_LABEL/POS_FULL/STATUS_LABEL/STATUS_CLASS` | ~290 | Амплуа/статусы (англ.). |
| `gradeColor` | 296 | Цвет по грейду (цвет-тиры + старые буквы). |
| `playerById/scoutById/playerName/playerSurname/playerLabel` | ~280 | Доступ к игрокам. |
| `sectionTitle/chartOpts/makeChart` | ~300 | Заголовок, опции и фабрика Chart.js. |
| `CRM.store` | ~315 | `load/get(path)/set(path,val)/save/exportJson/importJson/resetAll/defaults/defaultNorms`; внутри `migrate()` (add-only + нормализация дат + миграция beep→cooper). |
| `CRM.toast` | ~395 | Тосты. |
| `CRM.modal` | 406 | `open(html)/close()/frame(title,body)`. `<script>` внутри НЕ исполняется. |
| `CRM.modalNav` | ~420 | Стек «назад» для модалок. |
| `CRM.router` | 440 | `parse()/go(name,params)/render()`. Маршрут из `location.hash`; ошибки модулей ловятся в try/catch → маркер `Ошибка модуля`. |
| `CRM.boot` | 473 | Точка входа (`DOMContentLoaded`): `store.load()` → шапка → сайдбар → `render()`. |
| `EVAL_MODEL` + `evalScore/isEvaluated/evColor/posGroup/evalModelFor` | 550 | Движок скаут-оценки (см. §2.5). |
| `GRADE_TIERS` + `gradeTier/gradeChip` | 585 | Цветовые тиры грейда (см. §2.5). |
| `CRM.ui` | 2246 | Диалоги вместо нативных: `confirm({title,message,danger,okLabel,onOk})`, `formModal({title,fields,onSubmit})`. Поля: `text/number/date/textarea/select/checkbox`. Оверлей `#dialog`. |

> Удалено в сессии 2: `CRM.registry/registerModule/ownerOf/unregisterModule/CORE_META/customBlocks/
> registerCustomBlocks` и блок 16 «Map». Оставлен только `CRM.ui`.

### 2.2 Блоки (15) — начало `CRM.modules.X={...}`

| # | id (label) | Строка | Владеет (store) | Кратко |
|---|---|---|---|---|
| 01 | dashboard (Dashboard) | 485 | — (read-only) | KPI, алерты, форма |
| 02 | scouting (Scouting) | 595 | `scoutDB, scoutTarget, scoutPriorities` | кандидаты + **оценка** (см. §3.A) |
| 03 | profiles (Players) | 764 | `scout[]` | игроки команды + контракт/агент + зеркало оценки |
| 04 | roster (Lines) | 1311 | `roster.*` (+`staff`) | drag-drop состав, штаб, история (см. §3.C) |
| 05 | matches (Games) | 887 | `matches[]` (+ пишет `opponents[].lineups`) | карточки, предматч, зоны голов (см. §3.D) |
| 06 | stats (Stats) | 1437 | — | командная/инд. статистика |
| 07 | opponents (Opponents) | 1486 | `opponents[], playoffBracket` | соперники, плей-офф |
| 08 | physical (Assessments) | 1676 | `physical[]` | тесты/динамика/календарь (см. §3.E) |
| 09 | psycho (Mental) | 1759 | `psycho[]` | оценка + достоверность (см. §3.F) |
| 10 | preseason (Periodization) | 1868 | `preseason{}` (+`milestones`) | Бомпа + вехи + импорт (см. §3.G) |
| 11 | calendar (Calendar) | 2026 | `calendar[]` | месяц/список/оргплан (см. §3.H) |
| 12 | medical (Medical) | 1808 | `medical[]` (+ пишет `scout[].status`) | цвет-доска, травмы (см. §3.I) |
| 13 | analytics (Analytics) | 2108 | — | графики |
| 14 | notes (Notes) | 2135 | `notes[]` | заметки |
| 15 | settings (Settings) | 2166 | `settings{}` | нормативы, экспорт/импорт JSON |

### 2.3 Владение и потоки данных (стрелка = «пишет в»)
```
opponents ──► matches[]              (createGamePair)
matches   ──► opponents[].lineups    (updOppSlot/syncOppLineup — заглушка)
matches  ◄──► roster.history.matchId / matches.lineupId   (двусторонняя — чистить обе стороны)
medical   ──► scout[].status         (syncStatus)
scouting  ──► scout[]                (toTeam — мост кандидат→команда)
scouting/profiles ──► scout/scoutDB.eval  (общий движок scouting.evaluate(id,'db'|'team'))
ЧИТАТЕЛИ: dashboard, stats, analytics, calendar
```

### 2.4 Схема localStorage (`store.defaults()`, с полями сессии 2)
```js
{
  scout:[],            // игроки команды: {id,surname,firstName,number,pos,shot,age,h,w,status,
                       //   contract,agent,note,eval:{'<cat>.<param>':1..5},skills,name}
  scoutDB:[],          // кандидаты: то же + grade(цвет-тир),gkNotes,team (без contract/status медкарты)
  roster:{ lines:{<slot>:pid}, special:{}, history:[], showSpecial, staff:{g1:{title,members[]},g2:{...}} },
  matches:[],          // {id,date,opp,oppId,home,playoff,preseason,score,rating,notes,lineupId,
                       //   prep:{tasks,oppLines,officials[],oppStrengths,oppWeaknesses,planA,planB,ppTactics,pkTactics},
                       //   result:{finished,ppGoals,ppAttempts,pkGoalsAgainst,pkTimes,gk,playerStats,
                       //     comboRatings,periods[],team{shots/sog/fo/hits/blocks},shootout{happened,ours,theirs,goalieId},goals[]}}
  opponents:[], playoffBracket:{r8,r4,r2,f},
  physical:[],         // {id,playerId,date,results:{<normId>:num}}  (norm 'beep'→'cooper' мигрирован)
  psycho:[],           // {id,playerId,date,scores:{...},lie:{l1,l2,l3},note}
  practice:[], calendar:[],  // calendar event: {id,date,title,kind('тренировка'|...|'оргплан'),note}
  medical:[],          // {id,playerId,date,injuryType,bodyPart,diagnosis,status,returnDate,treatment,statusNote}
  preseason:{ plan[7][7]:{phase,intensity,minutes}, details[7][7], fatigue[7], cal:{start,end,seasonEnd},
              seasonPlan, seasonDetails, milestones:[{id,date,title,note}] },
  notes:[],
  settings:{ coach, team, league, season, norms:[{id,name,lower,gold,silver,bronze}] },
  scoutTarget:{}, scoutPriorities:{ F:[ids], D:[ids], G:[ids] },  // приоритеты по позиции (drag-drop)
  state:{ seedOpp10:false }
}
```

### 2.5 Движок скаут-оценки (ядро милстоуна A)
- **`EVAL_MODEL`** (стр. 550): `{ F:{label,cats:[{key,label,params:[[key,label],...]}]}, D:{...}, G:{...} }` —
  data-driven категории→параметры по позиции, шкала 1–5. **Чтобы изменить таксономию — править только этот объект.**
- `posGroup(pos)` → 'F'|'D'|'G'; `evalModelFor(pos)` → модель.
- Хранение: `player.eval['<catKey>.<paramKey>'] = 1..5`.
- `evalScore(p)` → `{overall(среднее всех параметров), cats:{catKey:среднее}, scored, total}`.
- `isEvaluated(p)` → есть ли хоть один балл. `evColor(v)` — цвет шкалы.
- **`GRADE_TIERS`** (стр. 585): цвет-тиры (= картинка 1): `green`(топ-5/ключ.звено), `purple`(5–10),
  `red`(скорость/размер), `yellow`(ролевой/PK), `white`(давление/роль). `gradeChip(key)` рендерит плашку.
  Грейд — ручной тир, отдельно от вычисляемого балла.

### 2.6 Инварианты (проверять перед сдачей)
1. `roster.history[].matchId` ⇄ `matches[].lineupId` согласованы; удаление чистит обе стороны.
2. `scout[].status` пишет только `medical` (`syncStatus`).
3. `scoutDB`(кандидаты) ↔ `scout`(команда) — мост только `scouting.toTeam`.
4. Статистика читается только из `result.finished===true`.
5. Даты — `YYYY-MM-DD` строки (через `parseLocalDate`/`normYmd`); все строки через `esc()`.
6. Позиция в составе: G→только G, D→только D, F-слот→любой не-G (`roster._eligible`).

### 2.7 Конвенции
- Один HTML-файл, без сборки. Новый блок = `CRM.modules.x` + `CRM.TABS` + `CRM.ICONS`.
- Внутри-блочные подвиды — через `params.view` и `CRM.router.go('<id>',{view:'...'})`.
- События — только инлайн-атрибуты (`onclick`, `ondragstart/over/drop`, `onchange`).
- Миграции — add-only; пользовательские данные не перезаписывать.

---

## 3. ЧТО СДЕЛАНО В СЕССИИ 2 (милстоуны A–I)

Перед фичами: **синхронизация веток** (рабочая ветка была на пресессионном коммите — fast-forward на
состояние из handoff) и **удаление блока Map** (по запросу: «Map убрать из системы, карту логики вести
в Handoff.md»). `CRM.ui` оставлен.

- **A. Scouting (02) + B. Profiles (03)** — `36e2cf3`. Движок `EVAL_MODEL` (поз.-зависимые формы D/F/G),
  табы **Кандидаты / Градация (цвет-тиры) / Приоритеты (HTML5 drag-and-drop по позиции) / Сравнение
  (side-by-side + радар)**, свёрнуто-развёрнутый отчёт (`<details>` + радар), статус Оценён/Не оценён,
  промоут `toTeam`. Профили: + **тип контракта + агент**, зеркало оценки (общий `scouting.evaluate(id,'team')`).
- **C. Roster (04)** — `230eba9`. Табы **Состав / Тактика / Штаб / История**. Интерактивная drag-drop доска
  (`_slotBox/_placeFromDrag`/пул, клик-постановка, проверка позиции, своп). Тактика (подсказки ПП/ПК из eval,
  спецбригады, лучшие сочетания). Штаб (две группы `roster.staff`). История split матч/тренировка.
- **D. Match (05)** — `3ce2307` (D1) + `245557d` (D2). Список карточками. Предматч: судьи, силы/слабости
  соперника, План A/B, тактика ПП/ПК. Итоги: счёт по периодам, командная стата, буллиты. **Зоны голов:**
  клик по SVG-площадке (`rinkClick`), теги способа (`GOAL_TAGS`), агрегация `goalStats`. `ensureExt` бэкфилит.
- **E. Physical → «Assessments» (08)** — `7d9124c`. bip-test → **Cooper** (миграция id+записей); ввод
  исключает травмированных/ограниченных; табы Результаты / Динамика (line+doughnut, период) / Календарь.
- **F. Psycho → «Mental Fitness Evaluation» (09)** — `7d9124c`. lie-scale достоверности (`validity` %),
  отчёт с радаром, баллами по уровням и текстовой интерпретацией.
- **G. Periodization (10)** — `de8a64a`. Переименование; **ключевые вехи** (`preseason.milestones`);
  **импорт JSON планировщика** (`importPlanner/doImport`, ключи plan/details/fatigue/cal или ps9p/d/f/c).
  Движок Бомпы (фазы, нагрузка, детектор конфликтов, индекс усталости, автопостроение) уже был.
- **H. Calendar (11)** — `2c9e8b6`. Месячная сетка с цветными индикаторами + день-детал + **Оргплан**
  (печатаемые расписания, картинка 2). Helper `monthGridHtml`.
- **I. Medical (12)** — `7d9124c`. Цвет-доска здоровья (🟢🟡🔴 по игрокам), тип травмы + часть тела +
  лечение + срок восстановления (обратный отсчёт), быстрое добавление по игроку.

---

## 4. ЧТО ОБСУЖДАЛОСЬ (решения)

- **Ветки:** работа идёт в `claude/admiring-carson-kv5qmv` (синхронизирована с `claude/busy-lamport-7lnt8o`).
- **Map:** убран из приложения по запросу; карта логики — в этом файле.
- **Таксономия скаут-оценки:** заказчик сначала выбрал «вставлю свои категории», затем сказал «делай всё» →
  использована **моя предложенная** таксономия (data-driven, легко заменить — см. §2.5 и §5).
- **Шкала параметров:** 1–5.
- **«сделай всё по очереди»** → выполнены все милстоуны A–I подряд, каждый с коммитом и проверкой.
- **PDF не читаются** в окружении (нет тулинга/сети) — делалось по 3 присланным картинкам:
  (1) цветовая философия линий → грейды/ростер; (2) расписание «Магнитогорск» → Calendar/Оргплан;
  (3) точки/зоны голов на льду → Match «Зоны голов».
- **PR #1** создан из Claude Code UI; пуши в ветку обновляют его (новый PR не создавать).

---

## 5. ЧТО ОСТАЛОСЬ / СЛЕДУЮЩИЕ ШАГИ

1. **Точные категории оценки** заказчика → заменить объект `EVAL_MODEL` (стр. 550). Движок/UI/радары/сравнение
   останутся как есть.
2. **Технический долг BLUEPRINT** (не входил в это ТЗ): канон `matches._create` / `scouting.promoteToTeam` /
   реальный `syncOppLineup` (сейчас заглушка ~стр. 905); реальная аналитика (блок 13) на `playerMatchHistory`;
   экспорт PDF/CSV + `@media print`; ⌘K-поиск по матчам/заметкам.
3. **Опционально:** полный английский интерфейс (сейчас переведены термины + меню; проза/кнопки на русском);
   визуальная доска «План A/B» в матче (сейчас текстом).

---

## 6. КАК ТЕСТИРОВАТЬ (воспроизводимо, без браузера)

1. **Синтаксис:** выдрать второй `<script>` (с `const CRM`) в `/tmp/app.js`, `node --check /tmp/app.js`.
2. **Рантайл (headless-харнесс):** замокать `window/document/localStorage/Chart/location` (forgiving
   Proxy-элемент с настоящими `innerHTML`), выполнить скрипт через indirect-eval `(0,eval)`, затем:
   `CRM.boot()` → прогнать `CRM.router.render()` по всем `CRM.TABS` (нет маркера `Ошибка модуля`);
   засеять `localStorage` и проверить миграции (даты, beep→cooper); прогнать подвиды и методы каждого модуля
   (scouting eval/drag/compare; roster доска; matches зоны/итоги; physical charts/cal; psycho validity;
   medical board; calendar month/org; periodization milestones/import).
3. Последний прогон сессии 2: синтаксис OK; все 15 вкладок + все новые пути — без ошибок.

> Браузера в окружении нет — визуальные баги ловятся только вручную. Если что-то выглядит не так:
> назвать вкладку + что видно.

---

## 7. БЫСТРЫЙ СТАРТ ДЛЯ НОВОГО ЧАТА (вставить как первый промпт)

> Контекст: офлайн-CRM `crm_hockey.html` в репо BadMause/MyCode, ветка `claude/admiring-carson-kv5qmv`
> (PR #1 — пуши обновляют его). Архитектура и правила — в `handoff.md` (особенно §2 карта, §2.5 движок
> оценки) и `BLUEPRINT.md`. Принципы: один владелец на сущность; данные через `CRM.store.get/set`; даты
> `YYYY-MM-DD`; строки через `esc()`; диалоги `CRM.ui.confirm/formModal`; новый блок = `CRM.modules.x` +
> `CRM.TABS` + `CRM.ICONS`; НЕ возвращать блок «Map». Всё ТЗ A–I выполнено. Прочитай `handoff.md`. Задача: <…>.

Перед сдачей правки: прогнать инварианты §2.6 и (если правил JS) синтаксис/рендер по §6.
