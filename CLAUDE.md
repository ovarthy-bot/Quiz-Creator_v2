# Quiz Creator v2 — CLAUDE.md

## Project Overview

A single-page web application for creating and managing quiz questions. Users import questions from Excel files, store them in Firebase Firestore, and take quizzes with auto-generated wrong answer choices.

**Tech stack:** Vanilla HTML/CSS/JS (no build system), Firebase Firestore, Firebase Auth (anonymous), SheetJS (XLSX parsing).

## File Structure

```
Quiz-Creator_v2/
└── index.html   # Entire application — CSS, HTML, and JS in one file
```

There is no build step, no package.json, no node_modules. The app runs directly in the browser.

## Architecture

Everything lives in `index.html`, structured as:

1. **`<head>`** — External CDN scripts (SheetJS XLSX library, Google Fonts)
2. **`<style>`** — CSS custom properties (design tokens) and all component styles
3. **`<body>`** — HTML structure: top nav, three page sections (Admin, Bildiklerim, Quiz)
4. **`<script type="module">`** (at bottom) — All application logic

### Pages / Tabs

| Tab ID | Page Element | Purpose |
|--------|-------------|---------|
| `admin` | `#pageAdmin` | Excel import, question list management |
| `biliyorum` | `#pageBiliyorum` | Questions marked as "known" (Bildiklerim) |
| `quiz` | `#pageQuiz` | Quiz setup and quiz taking interface |

### Firebase / Firestore

- **Project:** `quiz-creator-d9aa5`
- **Collection:** `questions`
- **Auth:** Anonymous (`signInAnonymously`) — required before any Firestore operation
- **SDK version:** `10.12.1` (loaded from CDN)

Each document in the `questions` collection has:
```js
{
  question: string,       // Question text
  correctAnswer: string,  // Correct answer text
  known: boolean          // true = moved to "Bildiklerim"
}
```

Wrong/distractor answers are NOT stored — they are generated dynamically at quiz time by shuffling other questions' `correctAnswer` values.

### Key JS Functions

| Function | Description |
|----------|-------------|
| `loadQuestions()` | Fetches all docs from Firestore, splits into `unknownQuestions` / `knownQuestions` |
| `confirmImport()` | Saves `pendingImport` array to Firestore via `writeBatch` |
| `saveQuestion(data)` | `addDoc` to Firestore |
| `updateQuestion(id, data)` | `updateDoc` |
| `deleteQuestion(id)` | `deleteDoc` |
| `deleteAllQuestions()` | `writeBatch` delete all |
| `markAsKnown(id)` / `unmarkAsKnown(id)` | Updates `known` field |
| `handleExcelFile(file)` | Reads Excel with SheetJS, calls `parseRows()` |
| `parseRows(rows)` | Builds `pendingImport` array, shows preview |
| `renderQuizSetup()` | Quiz configuration UI |
| `renderQuizQuestion()` | Renders current quiz question with auto-generated distractors |
| `switchTab(name)` | Shows/hides page sections |
| `showToast(msg, type)` | Toast notification (`success`, `error`, `warning`, `''`) |
| `setFirebaseStatus(state)` | Updates connection indicator in nav (`ok`, `error`, `loading`) |

### State Variables

```js
let allQuestions = [];       // All Firestore docs
let unknownQuestions = [];   // allQuestions where known !== true
let knownQuestions = [];     // allQuestions where known === true
let filteredQuestions = [];  // Search-filtered unknownQuestions
let filteredKnown = [];      // Search-filtered knownQuestions
let pendingImport = [];      // Questions parsed from Excel, not yet saved
let quizState = {};          // Active quiz session state
```

## Development Workflow

1. **Edit** `index.html` directly — no build needed
2. **Test** by opening in browser (or using a local HTTP server)
3. **Firebase rules** must allow authenticated reads/writes. The app signs in anonymously on load, so Firestore rules should be:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
4. **Anonymous Auth** must be enabled in Firebase Console → Authentication → Sign-in method

## Excel Import Format

| Column A | Column B |
|----------|----------|
| Question text | Correct answer |

- First row is skipped if column A contains "soru", "question", or "a" (header detection)
- Rows missing question or answer are skipped
- Wrong answers are generated dynamically — do NOT add them to the Excel file

## CSS Design Tokens (CSS Custom Properties)

Defined in `:root`:
- `--primary: #4f46e5` — indigo (main brand color)
- `--success: #10b981`, `--danger: #ef4444`, `--warning: #f59e0b`
- `--bg: #f8fafc`, `--surface: #ffffff`, `--surface2: #f1f5f9`
- `--radius: 12px`, `--radius-sm: 8px`
- `--shadow`, `--shadow-lg`, `--transition`

## Common Pitfalls

- **"Missing or insufficient permissions"** — Firebase Auth not initialized or anonymous sign-in not enabled in Firebase Console. The app now calls `signInAnonymously(auth)` before `loadQuestions()`.
- **Quiz distractors not working** — Need at least 4 questions total in the pool to generate 3 wrong answers.
- **Excel not parsing** — File must have question in column A and answer in column B. Header row is auto-detected and skipped.
- **All JS is in one `<script type="module">`** — Functions exposed to `onclick` handlers must be attached to `window` (e.g., `window.confirmImport = async function() {...}`).

## Git Branch Convention

Feature branches follow: `claude/<description>-<id>`

Current fix branch: `claude/fix-firebase-permissions-dEgYp`
