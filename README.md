# Legal AI
## 1. Overview

**Re-Volt Legal AI** is a single-file, client-side web application (`index.html`) that acts as a chatbot-style legal assistant for **Bangladesh business law**. It is built for a fictional/branded client, **Re-Volt Beverage Ltd.**, a premium electrolyte-drink company, and frames every legal explanation with practical examples drawn from that company's business.

There is **no backend and no real AI model** — the entire "AI" is a self-contained JavaScript keyword-matching engine running in the browser against a hardcoded legal knowledge base. Everything (UI, logic, data) lives in one `.html` file (~2,270 lines).

- **Tech stack:** Plain HTML5 + CSS3 (custom properties/theming) + vanilla JavaScript. No frameworks, no build step.
- **External dependencies (via CDN):** Google Fonts (Inter, Poppins), Font Awesome 6.5.1 (icons).
- **Browser APIs used:** `localStorage` (persistence), Web Speech API (`SpeechRecognition` for voice input, `speechSynthesis` for text-to-speech), `navigator.share` / Clipboard API, `window.print()`.

---

## 2. Purpose & Framing

The app is positioned as an educational legal-help tool, not a substitute for a lawyer — this disclaimer ("Educational information only. Not legal advice.") appears in the input bar and in exported chats.

The fictional company **Re-Volt Beverage Ltd.** is used as a running case study:
- Founded 2020, Dhaka, Bangladesh
- 500+ distributors, 64 districts, 1M+ customers (per the "About" modal)
- A "Legal Journey Timeline" ties each law to a stage of the company's growth:
  - **2020 Q1** – Partnership Formation → Partnership Act 1932
  - **2020 Q2** – Business Contracts → Contract Act 1872
  - **2020 Q3** – Company Registration → Companies Act 1994
  - **2021** – Sales Expansion → Sale of Goods Act 1930
  - **2022+** – Financial Operations → Negotiable Instruments Act 1881

Every legal answer in the knowledge base includes a "Re-Volt Practical Example" that re-explains the concept using a scenario involving the beverage company (e.g., bounced distributor cheques, mislabeled "sugar-free" drinks, supply contracts with time-is-essence clauses).

---

## 3. Core Features

### 3.1 Chat Interface
- ChatGPT-style layout: left sidebar (navigation/history), main chat panel, right-hand slide-out reference panel.
- Welcome screen with quick-start question buttons and topic shortcuts when no chat is active.
- User messages and AI responses rendered as chat bubbles with avatars, timestamps, and a simulated "Analyzing legal context..." typing indicator (700–1200ms delay) before each reply.

### 3.2 Legal Knowledge Base (the "AI")
- A JavaScript object `KB` contains 5 Bangladesh business-law statutes, each broken into topic entries with:
  - `keys` — an array of keyword/phrase triggers
  - `section` — the relevant statutory section(s)
  - `title`, `answer` (rich HTML explanation), `example` (Re-Volt scenario), `followups` (suggested next questions), `conf` (a confidence score 85–97%)
- **Matching logic (`classify()`):** converts the user's query to lowercase, builds a regex from each keyword (treating internal spaces as wildcards), and scores matches by keyword length + topic confidence. The highest-scoring match wins. If no match is found within the selected category, it falls back to searching all categories; if still nothing, a generic "How Can I Help?" fallback message is shown with links to Google and the Bangladesh Law Portal.
- This is **pattern matching, not a real LLM** — there is no external API call to any AI service.

### 3.3 Legal Topics Covered

| Statute | Category key | Sample topics included |
|---|---|---|
| **Contract Act, 1872** | `contract` | Valid contract essentials, free consent (coercion/undue influence/fraud/misrepresentation/mistake), performance & discharge, quasi-contracts (Sec 68–72), breach & remedies |
| **Sale of Goods Act, 1930** | `sale` | Sale vs. agreement to sell, conditions & warranties, rights of unpaid seller (lien, stoppage in transit, resale), caveat emptor exceptions |
| **Negotiable Instruments Act, 1881** | `negotiable` | Cheque bounce & Section 138 punishment, promissory notes, bills of exchange, endorsement & holder in due course, cheque types |
| **Companies Act, 1994** | `company` | Company registration, AGM/EGM & shareholder rights, types of companies (private/public/OPC/guarantee/unlimited), winding up |
| **Partnership Act, 1932** | `partnership` | Partnership formation & registration, (plus rights/duties/dissolution topics) |

Each topic entry links back to the official Bangladesh law portal (`bdlaws.minlaw.gov.bd`) and to Google/Google Scholar search shortcuts.

