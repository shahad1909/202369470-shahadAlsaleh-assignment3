# Technical Documentation

## Overview
This document explains the technical structure, design decisions, and
implementation details of my portfolio website. Assignment 3 builds on
Assignments 1 and 2 and focuses on **advanced functionality**:

1. Project filtering and sorting that work together (complex logic)
2. Remembering the visitor's filter and sort choices (state management)
3. A language breakdown for each project from a second GitHub API
   endpoint (API integration)
4. A visit timer (complex logic)
5. Performance and accessibility improvements measured with Lighthouse

Sections 1–6 describe the foundation from Assignments 1 and 2 (updated
where Assignment 3 changed it). Section 7 describes the new features.

---

## 1. Technology Stack
- **HTML5**: semantic page structure (`header`, `main`, `section`,
  `footer`), plus ARIA attributes for accessibility.
- **CSS3**: custom properties (variables) for theming, Flexbox, Grid,
  transitions, and `@keyframes` animations.
- **JavaScript (ES6+)**: no frameworks or libraries. All interactivity uses
  plain DOM APIs, `localStorage`, `fetch` with `async/await`,
  `Promise.allSettled`, and array methods (`map`, `sort`, `filter`,
  `reduce`, `slice`, `includes`).
- **GitHub REST API**: two public, read-only endpoints per project
  (`/repos/{repo}` and `/repos/{repo}/languages`), no API key needed.
- **Google Fonts**: Fraunces (headings) and Inter (body text).
- **GitHub Pages**: hosting for the live site.

No build tools, package managers, or external JS libraries are required.

---

## 2. File Structure

| File/Folder | Purpose |
|---|---|
| `index.html` | Main page: Navbar, then a `<main>` with the Hero, About, Achievements, Projects (with filter and sort controls), and Contact sections, then the Footer (with the visit timer). A small inline script in `<head>` applies the saved theme before the page is drawn. |
| `css/styles.css` | All styling, organized by section. Theme colors are CSS variables at the top, with a dark palette under `:root[data-theme="dark"]`. |
| `js/script.js` | All interactivity, inside one `DOMContentLoaded` listener and split into commented sections (one per feature). |
| `assets/images/` | Reserved for future images (kept in Git with an empty `.gitkeep` file). |
| `docs/images/` | Lighthouse screenshots used in the README. |
| `docs/ai-usage-report.md` | Documents all AI tool usage, with the usage log and declaration. |
| `docs/technical-documentation.md` | This file. |
| `.gitignore` | Excludes OS files, editor settings, and AI evidence files from the repository. |

---

## 3. Design System (CSS Variables)

All colors, radii, fonts, and shadows are defined once as CSS custom
properties in `:root`, so every component reads its colors from the same
place:

```css
:root {
  --bg: #fbf6f8;
  --surface: #ffffff;
  --ink: #2b2230;
  --muted: #71646f;
  --rose-deep: #ad4367;
  --error: #af435b;
  ...
}
```

The new Assignment 3 controls (filter buttons, sort menu) only use these
variables, so they adapt to dark mode automatically. For example, the
selected filter button uses `color: var(--surface)`: white text on dark
pink in light mode, and dark text on light pink in dark mode, so the
contrast stays good in both themes.

**Contrast fix (Assignment 3):** Lighthouse reported that some light-mode
text did not meet the WCAG AA contrast ratio of 4.5:1. I darkened three
variables, keeping the same hues:

| Variable | Before | After | Contrast on `--bg` / `--bg-alt` |
|---|---|---|---|
| `--muted` | `#8c7c89` (3.7 / 3.3) | `#71646f` | 5.2 / 4.7 |
| `--rose-deep` | `#c76e8c` (3.3 / 2.9) | `#ad4367` | 5.2 / 4.6 |
| `--error` | `#c15c73` (3.9) | `#af435b` | 5.2 / 4.6 |

Because every component uses the variables, changing three lines fixed
the whole site. The dark theme already passed, so it was not changed.

---
## 4. Responsive Design Approach
- **CSS Grid** is used for each project row (icon | text | link). It
  becomes two columns on tablet and a single column on mobile.
