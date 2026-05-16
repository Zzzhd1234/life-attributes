# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Three standalone, client-side single-page applications — no build step, no package manager, no external JS libraries:

| File | Purpose | Lines |
|------|---------|-------|
| `demo.html` | Life Attributes — iOS-style habit/todo/calendar gamification app | ~5600 |
| `speakup.html` | SpeakUp — English learning: flashcards, lyrics study, AI phrase extraction | ~3100 |
| `web.html` | Earlier Adventure Dashboard prototype (not actively developed) | — |

Deployed via Netlify from the `Zzzhd1234/life-attributes` GitHub repo. Push to `main` → auto-deploy. `netlify.toml` rewrites `/` → `/demo.html` (status 200). **Do not create an `index.html`** — it will shadow the redirect and serve stale content to the phone.

## Running the Apps

```bash
open demo.html      # Life Attributes (mobile)
open speakup.html   # SpeakUp English learning
```

No server required. For AI features in SpeakUp, a Claude API key must be saved in the app UI — stored in `localStorage.speakup_apikey`.

## Syntax check (run after every JS change)

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('demo.html', 'utf8');
const s = html.match(/<script>([\s\S]*?)<\/script>/g);
if (s) { new Function(s.map(x => x.replace(/<\/?script>/g,'')).join('\n')); console.log('OK'); }
"
```

Then: `git add … && git commit && git push origin main`

## Architecture: demo.html

Single file, iOS-inspired mobile-first UI (`max-width: 390px`). Navigation via `switchTab(name, el)` toggling `<div id="page-*">` and `.tab-item.active`.

**Pages/tabs:** `profile` (个人), `habits` (习惯), `calendar` (日历), `records` (记录), `report` (报告)

**Data layer:**
- `state` object — loaded via `load()`, saved via `save()`
- Key: `localStorage.lifeapp_v2`
- Shape defined by `DEFAULTS`:
  ```
  abilities: {颜值,魅力,智商,体质,专注力,认知}  // each 0–100
  habits: { key: {name,icon,desc,checkedToday,history[],weekCount,...} }
  todos: [{id,text,deadline,deadlineTime,recurrence,label,done,subtasks[],completedAt,isGroup}]
  reminders: [{id,title,datetime,endTime,location,remind,note,done}]
  timeLogs: [{id,title,date,startTime,endTime,note}]
  transactions: [{id,date,type,amount,note}]
  sleepHistory, wealth, streak, avatarSeed, avatarStyle, ...
  ```
- **Subtask shape:** text subtask `{id,text,done}`, video subtask `{id,type:'video',title,duration,progress,url}`
- **Reading books shape:** stored on `state.habits.reading.books[]` as `{id,title,currentPage,totalPages,cover?,description?,author?}`
- **Date handling:** always use `localDateStr(d)` (not `d.toISOString().slice(0,10)`) — `toISOString()` returns UTC and causes off-by-one errors in UTC+ timezones. `TODAY` is set via `localDateStr(new Date())`.

**Key subsystems:**

| Subsystem | Key functions |
|-----------|--------------|
| Habits | `toggleHabit(key)` — checks in, updates `history[]`, applies ability nudges via `habitFX()` |
| Todos | `addTodo()`, `toggleTodo(id)`, `renderTodos()` — supports recurrence, subtasks, labels |
| Group todos | `isGroup:true` todos have mixed subtasks (text + video). `_checkGroupCompletion(t)` — video done when `progress >= duration`. Subtask drag-to-reorder via `_startSubDrag()` with `data-sub-id` on each row. |
| Recurring todos | `nextRecurDate(fromDate, recurrence)` — computes next occurrence. `load()` resets done recurring todos on new day (no entity spawning). |
| Calendar | `renderCalendarPage()`, `openDayDetail(dateStr)` — aggregates all data types per day |
| Today view | `renderCalTodayTodos()` — shows todos + timeLogs + reminders for a given day. `calTodayOffset` (−1/0/1) controls yesterday/today/tomorrow. Swipe handled by `initTodaySwipe()` / `slideTodayTo()`. |
| Reminders | `saveReminder()`, `openReminderAddSheet(id?)` — handles both create and edit. `scheduleOneReminder()`, `checkAndNotify()` use browser Notification API. |
| Time logs | `openTimeLogSheet(dateStr)`, `addTimeLog()` — start/end time entries |
| Report | `renderReport()` — monthly ability/habit/finance summary |
| Settings | `openSettings()` — name, age, avatar, font size, accent color, data export/reset |
| Ability scoring | `nudge(attr, delta)`, `habitFX(key)`, `mergeTagEffects(tags)` — tags map to ability deltas |
| Avatar | `drawAvatar()` — DiceBear API (`dylan` style). `state.avatarSeed` picks the character from `DYLAN_SEEDS[]`. `state.avatarColor` sets the accent ring color. `renderAvatarPicker()` / `selectAvatarSeed(seed)` for the picker sheet. |
| Reading books | `renderReadingBooks()`, `saveBook()`, `openReadingProgress(id)`. Adding a book auto-searches Open Library API (`searchBooksAPI(query)`) by title to fill cover/author/pages. `openReadingEdit(id)` is both add and edit. |

**Sheet/modal pattern:** All modal panels use `toggleSheet(sheetId, overlayId, open)` — adds/removes `.open` class.

**Full-page slides:** `page-todo-add`, `page-todo-detail`, `page-reading` are `.todo-fullpage` elements that slide in via `transform: translateX`. `initSwipeBack()` adds swipe-right-to-close on all three.

**PWA:** `sw.js` registers a network-first service worker caching `demo.html` and `manifest.json`. Bump `CACHE` version string in `sw.js` whenever `demo.html` changes significantly, to force reinstall on devices.

## Architecture: speakup.html

Single file, all HTML/CSS/JS inline. Navigation via `switchPage(name)` toggling `<div id="page-*">` sections, with `.nav-tab` buttons calling it.

**Pages:** `home`, `phrases`, `flashcard`, `scenarios`, `tips`, `artists`, `add`

**Data layer:**
- `data` object (global) — loaded once via `loadData()`, saved via `saveData(data)`
- Key: `localStorage.speakup_data` — shape: `{ phrases: [{id, phrase, meaning, example, exampleSource, tags[], review:{stage,correct,wrong}}] }`
- Lyrics session persisted separately in `localStorage.speakup_lyrics_session` — shape: `{ track, lines, marked[], chat[], chatOpen }`
- Saved songs in `localStorage.speakup_songs`

**Key subsystems:**

| Subsystem | Key functions |
|-----------|--------------|
| Flashcard | `initFlash()`, `showCard()`, `flipCard()`, `answer(correct)` — 6-stage spaced repetition |
| AI autofill | `aiAutofill()` — fills meaning/example for a phrase via Claude Haiku |
| AI extract | `extractWithAI()` — extracts phrases from SRT/lyrics text via Claude Haiku |
| Lyrics search | `searchLyrics()` — fetches from `lrclib.net` API (no key needed) |
| Lyrics marking | `markSelectedPhrase()`, `applyMarks()`, `explainMarked()` — select text → highlight → AI explain |
| Inline AI chat | `sendLyricsChat()` — multi-turn chat with song/lyrics/marks as system context |
| Artists page | Static `ARTISTS[]` array + `renderArtistList()` / `openArtist()` |
| Saved songs | `saveSong()`, `getSavedSongs()`, `openSavedSong()` — appear in 艺人库 |
| TTS | `speakText(text)` — Web Speech API |

**All Claude API calls** use the same pattern: `fetch("https://api.anthropic.com/v1/messages")` with header `"anthropic-dangerous-direct-browser-access": "true"`, model `claude-haiku-4-5-20251001`, key from `getApiKey()`.

## Editing Rules

- **Large files (>300 lines):** always `grep -n` to find line numbers before reading. Use `Read` with `offset`+`limit`. Never read the whole file to find one function.
- **Edits:** keep `old_string`+`new_string` combined under ~150 lines to avoid context bloat. Split if needed.
- **Error handling:** see `.claude/rules/error-handling.md` — guard clauses, `showToast` for validation, `confirm()` for destructive actions, no `try/catch` outside `load()`.
- **Optional sections:** see `.claude/rules/optional-sections.md` — always use `.toggle-switch` + `.recur-check-row` pattern; toggle class is `.on`; collapsing clears inputs.
- **Touch events:** see `.claude/rules/touch-events.md` — never `e.preventDefault()` on container touchstart; use `touch-action:none` on drag handles; gate swipe-back handlers on drag state variable; no `willChange:transform` inside composited pages (causes GPU layer conflict with `position:fixed` ghost).
- **No auto-focus:** never call `.focus()` or `.select()` on sheet open or after adding items. Users tap to focus.
- **No comments** unless the why is non-obvious. No docstrings.
- **Dates:** always use `localDateStr(d)` — never `d.toISOString().slice(0,10)`.
