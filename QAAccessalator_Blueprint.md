# QA Accessalator — App Blueprint

Paste this entire document into Claude to recreate an exact replica of the **QA Accessalator** React application.

---

## Overview

| Property | Value |
|---|---|
| App Name | QA Accessalator |
| Framework | React (single `.jsx` file, no external dependencies beyond Google Fonts) |
| Target | Claude.ai Artifact (React) |
| Theme | Dark — Navy/Charcoal bg, Amber/Gold accent |
| Fonts | Syne (headings) + DM Mono (body/code) via Google Fonts |

---

## App Structure

```
App
├── Header (sticky, always visible)
├── HomeScreen        ← default screen
├── BookOfWork        ← screen: "bow"
├── InProgressScreen  ← screen: "ia"  (IA Reports)
└── InProgressScreen  ← screen: "qa"  (QA Reports)
```

Screen routing is handled via a single `useState("home")` — no router library needed.

---

## Design Tokens (CSS Variables)

```css
--bg:       #0d1117      /* page background */
--surface:  #161b22      /* card / header background */
--panel:    #1c2330      /* table header rows */
--border:   #30363d      /* all borders */
--accent:   #f0a500      /* primary amber accent */
--accent2:  #e05c00      /* gradient end for progress bar */
--text:     #e6edf3      /* primary text */
--muted:    #8b949e      /* secondary / placeholder text */
--success:  #3fb950
--danger:   #f85149
--radius:   8px
--font-h:   'Syne', sans-serif
--font-b:   'DM Mono', monospace
```

Import fonts at top of `<style>` block:
```css
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Mono:wght@300;400;500&display=swap');
```

---

## Component Specifications

### 1. `<Header>`

- **Position**: sticky top, height 56px, `z-index: 100`
- **Background**: `var(--surface)`, bottom border `var(--border)`
- **Left**: Logo — `"QA "` + `<span color=accent>"Accessalator"</span>` + badge `"v1.0"` (amber bg, black text, 10px bold uppercase)
- **Right** (hidden on home): Breadcrumb — `"Home › {Page Name}"`, active crumb in amber
- Props: `screen` (string), `onBack` (fn)

---

### 2. `<HomeScreen>`

- Centered column, max-width 720px
- **Title**: `"Quality Assurance Accessalator"` — "Assurance" on second line; "Assurance" text in amber accent; font Syne 42px 800 weight
- **Subtitle**: `"SELECT A MODULE TO GET STARTED"` — muted, 13px, 0.5px letter-spacing
- **3-column nav grid** (`grid-template-columns: repeat(3,1fr)`, gap 20px)

Each card:
```
background: var(--surface)
border: 1px solid var(--border)
border-radius: 8px
padding: 28px 24px
cursor: pointer
hover: border → accent, translateY(-3px), amber box-shadow
hover::before: 3px top amber bar slides in (scaleX 0→1)
```

| Card | Icon | Title | Description |
|---|---|---|---|
| bow | 📋 | Book of Work | Track colleagues, allocations and project assignments in an editable weekly table. |
| ia  | 🔍 | IA Reports   | Internal audit report views and analysis summaries. |
| qa  | ✅ | QA Reports   | Quality assurance report dashboard and metrics. |

Arrow `↗` positioned absolute bottom-right, turns amber on hover.

---

### 3. `<InProgressScreen>`

Used for both **IA Reports** and **QA Reports**.

- Centered, max-width 520px, margin-top 80px, text-align center
- Large icon (64px emoji)
- Title: Syne 28px 700
- Subtitle: muted, 13px, line-height 1.7 — `"This module is currently under development. Check back soon for updates."`
- **Progress bar**: 300px wide, 6px tall, amber→accent2 gradient, pulsing opacity animation (2s infinite)
- Back button: transparent, border `var(--border)`, hover border+text amber

Props: `title` (string), `icon` (emoji string), `onBack` (fn)

---

### 4. `<BookOfWork>`

#### State
```js
rows       // array of row objects
extraCols  // array of { key, label } for user-added columns
nextId     // incrementing row id counter, starts at 4
```

#### Row object shape
```js
{
  id: number,
  colleague: "",
  brid: "",
  location: "",   // county
  jobType: "",
  role: "",
  grade: "",
  project: "",    // allocation week → project
  allocation: "", // allocation week → units (%)
  extra: {}       // keys match extraCols[].key
}
```

#### Fixed Base Columns (in order)

| key | Label |
|---|---|
| colleague | Colleague |
| brid | BRID |
| location | Location (County) |
| jobType | Job Type |
| role | Role |
| grade | Grade |