- **Flexbox** is used for the navbar, hero, About section, tag lists, and
  the contact form's error/counter row.
- Two breakpoints:
  - `@media (max-width: 768px)`, tablet: the nav links become a slide-in
    side menu opened by the hamburger button, and project rows switch to
    two columns.
  - `@media (max-width: 480px)`, mobile: project rows stack into one
    column, thumbnails get smaller, and achievements stack vertically.
    The project sort menu also moves under the filter buttons and
    stretches to the full width (Assignment 3).
- The new navbar layout uses `margin-left: auto` on `<nav>` so the links
  and the `.nav-actions` group (theme toggle + hamburger) sit on the right
  on every screen size.

---

## 5. Features Carried Over from Assignment 1

### 5.1 Smooth Scrolling
Internal links (`href="#section"`) are intercepted with a `click`
listener. JavaScript subtracts the sticky navbar's height from the target
position so section titles are never hidden behind the navbar.

**Changed in Assignment 3:** the navbar height (`offsetHeight`) was read
once on page load, which forced the browser to calculate the layout early
(Lighthouse "Forced reflow"). It is now read inside the click handler, so
there is no extra layout calculation on load, and the value is always
current (for example after rotating a phone). The deprecated
`window.pageYOffset` was also replaced with `window.scrollY`.

### 5.2 Mobile Navigation Toggle
The hamburger button toggles an `.open` class on `.nav-links`, which
slides the side menu in and out with a CSS `transform` transition.