### 3.4 Sidebar Navigation
- **Law Categories** filter: "All Topics" or one specific act; filtering narrows which knowledge-base entries `classify()` will search.
- **Company card** ("Re-Volt Beverage Ltd.") opens an "About" modal with company stats and the legal timeline.
- **Chat History**: past questions are saved to `localStorage` (up to 15 sessions) and listed; clicking one re-asks that question. Each entry can be deleted individually.
- **Footer actions**: Export chat (.txt), Print, Clear chat, Share (uses native Share API or copies the URL).

### 3.5 Input & Interaction
- Auto-resizing textarea; `Enter` sends, `Shift+Enter` adds a new line.
- **Quick category pills** above the input to scope the next question to one law.
- **Voice input** via the Web Speech API (`webkitSpeechRecognition`), with language mapped to the UI's selected language.
- **Text-to-speech ("Speak")** button on AI replies, also language-aware.
- Message actions: Copy, Speak, Like/Dislike (client-side only, no backend to record feedback).
- Keyboard shortcut: `Ctrl/Cmd+K` focuses the input box; `Escape` closes open panels.

### 3.6 Multi-language UI
- Language switcher (🇺🇸 English, 🇧🇩 বাংলা, 🇮🇳 हिन्दी, 🇸🇦 العربية) changes the input placeholder text and the speech-recognition/synthesis locale. Note: the legal knowledge-base answers themselves remain in English regardless of language selection — only the placeholder and speech locale change.

### 3.7 Theming
- Light/dark theme toggle, persisted in `localStorage`, using CSS custom properties (`--bg`, `--text`, `--primary`, etc.) redefined under `[data-theme="dark"]`.

### 3.8 Reference Sidebar
- A slide-out panel listing all 5 acts with short descriptions and links to the official law text and a Google search.

### 3.9 Responsive / Print
- Media queries collapse the sidebar into a toggleable off-canvas drawer on mobile (`≤768px`).
- A print stylesheet hides all chrome (sidebar, topbar, input, modals) so only chat content prints.

---

## 4. Data & State Management

All state lives in a single in-memory `state` object plus `localStorage`:

```js
let state = {
  lang: localStorage.getItem('lang') || 'en',
  theme: localStorage.getItem('theme') || 'light',
  cat: 'all',
  history: [],
  sessions: JSON.parse(localStorage.getItem('sessions') || '[]'),
  recording: false,
  recog: null,
  synth: window.speechSynthesis
};
```

- **Persisted:** theme, language, chat session titles (question history).
- **Not persisted:** full chat transcript (`state.history` resets on reload); it only lives for the current page session and can be exported as a `.txt` file before that happens.

There is a `sanitize()` helper that escapes user input into text content before inserting it into the DOM (basic XSS mitigation for the user's own message bubble). AI-generated content is inserted as raw HTML (safe here only because it originates from the hardcoded knowledge base, not from user input).

---

## 5. File Structure

Since this is a single HTML file, the internal structure is:

```
index.html
├── <head>
│   ├── Meta tags, title, Google Fonts, Font Awesome
│   └── <style> — all CSS (theming variables, layout, animations, responsive rules)
└── <body>
    ├── Toast container
    ├── .app
    │   ├── Sidebar (brand, company card, nav categories, history, footer actions)
    │   └── Main
    │       ├── Topbar (title, language menu, references button, theme toggle)
    │       ├── Chat area (welcome screen + message list)
    │       └── Input bar (category pills, textarea, voice/send buttons, disclaimer)
    ├── Reference sidebar (law cards)
    ├── Company info modal
    └── <script>
        ├── KB — full legal knowledge base object
        ├── state — app state
        ├── UI helpers (theme, sidebar, modal, language)
        ├── classify() / process() — the "AI" matching engine
        ├── Message rendering (addMsg, buildRefs, buildGoogle, buildFollowups)
        ├── Session/history management
        └── Voice, export, print, share, toast utilities
```

---

## 6. Limitations / Things to Know

- **Not a real AI/LLM** — it's deterministic keyword-regex matching against a fixed dataset; it cannot answer questions outside the ~20 hardcoded topics beyond showing a generic fallback.
- **No backend, no database, no authentication** — everything runs client-side; refreshing the page loses the open conversation (only question titles persist).
- **Legal content is for Bangladesh law only** and is explicitly labeled as educational, not a substitute for professional legal advice.
- **Translation is superficial** — switching language only changes placeholder text and speech locale, not the actual legal answers.
- **Client-side "Like/Dislike" feedback** has no backend to record it — it's a local UI toggle only.

---

## 7. Suggested Next Steps (if extending this project)

- Connect a real LLM (e.g., via an API) for open-ended questions beyond the fixed knowledge base, using the existing KB as grounding/context.
- Add real localization of answer content, not just UI strings.
- Add a backend to persist full chat history across sessions/devices and to collect like/dislike feedback for quality improvement.
- Expand the knowledge base (more sections, more acts — e.g., Labour Act, VAT/Tax law, Consumer Rights Protection Act) relevant to Re-Volt's operations.
