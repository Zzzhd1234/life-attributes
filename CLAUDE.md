# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Two standalone, client-side single-page applications with no build step, no package manager, and no external JS libraries:

- **SpeakUp** ([index.html](index.html)) — English language learning platform with spaced repetition flashcards, phrase management, and AI-assisted extraction from video subtitles
- **Life Attributes Demo** ([demo.html](demo.html)) — Personal life gamification app tracking habits, abilities, exercise, sleep, and finances; inspired by Japanese "人生属性" games

## Running the Apps

Open HTML files directly in a browser — no server required:

```bash
open index.html   # SpeakUp
open demo.html    # Life Attributes
```

For the AI extraction feature in SpeakUp, a Claude API key is needed. It is stored client-side in `localStorage.speakup_apikey`.

## Architecture

### SpeakUp (index.html, ~1450 lines)

Single-file app with all HTML, CSS, and JS inline. Pages are `<div id="page-*">` elements toggled via `showPage()`.

**Data layer:**
- `data.phrases[]` stored in `localStorage.speakup_data` — each phrase has `id`, `phrase`, `meaning`, `example`, `tags[]`, and `review` (spaced repetition stage 0–5)
- `loadData()` / `saveData()` are the only persistence functions

**Key features:**
- `extractWithAI()` — calls `https://api.anthropic.com/v1/messages` (Claude Haiku) to extract phrases from SRT subtitle text pasted by the user
- `flipCard()` / `answer()` — spaced repetition flashcard logic across 6 stages
- `speakText()` — Web Speech API text-to-speech
- `parseSRT()` — client-side SRT subtitle parser
- YouTube video embedding with timestamp binding on the Artists page

### Life Attributes Demo (demo.html, ~1960 lines)

Single-file app with iOS-inspired mobile-first UI (max-width 390px). Pages toggled via `showPage()`.

**Data layer:**
- `state` object persisted in `localStorage.lifeAttributes` — includes `abilities` (5 stats), `habits[]`, `transactions[]`, `sleep`, `exercise`

**Key features:**
- `drawAvatar()` — renders a pixel-art avatar on `<canvas>` using `state.avatarColor`
- Habit tags (e.g. "运动", "学习") map to ability score increases; streak multipliers apply consecutive-day bonuses
- Monthly financial tracking with income/expense ledger in `state.transactions[]`
- Settings sheet allows name, age, avatar color customization plus full data export/reset

### Data File

[english_learning.json](english_learning.json) is a sample phrase dataset importable into SpeakUp via the JSON batch import feature.
