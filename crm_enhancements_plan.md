# Enhancement Plan — Hockey Coach CRM (`crm_hockey.html`)

> **Scope:** Profile, Profile View, Scout, and Composition blocks.
> **Base implementation:** the session‑2 build described in `handoff.md` (milestones A–I,
> `crm_hockey.html`, namespace `window.CRM`, store key `crm_hockey_v1`).
> **Companion artifact:** `crm_enhancements_mockup.html` — a self‑contained, interactive
> prototype of every change below, ready to open in a browser.

> ⚠️ **Branch/version note (must read before implementing).** The checked‑out branch
> `claude/gallant-archimedes-okmrke` currently sits at `34725c0`, a **pre‑session‑2** state, so
> its `crm_hockey.html` does **not** contain the modules this plan extends (Scouting eval engine,
> drag‑drop roster, Match zones, etc.). Per `handoff.md` the canonical session‑2 code is on
> `claude/admiring-carson-kv5qmv` (PR #1). Line numbers below reference the **session‑2** file
> (the uploaded `crm_hockey_5.html`). Confirm the integration target branch before applying.

---

## 0. Cross‑cutting helpers & data model

Add three pure helpers next to the existing date helpers (`crm_hockey.html` ~line 257):

```js
function ageFromDob(dob){ const d=parseLocalDate(dob); if(!d) return null;
  const t=new Date(); let a=t.getFullYear()-d.getFullYear();
  const m=t.getMonth()-d.getMonth(); if(m<0||(m===0&&t.getDate()<d.getDate())) a--; return a; }

function monthsUntil(dateStr){ const d=parseLocalDate(dateStr); if(!d) return null;
  return (d-new Date())/(1000*60*60*24*30.44); }

// Color gradation analogous to the Scout block's GRADE_TIERS, but time‑driven.
function contractTier(expiry){ const m=monthsUntil(expiry);
  if(m==null) return {key:'none',  color:'#c9c9cf', label:'Без даты'};
  if(m<0)     return {key:'expired',color:'#7a1f1f', label:'Истёк'};
  if(m<=3)    return {key:'urgent', color:'#e24b4a', label:'< 3 мес'};
  if(m<=12)   return {key:'soon',   color:'#ba7517', label:'< 1 года'};
  if(m<=24)   return {key:'mid',    color:'#0071e3', label:'1–2 года'};
  return            {key:'long',   color:'#1d9e75', label:'> 2 лет'}; }
```

**Store / migration (`CRM.store`, add‑only — invariant §2.7).** `scout[]` player objects gain two
optional `YYYY-MM-DD` strings: `dob` and `contractExpiry`. No `defaults()` change is required for
existing rows; `normalizePlayer()` (~line 365) should default both to `''`. `age` remains stored as a
fallback but is **displayed** as `ageFromDob(p.dob) ?? p.age`. Dates flow through `normYmd()` like all
others (invariant §2.6.5). **Do not migrate or remove `status`** (see Block 1).

---

## 1. Profile Block — creation form

**File:** `CRM.modules.profiles.edit()` (~line 832) and `save()` (~line 853).

| Change | Detail |
|---|---|
| **+ Date of Birth** | `<input id="pf_dob" type="date">`. On change, show derived age (`ageFromDob`) as a hint; keep the numeric `age` input as a manual fallback or make it read‑only/computed. |
| **+ Contract Expiry Date** | `<input id="pf_contractExpiry" type="date">`, placed next to the existing *Тип контракта*/*Агент* row. |
| **Color gradation** | Render a live `contractTier()` chip beside the expiry input and a colored left‑border on the player card (Block 2), mirroring the Scout block's `border-left:3px solid <tier.color>` pattern (line 614). |
| **− Remove Status** | Delete the *Статус* `<select id="pf_status">` (line 843) from the form, and the status pill from the list (line 778) and modal header (line 788). |

**Markup (new rows in `edit()`):**

```html
<div class="row" style="margin-top:10px">
  <div style="flex:1"><label class="fld">Дата рождения</label>
    <input id="pf_dob" type="date" value="${esc(p.dob||'')}"
           onchange="document.getElementById('pf_ageHint').textContent =
                     (ageFromDob(this.value)??'—')+' лет'"></div>
  <div style="flex:1"><label class="fld">Возраст</label>
    <div class="pill" id="pf_ageHint">${ageFromDob(p.dob)??p.age??'—'} лет</div></div>
</div>
<div class="row" style="margin-top:10px">
  <div style="flex:1"><label class="fld">Тип контракта</label><select id="pf_contract">…</select></div>
  <div style="flex:1"><label class="fld">Контракт до</label>
    <input id="pf_contractExpiry" type="date" value="${esc(p.contractExpiry||'')}"></div>
</div>
```

**`save()`** — add `dob:v('dob')` and `contractExpiry:v('contractExpiry')` to the `data` object;
**stop writing** `status` from this form (line 857). `data.name`, `eval`, `skills` unchanged.

> **⚠ Invariant guard (§2.6.2).** `status` is **owned by Medical** (`syncStatus`) and **read by the
> roster pool filter** (`p.status!=='injured'…`, line 1335). Removing it from the *profile UI* is correct;
> **deleting the field** would break Medical color‑coding and the lineup pool. Keep the property; only
> drop its manual editor and its profile‑side display.

**Visual cues:** expiry chip uses `contractTier().color` as background tint (`color` + 12 % alpha),
text = `tier.label`. Expired/urgent tiers draw attention (red); long contracts read green.

---

## 2. Profile View Block — card grid + expandable 7‑tab view

**File:** `CRM.modules.profiles.render()` (list, line 765) and `open()` (modal, line 781).

**2.1 Replace the table with a responsive card grid.** Reuse the Scouting list pattern
(`paneList`, line 613): `<div class="grid g3">` of `.card` items. Each card:

- number badge (reuse the 48 px accent square, line 786) · **name** · `POS_FULL` · age (`ageFromDob`);
- **contract‑expiry color**: `border-left:3px solid ${contractTier(p.contractExpiry).color}` + a tier chip;
- two compact KPIs from `matches.playerAgg(p.id)` (GP, Pts);
- `role="button"`, `tabindex="0"`, `aria-label`, `onclick=open(id)` (no Status pill).

**2.2 Expandable view = the existing `CRM.modal` with a 7‑tab `.tabsrow`.** Replace the current
3 tabs (overview/matches/med) with:

| Tab key | Label | Source |
|---|---|---|
| `bio` | **BIO** | creation fields: DOB+age, height/weight, shot, contract type + **expiry chip**, agent, note |
| `stats` | **Statistics** | `matches.playerAgg(id)` — GP, G, A, Pts, +/−, avg rating |
| `matches` | **Matches** | `matches.profilePane(id)` — per‑match list with individual ratings |
| `medical` | **Medical** | `medical[]` filtered by `playerId` (date · diagnosis · status) |
| `physical` | **Physical Tests** | `physical[]` filtered by `playerId` (date · per‑norm results) |
| `psycho` | **Psychoprofile** | `psycho[]` filtered by `playerId` (scores, validity, note) |
| `scout` | **Scout Evaluation** | `evalScore(p)` radar + per‑category `.bar` rows (reuse `_drawReportRadar`) |

**Constraints met:** tabs **load dynamically** — keep the established pattern of re‑opening with a tab
param (`open(id, tab)`); each branch builds only its own panel, and chart/radar canvases are drawn
**after** injection (as `open()` already does at line 830). Mark up as
`role="tablist"` / `role="tab"` (`aria-selected`) / `role="tabpanel"` for accessibility.

**Style / responsive:** `.grid.g3` collapses to 1 column under ~720 px; `.tabsrow` gets
`overflow-x:auto` so all 7 tabs remain reachable on mobile; content uses existing `.card`/`.kpi`/`.evrow`.

---

## 3. Scout Block — "Estimated Composition" tab

**File:** `CRM.modules.scouting.render()` nav (line 599).

Add a 5th tab: `['estimated','Предполагаемый состав']`, dispatching to a new `paneEstimated()`.

**Interface.** A formation board that assigns **candidates (`scoutDB`)** — not team players — to
positions, reusing the roster board mechanics (`.rslot` slots + `.pchip` pool, drag‑drop **and**
click‑to‑place, position eligibility). Persist to a **new, isolated** store key
`scoutLineup:{<slot>:candidateId}` so it never touches the real `roster.lines`
(keeps Scouting↔Roster bridge limited to `toTeam`, invariant §2.6.3).

- **Board:** forwards 4×(L/C/R), defense 4×(L/R), goalies — same `ROSTER_SLOTS` geometry.
- **Pool:** candidates grouped F / D / G, sorted by `evalScore().overall`, showing `gradeChip`.
- **Eligibility:** G→G only, D→D only, F‑slot→any non‑G (reuse `_eligible` logic, invariant §2.6.6).
- Handlers mirror `roster.slotDrop/slotClick/poolClick` but read `scoutDB` and write `scoutLineup`.

> **Reference screenshot:** *not supplied in the provided materials.* The layout is designed to match
> the existing roster board + a standard hockey formation; **confirm against the user's screenshot**
> before finalizing element placement.

**Output spec (placement):** left = formation board (two cards: Нападение / Защита+Вратари, as in
`paneLineup`, line 1328); below = candidate pool card grouped by position; interaction = drag a chip
onto a slot, or click chip then click a highlighted slot; `×` returns a candidate to the pool.

---

## 4. Composition Block — lineup like "Match Day"

**File:** `CRM.modules.roster.paneLineup()` (line 1328) / new presentation view.

Render the lineup using the **Match‑Day** presentation (`matches._lineupMini`, line 1078): the `.ice`
rink container with **НАПАДЕНИЕ / ЗАЩИТА / ВРАТАРИ** sections, `.slot` cells showing position label +
surname, goalies as `.pill.accent`. The roster is **sorted by role/position** by construction
(forward lines → defense pairs → goalies; L/C/R and L/R order from `ROSTER_SLOTS`).

- Add a segmented toggle on the *Состав* tab: **Edit** (current interactive `.rslot` board) vs.
  **Match Day** (read‑only `.ice` presentation) — so coaches build in one and present in the other.
- The Match‑Day view replicates the reference design already used in the Matches block (no new
  styling needed; reuse `.ice` / `.slot`).

---

## 5. Integration & invariants checklist

1. **Add‑only migration** for `dob` / `contractExpiry`; never overwrite user data (§2.7).
2. **Keep `status`** as a field (Medical‑owned, §2.6.2; roster pool reads it, line 1335) — remove only
   its profile UI.
3. **`scoutLineup` is separate** from `roster.lines`; candidate→team promotion stays via `toTeam` (§2.6.3).
4. All new strings through `esc()`; all dates `YYYY-MM-DD` via `parseLocalDate`/`normYmd` (§2.6.5).
5. Events via inline attributes only; dialogs via `CRM.ui` (§2.7) — no native `confirm/prompt`.
6. New tabs follow the in‑block `params.view` convention (`CRM.router.go('<id>',{view:'…'})`).
7. After edits: `node --check` the extracted script and re‑render all 15 tabs (handoff §6).

---

## 6. Deliverable mockup

`crm_enhancements_mockup.html` is self‑contained (inline CSS/JS, **no external deps** — the Scout
radar is drawn as inline SVG), seeded with demo data, and demonstrates all four blocks:
the profile **card grid**, the **7‑tab** expandable profile, the Scout **Estimated Composition**
drag‑drop board, and the **Match‑Day** composition view. It uses semantic landmarks
(`<nav>/<main>/<section>`), ARIA tab roles, click‑to‑place as a keyboard‑friendly alternative to
drag‑drop, and mobile breakpoints. Class names and CSS tokens mirror `crm_hockey.html` so markup can
be lifted into the modules above with minimal change.