### 5.3 Time-Based Greeting
`new Date().getHours()` picks a greeting ("Good morning", "Good
afternoon", etc.) for the hero badge.

---

## 6. Features from Assignment 2

### 6.1 Expandable Project Details
**Requirement covered:** Dynamic Content (show/hide on click), Animation.

**HTML:** each project has a toggle button and a details panel, linked by
`aria-controls` → `id`:

```html
<button class="details-toggle" type="button"
        aria-expanded="false" aria-controls="details-fs">
  <span class="toggle-label">Show more</span>
  <span class="toggle-icon" aria-hidden="true">+</span>
</button>
<div class="project-details" id="details-fs">
  <div class="details-inner">...</div>
</div>
```

**CSS: the smooth slide.** CSS cannot animate `height: auto`, so the panel
is a one-row grid whose row animates from `0fr` (closed) to `1fr` (open,
full content height):

```css
.project-details { display: grid; grid-template-rows: 0fr; visibility: hidden;
                   transition: grid-template-rows 0.25s ease-in-out, visibility 0.25s; }
.project-details.open { grid-template-rows: 1fr; visibility: visible; }
.details-inner { overflow: hidden; min-height: 0; }
```

- `.details-inner` is the single grid item; `min-height: 0` lets it
  shrink to zero and `overflow: hidden` clips the text while it slides.
- `visibility: hidden` stops keyboard and screen-reader users from
  reaching the hidden text. Because it is transitioned too, the text
  stays visible until the closing slide finishes.
- The `+` icon rotates 45° into an `×` using the attribute selector
  `.details-toggle[aria-expanded="true"] .toggle-icon`, so the visual
  state and the accessibility state always match.

**JavaScript:** one `click` listener per button (added with `forEach`):

1. Find the panel with `document.getElementById(toggle.getAttribute('aria-controls'))`.
2. Read the current state from `aria-expanded`.
3. Flip `aria-expanded`, toggle the `.open` class on the panel, and switch
   the label between "Show more" and "Show less".

The label is in its own `<span>`, so changing its text does not remove
the `+` icon.

---
### 6.2 Dark / Light Theme Toggle (localStorage)
**Requirement covered:** Data Handling (save user preferences in
`localStorage`).

**How the theme switches:** the dark palette redefines the same variable
names under a more specific selector:

```css
:root[data-theme="dark"] {
  --bg: #1d1820;
  --ink: #f3eaf0;
  ...
  color-scheme: dark;
}
```

When `<html>` has `data-theme="dark"`, this rule wins by specificity
(pseudo-class + attribute selector beats pseudo-class alone), every
`var(...)` resolves to the dark value, and the whole site changes at
once. `color-scheme: dark` also makes browser-drawn parts such as
scrollbars dark.

**JavaScript logic:**

| Function / step | What it does |
|---|---|
| `applyTheme(theme)` | Adds or removes `data-theme="dark"` on `<html>`, and updates the button icon (🌙 / ☀️) and its `aria-label`. |
| `readSetting('theme')` | `localStorage.getItem('theme')`, which returns `'dark'`, `'light'`, or `null` on a first visit. |
| `saveSetting('theme', theme)` | `localStorage.setItem('theme', theme)`. |
| On page load | Uses the saved theme first. If nothing is saved, follows the device setting with `window.matchMedia('(prefers-color-scheme: dark)')`. |
| On click | Flips the current theme, applies it, and saves it. |

In Assignment 3, the original `getSavedTheme()` / `saveTheme()` were
replaced by the shared `readSetting(key)` / `saveSetting(key, value)`
helpers (see section 7.2), which the project filter and sort also use.

Both storage functions are wrapped in `try/catch`: in some private
browsing modes, `localStorage` throws an error, and without the
`try/catch` the whole script would stop. In that case the theme still
works for the current visit; it just is not remembered.

**Preventing a light flash on load:** `script.js` runs at the end of the
page, so a dark-mode visitor could briefly see the light theme while the
page loads (most visible on slow connections). A small inline script in
`<head>` reads `localStorage` and sets `data-theme="dark"` **before** the
body is drawn. `script.js` still runs afterwards to update the icon and
label, because the button does not exist yet when the head script runs.

---
### 6.3 Live GitHub Repository Info (Public API)
**Requirements covered:** Data Handling (fetch from a public API), Error
Handling (loading indicator and friendly error message).

**HTML:** a placeholder under each project title stores the repository
name in a `data-repo` attribute and starts in a loading state:

```html
<p class="repo-meta is-loading" data-repo="shahad1233/File-System-Simulator">
  Loading GitHub info…
</p>
```

**Request flow (`loadRepoInfo`, an `async` function):**

```
fetch("https://api.github.com/repos/" + repo)
  ├─ response.ok is false (404, 403, 500…) → throw an Error
  └─ response.ok is true → response.json()
       → show "Created <date> · Updated <date>"
catch   → show a friendly red error message + a Try again button, and log details to the console
finally → remove the .is-loading class (stop the pulse)
```

- **`async/await`** lets the page stay responsive while waiting for the
  network. All three requests start at the same time (one per project).
- **`response.ok` check:** `fetch` only rejects on network failures, not
  on HTTP errors such as 404, so the code checks `response.ok` and throws
  its own error for those cases.
- **Dates:** `created_at` and `pushed_at` (the date of the last code
  push, which is more meaningful than `updated_at`, which also changes
  when a repo is starred). Dates are formatted with
  `toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })`,
  for example `June 12, 2026`.
- **Feedback states:**

| State | What the user sees | CSS |
|---|---|---|
| Loading | "Loading GitHub info…" | `.is-loading` → `pulse` animation (opacity 1 → 0.4 → 1, infinite) |
| Success | "Created June 12, 2026 · Updated June 12, 2026" | muted small text |
| Error | "We couldn't fetch GitHub data. Check your connection, then click Try again." plus a Try again button that reloads only that project | `.is-error` → `var(--error)` color |

- **Specificity note:** the selector is `.project-body .repo-meta`
  (two classes) so it overrides the existing `.project-body p` rule
  (one class + one element).
- Each project is handled independently, so one failed request does not
  affect the others.
- **Assignment 3 additions:** a successful request also saves
  `created_at` on the project card as `data-created` (used for sorting,
  section 7.1) and starts the language request (section 7.3). After a
  successful **Try again**, `loadRepoInfo(metaEl).then(updateProjects)`
  re-sorts the list so the newly loaded date is used.

---
### 6.4 Improved Contact Form Feedback
**Requirements covered:** Error Handling (form error messages),
Animation (animated success and error messages).

The Assignment 1 validation rules are unchanged (name ≥ 2 characters,
valid email pattern, message ≥ 10 characters). What changed is **when**
and **how** feedback is shown.

| Behavior | Implementation |
|---|---|
| One validator per field | `validateName()`, `validateEmail()`, `validateMessage()`, each reusing the original `validateField()` helper, so the same rules run on submit, on blur, and while typing. |
| Check on leaving a field | `blur` listener, only if the field is not empty, so tabbing through an empty form does not show errors. |
| Errors clear as soon as fixed | `input` listener that re-validates only while the field has the `.invalid` class. |
| Shake on submit | Invalid fields get a `.shake` class (`@keyframes shake`: ±8px horizontally over 0.7s). An `animationend` listener removes the class so the field can shake again on the next submit. |
| Focus the first error | `invalidFields[0].input.focus()` (uses `Array.filter` to collect the invalid fields). |
| Character counter | `input` listener updates "`n` / 10 characters minimum". It turns green (`.is-complete`) at 10. It uses `.trim().length`, the same as the validator. The minimum lives in one constant, `MIN_MESSAGE_LENGTH`. |
| Personalized success | "Thanks for reaching out, *FirstName*! I’ve received your message and will reply as soon as possible.". Inserted with `textContent` (never `innerHTML`) so user input cannot inject HTML. |
| Success entrance | `@keyframes fade-slide-in` (fade in + slide down 8px). An `animation` is used instead of a `transition` because the element changes from `display: none`, which transitions cannot animate. |
| Reliable auto-hide | The 5-second hide timer is stored in `successTimer` and cleared with `clearTimeout` on every submit, so an old timer cannot hide a newer message early. |
| Counter reset | `updateCounter()` is called after `form.reset()`, because `reset()` does not fire `input` events. |

---

## 7. New Features in Assignment 3

### 7.1 Project Filter + Sort (working together)
**Requirement covered:** Complex Logic (filter and sort at the same time).

**HTML:** each project has a category, and each filter button has a
matching value:

```html
<article class="project-row" data-category="software">...</article>

<button type="button" class="filter-btn active" data-filter="all" aria-pressed="true">All</button>
<button type="button" class="filter-btn" data-filter="software" aria-pressed="false">Software</button>

<select id="sort-select" class="sort-select">
  <option value="newest">Newest first</option>
  <option value="oldest">Oldest first</option>
  <option value="name">Name (A–Z)</option>
</select>

<p class="empty-state" id="empty-state" hidden>No projects in this category yet.</p>
```

**State:** two variables hold the current choices. They start with the
same defaults as the HTML (`all`, `newest`):

```js
let currentFilter = 'all';
let currentSort = 'newest';
```

Keeping both in variables is what lets the two controls work together:
when the sort changes, the current filter is still known, so hidden
projects stay hidden.

**Logic, split into three small functions:**

| Function | Responsibility |
|---|---|
| `matchesFilter(row)` | Returns `true` if `currentFilter === 'all'` **or** the project's `data-category` equals `currentFilter`. |
| `compareProjects(a, b)` | Comparison function for `Array.sort()` (negative → `a` first, positive → `b` first, `0` → keep order). |
| `updateProjects()` | Sorts a copy of the projects, shows/hides each one, renumbers the visible ones, moves them into the new order, and shows the empty-state message if nothing matches. |

**`compareProjects` rules, step by step:**
1. **Name (A–Z):** compare the `<h3>` titles with `localeCompare`, which
   sorts like a dictionary (unlike `<`, which puts every capital letter
   before every lowercase letter).
2. **Projects without a date go last:** if the GitHub request failed,
   the card has no `data-created`. These projects are always placed at
   the end, for both Newest and Oldest, because their position cannot be
   known:
   ```js
   if (!dateA && !dateB) return 0;
   if (!dateA) return 1;    // a goes after b
   if (!dateB) return -1;   // a goes before b
   ```
3. **By date:** subtracting two dates gives the difference in
   milliseconds, which already sorts oldest first. For "Newest first",
   the sign is flipped:
   ```js
   const difference = new Date(dateA) - new Date(dateB);
   return currentSort === 'oldest' ? difference : -difference;
   ```
   Example: A is from 2023 and B from 2024 with "Newest first":
   `difference` is negative, `-difference` is positive, so B (newer)
   comes first.

**`updateProjects()` details:**
- `[...projectRows].sort(compareProjects)` sorts a **copy**, so the
  original array is not changed.
- `row.hidden = !matchesFilter(row)` hides or shows each project.
- `projectsList.appendChild(row)` **moves** the existing card to the end
  of the list (an element cannot be in two places), so appending the
  cards in sorted order puts the page in that order.
- Visible projects are renumbered with
  `String(visibleCount).padStart(2, '0')` (01, 02, 03), so the numbers
  always follow what the visitor sees. The numbers written in the HTML
  remain as a fallback if JavaScript does not run.
- `emptyState.hidden = visibleCount > 0` shows the message only when no
  project matches.

**Filter buttons:** a click updates `currentFilter`, marks the button as
active with `setActiveFilter(button)` (the `.active` class for the style
and `aria-pressed` for screen readers), saves the choice, and calls
`updateProjects()`. The sort menu does the same on its `change` event.

**CSS notes:**
- `.project-row` uses `display: grid`, which overrides the browser's
  default style for the `hidden` attribute, so hidden projects would
  still appear. A stronger selector fixes it:
  ```css
  .project-row[hidden] { display: none; }
  ```
  (class + attribute is more specific than the class alone).
- `:focus-visible` shows an outline only for keyboard users, not on
  mouse clicks.

**Waiting for the GitHub dates:** sorting by date needs every date
first. `loadRepoInfo` is an `async` function, so it returns a Promise.
The requests start together, and the list is sorted again once all have
finished:

```js
const repoRequests = Array.from(repoMetaEls).map(loadRepoInfo);

Promise.allSettled(repoRequests).then(() => {
  updateProjects();
});
```

I chose `Promise.allSettled` instead of `Promise.all`: `Promise.all`
stops at the first rejected Promise, while `allSettled` always waits for
every request. `loadRepoInfo` already catches its own errors, so both
work today, but `allSettled` keeps the sorting working even if that
function changes later.

---

### 7.2 State Management (remembered choices)
**Requirement covered:** State Management.

**Shared storage helpers:** instead of separate theme functions,
Assignment 3 uses two general functions that take the key as a
parameter (avoiding repeated `try/catch` code):

```js
function readSetting(key) {
  try { return localStorage.getItem(key); }
  catch (error) { return null; }          // storage unavailable
}

function saveSetting(key, value) {
  try { localStorage.setItem(key, value); }
  catch (error) { /* still works for this visit */ }
}
```

| Key | Saved value | Saved when |
|---|---|---|
| `theme` | `'dark'` / `'light'` | The theme button is clicked |
| `projectFilter` | `'all'`, `'software'`, `'hardware'`, `'ml'` | A filter button is clicked |
| `projectSort` | `'newest'`, `'oldest'`, `'name'` | The sort menu changes |

**Restoring on load, with validation:** a saved value is only used if
it still matches an existing button or option. For example, if a
category is removed in the future, a visitor with that old value would
otherwise see an empty list:

```js
const savedFilterButton = document.querySelector(
  `.filter-btn[data-filter="${readSetting('projectFilter')}"]`
);
if (savedFilterButton) { ... }            // null → keep "All"

const sortOptions = Array.from(sortSelect.options).map(option => option.value);
if (sortOptions.includes(savedSort)) { ... }   // otherwise keep "Newest"
```

`updateProjects()` is then called immediately, so the saved filter is
visible before the GitHub data arrives.

I tested this by changing `projectFilter` to `xyz` in DevTools
(Application → Local Storage): the page correctly fell back to "All".

---

### 7.3 Language Breakdown (second GitHub API endpoint)
**Requirement covered:** API Integration.

The AI first suggested showing each repository's star count. I chose a
language breakdown instead, because stars are almost always 0 for
student projects, while the languages show what each project is built
with.

**Endpoint:** `GET https://api.github.com/repos/{repo}/languages`
returns the number of bytes of code per language:

```json
{ "Java": 45210, "Shell": 830 }
```

**`getLanguagePercentages(languageBytes)`** turns it into a sorted list:
1. `Object.values(...).reduce(...)` adds up the total bytes. If it is
   `0`, it returns an empty list.
2. `Object.entries(...)` gives `[name, bytes]` pairs, which `map`
   converts to `{ name, percent }`.
3. `sort((a, b) => b.percent - a.percent)` puts the largest first.
4. `slice(0, 3)` keeps the top 3, and the rest (`slice(3)`) are added
   together as **"Other"**, so the line stays short.

Example: `{ Python: 800, HTML: 100, CSS: 60, JavaScript: 30, Shell: 10 }`
→ `Python 80.0% · HTML 10.0% · CSS 6.0% · Other 4.0%`.

**`loadRepoLanguages(metaEl, repo)`** fetches the data, builds a new
`<p class="repo-languages">` with `document.createElement`, joins the
parts with `' · '` (using `toFixed(1)` for one decimal place), and
inserts it under the dates with `metaEl.after(...)`.

**Design decisions:**
- **Graceful degradation:** the languages are extra information, so if
  this request fails, the line is simply not shown and a
  `console.warn` (not `console.error`) is logged. The dates still work.
- **Started only after the dates succeed:** if the first request fails,
  GitHub is probably unreachable or rate-limited, so a second request
  would fail too.
- **No `await`:** `loadRepoLanguages` is called without `await`, so the
  dates appear immediately and the languages appear when ready.

---

### 7.4 Visit Timer
**Requirement covered:** Complex Logic (timer).

The footer shows "You've been here for 45s", then "2m 05s".

- The start time is saved once with `Date.now()`. Every second
  (`setInterval(updateVisitTimer, 1000)`), the elapsed time is
  calculated as `Date.now() - visitStart`. This is more accurate than
  adding 1 to a counter, because browsers slow down timers in
  background tabs, which would make a counter fall behind.
- `formatDuration(totalSeconds)`: minutes with `Math.floor(total / 60)`,
  remaining seconds with `total % 60`. Under one minute it shows only
  seconds; otherwise the seconds are padded with `padStart(2, '0')`.
- `updateVisitTimer()` is also called once immediately, so the text
  appears without waiting one second.
- `aria-live="off"` stops screen readers from reading the number every
  second, and `font-variant-numeric: tabular-nums` gives every digit the
  same width so the text does not shake.

---

## 8. Accessibility
- All interactive controls are real `<button>` elements (keyboard
  accessible by default).
- `aria-expanded` + `aria-controls` on the project toggles.
- `aria-pressed` on the filter buttons, so screen readers know which
  filter is selected (Assignment 3).
- `:focus-visible` outlines on the filter buttons and sort menu for
  keyboard users (Assignment 3).
- `<label for="sort-select">` connects the "Sort by" text to the menu.
- `aria-label` on icon-only buttons (theme toggle, hamburger), updated by
  JavaScript when the theme changes.
- `aria-hidden="true"` on decorative icons and emojis.
- `aria-describedby` links the message field to its character counter.
- `role="status"` + `aria-live="polite"` on the success message, so
  screen readers announce it; `aria-live="off"` on the visit timer.
- Light-mode colors meet the WCAG AA contrast ratio (section 3), and the
  sections are inside a `<main>` landmark (Assignment 3). Together these
  raised the Lighthouse Accessibility score from 95 to 100.
- `@media (prefers-reduced-motion: reduce)` disables all transitions
  and animations.

---

## 9. Error Handling Summary

| Situation | How the site responds |
|---|---|
| GitHub API unreachable, repo not found, or rate limit reached (403) | Friendly message in red with a Try again button; details logged with `console.error`. Other projects still load. After a successful retry, the list is re-sorted. |
| Language request fails, or GitHub detects no code | The language line is not shown; a `console.warn` is logged. |
| Some projects have no date | They are placed last when sorting by date. |
| No project matches the filter | "No projects in this category yet." is shown. |
| Saved filter/sort value is no longer valid | Ignored; the default (All / Newest) is used. |
| `localStorage` unavailable | Caught with `try/catch`; everything still works for the current visit. |
| Invalid or empty form field | Inline error message, red border, shake, and focus moved to the first invalid field. |

---

## 10. Performance

Tested with PageSpeed Insights (Lighthouse) on the live GitHub Pages site.

| | Performance | Accessibility | Best Practices | SEO |
|---|---|---|---|---|
| Mobile, before | 100 | 95 | 100 | 100 |
| Mobile, after | 100 | 100 | 100 | 100 |
| Desktop, after | 100 | 100 | 100 | 100 |

Performance was already 100, but Lighthouse listed improvement
opportunities, which I fixed:

| Lighthouse insight | Cause | Fix |
|---|---|---|
| Render-blocking requests (≈690 ms) | Google Fonts loaded 8 font weights, but only 5 are used | Load only Fraunces 500/600 and Inter 400/500/600 (3 fewer font files) |
| Network dependency tree | The font files come from a second server (`fonts.gstatic.com`) that was only contacted late | `<link rel="preconnect">` to `fonts.gstatic.com` (with `crossorigin`) and to `api.github.com` |
| Forced reflow | `offsetHeight` read on page load | Read the navbar height on click instead (section 5.1) |
| Efficient cache lifetimes | GitHub Pages sets a 10-minute cache | Not changeable on GitHub Pages (server setting) |

Other performance choices:
- No frameworks or libraries; only three small files plus fonts.
- The language breakdown reuses the same repositories and runs in the
  background, and all GitHub requests are sent in parallel (`map` starts
  them together), so the total wait is about one request, not six in a row.
- The filter and sort move existing elements instead of rebuilding the
  HTML, and the duplicate sort listener left over from development was
  removed so `updateProjects()` runs once per change.
- Animations use `transform` and `opacity`; transitions list their
  properties instead of `all`.

The Performance score varies slightly between runs (98–100), depending on
network conditions at the time of the test.

---

## 11. Browser Compatibility
Tested on the live site in Chrome and Edge at desktop, tablet, and
mobile sizes using DevTools device emulation, in both light and dark
themes, and with keyboard-only navigation. All features use standard,
widely supported APIs (CSS variables, Grid, Flexbox, `fetch`,
`async/await`, `Promise.allSettled`, `localStorage`, `:focus-visible`).

I also tested the live site on my iPhone using Safari. I checked the main features like the menu, the theme button, filtering and sorting the projects, the GitHub dates and language percentages, and the visit timer, and everything worked fine.

---

## 12. Debugging Notes

**Language percentages did not appear.** The code had no errors, so I
checked the Network tab: the `/languages` requests returned status
`200`, but the response was `{}`. The code handled this correctly
(`languages.length === 0` → show nothing). The real cause was that my
three project repositories contained only `.zip` files and a README, and
GitHub cannot detect languages inside a zip. I uploaded the actual
source files and deleted the zips, and the percentages appeared (Java,
Jupyter Notebook, Verilog). This also explains why, in Assignment 2, the
API returned `null` as the main language for all three repositories.

**GitHub error 403 while testing.** After many refreshes (with
"Disable cache" on), GitHub returned `403` because the 60-requests-per-hour
limit was reached. The site showed the friendly error and Try again
button as designed. I waited and used PageSpeed Insights, which runs
from Google's servers, in the meantime.

**A misplaced bracket stopped all JavaScript.** When adding the project
renumbering, I placed a line outside its `if` block, leaving an extra
`}`. The editor highlighted the error in red, and I fixed it before
testing.

**Duplicate event listener.** A review found that the old sort listener
from step 4 was still in the file next to the new one, so
`updateProjects()` ran twice per change. I removed the old one.

---

## 13. Known Limitations / Future Improvements
- **GitHub API rate limit:** unauthenticated requests are limited to 60
  per hour per IP address, and each page load uses 6 (2 per project).
  If the limit is reached, the friendly error message is shown.
- **Sorting by date uses the repository creation date:** all three
  repositories were uploaded to GitHub on the same day, so "Newest" and
  "Oldest" currently give a similar order. The sort uses when the repo
  was created on GitHub, not when the project was actually done.
- **Cache lifetime** is controlled by GitHub Pages and cannot be changed.
- **No backend:** the contact form validates and confirms, but does not
  send the message anywhere (matches the assignment scope).
- **Future improvement, caching GitHub responses:** saving the API
  responses in `localStorage` for about an hour would make repeat visits
  faster and avoid the rate limit.
- **Future improvement:** store a real project date in the HTML (for
  example `data-date`) so sorting reflects when each project was done.