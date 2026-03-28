# Mental Models Flashcard App — Project Brief
_Brainstorming summary for Claude Code handoff_

---

## Vision

A mobile-first flashcard app that quizzes users on the mental models and principles of great thinkers (called **Archetypes**). The goal is a genuinely delightful, fluid iPhone experience — inspired by the design philosophy of the [Family crypto wallet](https://benji.org/family-values): **simplicity, fluidity, and delight**.

The app is for personal growth and intellectual development. It should feel like a premium, calm, focused tool — not gamified or cluttered.

---

## Design Philosophy (from `benji.org/family-values`)

Three core principles must govern every design decision:

- **Simplicity** — Show only what's needed, when it's needed. Gradual revelation. Never overwhelm.
- **Fluidity** — Every transition must feel physically connected. No teleporting between states. Elements travel, not jump.
- **Delight** — Selectively placed moments of magic. The card physics, the bottom sheet spring, the score update — each should feel considered.

Avoid: cluttered UI, jarring transitions, generic design, any "AI slop" aesthetics.

---

## Current State

A fully working HTML prototype (`mental-models-app.html`) has been created. It contains:
- All core interactions (tap, swipe, double-tap, bottom sheet)
- All sample data for 3 archetypes
- Score tracking via localStorage
- Light + dark mode
- Correct font choices (Lora serif + DM Sans)

Claude Code should use this HTML file as the reference implementation and evolve it into a production-quality PWA or React app.

---

## Archetypes

Archetypes are anonymised versions of real thinkers. They must **not** use the real person's name anywhere in the UI. Use the archetype label only.

| Archetype Label | Inspired By | Emoji | Theme Colors |
|---|---|---|---|
| The Philosopher Investor | Ray Dalio | 📖 | Green `#3B6D11` / `#EAF3DE` |
| The Innovative Entrepreneur | Elon Musk | 🚀 | Blue `#185FA5` / `#E6F1FB` |
| The Philosophical Investor | Naval Ravikant | 🧭 | Purple `#7F77DD` / `#EEEDFE` |

Plans: scale to **100+ archetypes** over time. The architecture must support this gracefully — lazy loading, paginated home screen, search/filter.

---

## Data Structure

Each card object from the backend looks like this:

```json
{
  "id": 3,
  "principle": "Why Are Principles Important?",
  "question": "Full question text here...",
  "short_answer": "Concise 1-2 sentence answer shown on card reveal.",
  "elaborated_answer": "Full multi-paragraph deep-dive shown in the bottom sheet.",
  "chapter": "Part 1: The Importance of Principles",
  "section": "Part 1 — Why Are Principles Important?"
}
```

Each archetype has an array of these card objects. In the prototype, the data is hardcoded as a JS constant. In production, this should be fetched from an API endpoint.

---

## Core User Flow

```
Home Screen
  → Select Archetype
    → Card: Question shown (front)
      → Tap → Answer revealed (back)
        → Swipe Right ("Got it") → next card, score +1 correct
        → Swipe Left ("Missed it") → next card, score +1 reviewed
        → Double-tap (on revealed answer) → Bottom sheet opens with elaborated_answer
    → All cards done → Completion Screen
      → "Study again" → restart deck
      → "Back to archetypes" → Home Screen
```

---

## Interactions & Gestures (critical to get right)

### Card Reveal
- Tap anywhere on the card when in "question" state
- Smooth opacity fade between front and back face
- Action buttons (Missed / Got it) appear below the card after reveal
- While unrevealed: show "Tap card to reveal answer" hint with a pulsing dot

### Swipe to Score
- Works only after card is revealed
- Swipe **right** = correct ("Got it") — card flies off to the right with a clockwise rotation arc
- Swipe **left** = wrong ("Missed it") — card flies off to the left with counter-clockwise rotation arc
- While dragging: live overlay feedback on the card ("Got it" / "Missed" label with stamp-style border appears, opacity scales with drag progress)
- Velocity also triggers swipe (fast flick counts even before THRESHOLD px)
- After swipe: next card animates in from slight offset, spring physics
- Buttons below the card are tap alternatives to swiping — same result

### Double-Tap → Bottom Sheet
- Available only after card is revealed
- Double-tap (< 300ms between taps) opens the bottom sheet
- Bottom sheet contains: principle tag, short_answer as title, elaborated_answer as body
- Spring animation up (cubic-bezier with overshoot)
- Close by: tapping the backdrop, or dragging the sheet down (implement drag-to-dismiss)
- Sheet handle (pill) at top

### Card Stack Visual
- Two ghost cards sit behind the active card (scaled down, offset upward)
- They give the impression of a physical deck
- Ghost cards fade out as you approach the last cards in the deck

---

## Scoring

- **Per session per archetype**: track `total` reviewed and `correct`
- **Global totals**: sum across all archetypes, shown on home screen
- Persist in `localStorage` with key `mm_scores`
- Score structure: `{ [archetypeId]: { total: number, correct: number } }`
- Home screen shows: Reviewed / Correct / Accuracy %
- Each archetype card on home screen shows a progress bar (cards reviewed / total cards)
- Completion screen shows session stats for that archetype

Future: spaced repetition — missed cards resurface at end of deck or next session.

---

## Visual Design

### Typography
- **Display / headings / card text**: `Lora` (Google Fonts) — serif, warm, intellectual
- **UI / labels / buttons / counts**: `DM Sans` (Google Fonts) — clean, geometric, modern
- Weights: 400 regular, 500 medium only. Never bold/700.

### Color System (CSS custom properties)

```css
/* Light mode */
--bg: #F7F5F0;           /* warm off-white page background */
--surface: #FFFFFF;       /* card and panel surface */
--surface2: #F0EDE6;      /* subtle secondary surface */
--text: #1A1814;          /* primary text */
--text2: #6B6760;         /* secondary text */
--text3: #A8A49E;         /* hints, labels, disabled */
--gold: #B8860B;          /* accent — principle labels, progress bars */
--gold-light: #F5EDD6;    /* gold tint for backgrounds */
--green: #2E7D52;         /* correct / "Got it" */
--green-light: #E8F5EEE;
--red: #C0392B;           /* wrong / "Missed" */
--red-light: #FDECEA;
--border: rgba(0,0,0,0.08);
--border2: rgba(0,0,0,0.14);

/* Dark mode (via prefers-color-scheme: dark) */
--bg: #141412;
--surface: #1E1D1A;
--surface2: #252420;
--text: #F0EDE6;
--text2: #9A9690;
--text3: #5A5750;
--gold: #D4A820;
--gold-light: #2A2410;
--green: #4CAF7A;
--green-light: #0D2418;
--red: #E57373;
--red-light: #2A0E0E;
--border: rgba(255,255,255,0.07);
--border2: rgba(255,255,255,0.12);
```

### Layout & Spacing
- Max width: `430px`, centered — optimised for iPhone
- Safe area insets: respect `env(safe-area-inset-bottom)` for home indicator
- Card fills the stage area with `20px` horizontal padding, `16px` vertical
- Border radius: cards `20px`, buttons `14px`, avatars `14px`
- Borders: `1px solid var(--border)` on cards and panels — never thicker
- No drop shadows except on the active flashcard (subtle, multi-layer)

### Motion Principles
- Card fly-off: `transform 0.35s cubic-bezier(0.4, 0, 1, 1)` with rotation + fade
- Card entry: spring `cubic-bezier(0.34, 1.56, 0.64, 1)` — slight overshoot
- Bottom sheet: spring open, linear close
- Progress bar: `0.4s ease` transition
- Button appearance: opacity fade `0.2s`
- All drag interactions: `transition: none` while dragging, spring on release

---

## Screen-by-Screen Spec

### Home Screen
- Header: small uppercase label "Mental Models", large serif heading ("Study great minds, build sharper thinking.")
- Score strip: 3-column panel — Reviewed / Correct / Accuracy
- Archetype list: scrollable cards, each showing emoji avatar, name, description, card count, progress bar
- Tapping an archetype starts that deck from card 0 (resets session score for that archetype)

### Study Screen
- Header: back chevron, archetype name, card counter (e.g. "3 / 12")
- Thin gold progress bar just below header
- Chapter label (small uppercase, centered) above the card stack
- Ghost cards × 2 behind active card
- Active flashcard (front or back state)
- Action buttons below card (hidden until revealed)

### Flashcard — Front
- Small uppercase section tag
- Italic gold principle name
- Large serif question text
- Pulsing dot + "Tap card to reveal answer" hint at bottom

### Flashcard — Back
- "Answer" uppercase label
- Short answer text in serif
- Pulsing dot + "Double-tap for deeper context" hint

### Completion Screen
- Gold icon `✦`
- "Deck complete" serif heading
- Subtitle with archetype name and card count
- 3-stat row: Reviewed / Correct / Accuracy
- "Study again" primary button (dark bg)
- "Back to archetypes" ghost button

### Bottom Sheet (Deep Context)
- Handle pill at top
- "Deep context" uppercase tag
- short_answer as the sheet title (serif)
- Divider line
- elaborated_answer as body text (serif, slightly muted)
- Drag to dismiss + backdrop tap to close

---

## Technical Requirements

### Must-Haves
- Works as a **PWA** (Progressive Web App) installable on iPhone home screen
  - `manifest.json` with correct icons and `display: standalone`
  - Service worker for offline caching of HTML/CSS/JS/fonts
- `viewport` meta tag: `width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no`
- `touch-action: none` on draggable elements; `touch-action: pan-y` on scrollable sheets
- `-webkit-overflow-scrolling: touch` on scrollable sheet body
- `-webkit-font-smoothing: antialiased`
- No 300ms tap delay: handled via `touch-action` or FastClick

### Nice-to-Haves (Phase 2)
- API integration: fetch archetypes and cards from a backend endpoint
- Spaced repetition: missed cards requeue at end of deck
- Topic-based shuffle across archetypes
- Archetype library screen with search and filter by theme
- Haptic feedback on correct/incorrect swipe (using Vibration API where available)
- Share card feature (screenshot or share sheet)
- Onboarding / tutorial for first-time users

### State Management
- Keep it simple for now: vanilla JS + localStorage
- If migrating to React, use `useReducer` + `localStorage` sync
- No external state library needed at this scale

### Performance
- Fonts preloaded from Google Fonts
- Images: only emoji avatars (no raster images needed at launch)
- Animations: CSS transforms only (GPU composited), no layout-triggering properties
- `will-change: transform` on the active flashcard only

---

## File Deliverables

The Claude Code session should receive:
1. **This brief** (`mental-models-app-brief.md`)
2. **The HTML prototype** (`mental-models-app.html`) — working reference implementation

From there, Claude Code should either:
- Evolve the HTML file into a production-quality standalone PWA, OR
- Port it to a React/Vite project if greater component structure is needed

---

## Open Questions for Later

- Backend: where does card data live? (Notion, Airtable, custom API, JSON files?)
- Auth: is this personal/private or multi-user?
- Analytics: track which cards users consistently miss?
- Monetisation: freemium with some archetypes gated?
- Naming: the app needs a name ("Archetype", "Mentor", "Principles", "Luminary"?)
