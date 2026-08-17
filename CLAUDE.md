# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

이 저장소(OverTime_App, "초과근무 기록부")는 지방공무원 초과근무 시간과 예상 수당을 계산해주는 단일 페이지 웹앱입니다. 전체 애플리케이션이 `index.html` 한 파일(HTML + CSS + JS, 인라인)로 구현되어 있으며, 빌드 시스템, 패키지 매니저, 서버, 테스트 프레임워크가 전혀 없습니다.

## Development workflow

- No build step, no dependencies, no package.json. `index.html` is opened directly in a browser.
- To preview locally, just open the file in a browser, or serve the directory with any static file server, e.g. `python3 -m http.server` from the repo root, then visit `http://localhost:8000/index.html`.
- There is no linter, formatter, or test suite configured — verify changes manually by loading the page and exercising the UI (add a record, delete a record, change the hourly rate, check the gauge/pay totals update).
- All persistence is client-side via `localStorage` (see Architecture below) — there is no backend.

## Architecture

Everything lives in `index.html`, split into three inline sections:

1. **`<style>`** — CSS custom properties define the color palette (`--navy`, `--cream`, `--accent`, etc.) at `:root`. Layout is a single-column mobile-first card list (`.wrap` capped at `max-width:520px`).
2. **HTML body** — Four cards inside `.wrap`:
   - 기록 추가 (add-record form: date, type `야근`/`조출`, start/end time)
   - 이번 달 인정시간 (monthly recognized-hours gauge)
   - 이번 달 예상 수당 (estimated pay breakdown)
   - 기록 목록 (records table with delete buttons)
3. **`<script>`** — All app logic, no modules/bundler, everything in global scope.

### Data model and persistence

- Records are plain objects `{ id, date, type, start, end }`, stored as a JSON array in `localStorage` under key `overtime_records_v1` (`STORAGE_KEY`).
- The hourly pay rate is stored separately in `localStorage` under `overtime_hourly_rate_v1` (`RATE_KEY`), defaulting to `10949`.
- `loadRecords()`/`saveRecords()` and `loadRate()`/`saveRate()` are the only persistence entry points — any new persisted field should follow this same load/save-to-localStorage pattern.

### Core calculation logic (the part most likely to need changes)

- `workedHours(start, end)` — raw worked hours between two `HH:MM` times, handling midnight rollover.
- `recognizedByDate(list)` — business rule: sums worked hours **per date** (multiple entries on the same day are combined first), then subtracts 1 hour and caps at 4 hours/day. This 1-hour-deduction + 4-hour-daily-cap rule is government overtime policy, not arbitrary — keep it in sync with the `.hint` text in the HTML if changed.
- Monthly aggregation filters recognized hours by `currentMonthKey()` (`YYYY-MM` of today) and sums them, capped for display against `MONTHLY_CAP = 57` hours.
- Pay estimate: `gross = monthTotal * HOURLY_RATE`, `tax = gross * TAX_RATE` (`TAX_RATE = 0.066`), `net = gross - tax`. The formula in the `.hint` text (월급×60%÷209×150%) describes how `HOURLY_RATE` should be derived for a 2026 9급 3호봉 employee, but the app itself takes `HOURLY_RATE` as direct manual input rather than computing it from a base salary.

### Rendering

- `render()` is the single re-render function, called after every mutation (`addRecord`, `removeRecord`, rate input change). There is no diffing/virtual DOM — it rebuilds the table body's `innerHTML` and updates gauge/pay text content directly by element ID.
- No component framework: all DOM updates go through `document.getElementById(...)` and template literals. Follow this existing style rather than introducing a framework or build tooling for small changes.
