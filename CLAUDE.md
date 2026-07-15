# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static practice-question web app for Claude AI Architect certification prep ("rewrite the questions with your own words"). There is no build system, package manager, or test suite — `index.html` is a self-contained HTML/CSS/JS document that loads question data from `questions.json`.

## Running locally

`index.html` fetches `questions.json` via `fetch()`, which browsers block under the `file://` origin — you must serve the directory over HTTP, e.g. `python3 -m http.server` and open `http://localhost:8000/index.html`. There is no build/compile/lint/test step.

## Deployment

Pushing to `main` triggers `.github/workflows/static.yml`, which deploys the entire repository as-is to GitHub Pages (no build step in CI either — it just uploads the repo contents).

## Architecture

- **`questions.json`** is the question bank: an array of question objects (`id`, `domain`, `scenario`, optional `constraints`, `question`, `options` (`key`/`text` pairs), `correct` (the right option's key), `correctReason`, and `whyFail` (a reason per incorrect key)). `scenario`/`constraints` values are injected via `innerHTML` (safe here since content is authored, not user-supplied), so they can include inline `<em>` for emphasis.
- **`index.html`** renders one question at a time — scenario, constraints, options, free-text answer box, hand-drawn whiteboard (HTML canvas), and answer key — driven entirely by JS against the fetched question array. All styling is inline `<style>` in the `<head>`; all behavior is inline `<script>` at the end of the body — no external assets or frameworks.
- **Question selection**: a dropdown (`#questionSelect`) lists all questions; selecting one calls `loadQuestion(id)`, which re-renders the scenario/options/reveal panel and reloads that question's saved state.
- **Option shuffling**: `shuffleOptions()` runs a Fisher-Yates shuffle over a question's options on every render and reassigns display labels A-D, so the correct answer's on-screen position changes each time a question is viewed. Grading and the answer-key text are matched by each option's stable `key` from the JSON (via `currentShuffled`), not by its display label, so shuffling never breaks correctness checking.
- **Grading**: clicking an option selects it (`pickedKey`); "Check answer" compares it to `current.correct`, colors the picked option green/red and (if wrong) dashes the correct option, and persists the result.
- **Persistence** uses browser cookies via local `setCookie`/`getCookie` helpers (365-day expiry), not `localStorage` — chosen deliberately so state survives a plain page refresh without any backing service. All keys are namespaced per question ID: `${id}_answer` (free-text answer), `${id}_board` (whiteboard shapes JSON), `${id}_selected` (chosen option key), `${id}_result` (`correct`/`incorrect`). Cookies cap out around 4KB, so a very large whiteboard sketch could eventually hit that limit.
- **Gap analysis & wrong-only mode**: `computeStats()` reads the `_result` cookie for every question to build progress (`updateProgress()`, the bar under the title) and a per-question breakdown (`renderGapAnalysis()`, the collapsible panel). The "Focus: wrong only" toggle filters the dropdown to questions currently marked `incorrect`; a question drops out of that filter the moment it's answered correctly.
- The whiteboard stores shapes as plain JS objects (`{type: 'rect'|'arrow'|'text', ...coords, label}`) in an array, serialized to JSON for persistence, and redrawn from scratch on every change via a single `render()` function — there's no incremental canvas diffing.

## Adding new questions

Append an object to `questions.json` following the existing shape (see any entry for the exact fields). No changes to `index.html` are needed — the dropdown, shuffling, grading, and gap analysis all derive from the array length and contents automatically.