#### Allocation Week Columns (always after base cols)

Two columns grouped visually with `border-top: 2px solid var(--accent)` and a subtle amber background tint:

| key | Label | Sub-label |
|---|---|---|
| project | Project | `Allocation wk {weekLabel}` |
| allocation | Allocation % | `Units this week` |

`weekLabel` = Mon–Fri of current week, formatted as `D/M–D/M` (computed at render time).

#### Extra Columns

- Added by "+ Add Column" button → appends `{ key: "col_{timestamp}", label: "New Column" }` to `extraCols`
- Column header renders as an `<input>` (editable label) + `✕` delete button
- Deleting a column removes it from `extraCols` and purges its key from all row `extra` objects

#### Delete Row column

Final column in every row — `✕` button, `btn-danger` style, calls `removeRow(id)`

#### Header Bar (above table)

Left: `"📋 Book of Work"` (Syne 24px 700)

Right buttons:
- `"+ Add Column"` → `btn-secondary`
- `"+ Add Row"` → `btn-primary` (amber fill, black text)
- `"← Home"` → `btn-secondary`

#### Stats Bar (3 chips between header and table)

```
[ {rows.length}   Colleagues ]  [ {weekLabel}   Current Week ]  [ {totalCols}   Columns ]
```
Each chip: `var(--surface)` bg, border, padding 10px 16px. Value in Syne 18px, label in muted 11px.

#### Table Styling

- Sticky `<thead>`
- `overflow: auto`, `max-height: calc(100vh - 220px)` on container
- Row number column: 36px wide, `var(--panel)` bg, muted 10px text
- Column headers: Syne 11px 600 uppercase, muted color, letter-spacing 0.5px
- `<input>` cells: transparent bg, no border, DM Mono 12px; on focus → amber `box-shadow: inset 0 0 0 1px var(--accent)` + `rgba(240,165,0,0.08)` bg
- Row hover: `rgba(240,165,0,0.04)` tint on all cells
- `border-collapse: collapse`; every `td`/`th` has right + bottom border (`var(--border)`); last-child and last-row borders removed

---

## Button Classes

| Class | Style |
|---|---|
| `.btn-primary` | Amber bg `#f0a500`, black text, font-weight 600; hover darkens to `#d49200` |
| `.btn-secondary` | Transparent bg, `var(--border)` border, white text; hover → amber border + text |
| `.btn-danger` | Transparent, muted border+text; hover → `var(--danger)` red border + text |

All buttons: flex, align-items center, gap 6px, DM Mono 12px, 8px border-radius, transition 0.15s.

---

## Routing Logic

```jsx
// Single state in App root
const [screen, setScreen] = useState("home");

// Rendered conditionally
{screen === "home" && <HomeScreen navigate={setScreen} />}
{screen === "bow"  && <BookOfWork onBack={() => setScreen("home")} />}
{screen === "ia"   && <InProgressScreen title="IA Reports" icon="🔍" onBack={() => setScreen("home")} />}
{screen === "qa"   && <InProgressScreen title="QA Reports" icon="✅" onBack={() => setScreen("home")} />}
```

---

## Helper Function

```js
const getWeekLabel = () => {
  const now = new Date();
  const start = new Date(now);
  start.setDate(now.getDate() - now.getDay() + 1); // Monday
  const end = new Date(start);
  end.setDate(start.getDate() + 4); // Friday
  const fmt = (d) => `${d.getDate()}/${d.getMonth() + 1}`;
  return `${fmt(start)}–${fmt(end)}`;
};
```

---

## Initial Data

Start with **3 empty rows** (id 1, 2, 3) and `nextId = 4`. No pre-filled data.

---

## Animations

| Element | Animation |
|---|---|
| Nav card hover | `translateY(-3px)` + amber box-shadow + top bar `scaleX(0→1)` |
| Progress bar | `pulse-bar` — opacity 1→0.5→1, 2s infinite ease-in-out |
| All buttons | `transition: all 0.15s` for color/border changes |
| Card `::before` bar | `transition: transform 0.25s ease` |

---

## File Format

- Single `.jsx` file
- All CSS in a JS template literal `const css = \`...\`` injected via `<style>{css}</style>` inside the root component's JSX
- All styles use plain class names (no CSS modules, no Tailwind)
- Default export: `export default function App()`
- Hooks used: `useState`, `useCallback`

---

*End of blueprint. Paste into Claude → React Artifact to reproduce.*
