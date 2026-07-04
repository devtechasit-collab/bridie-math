# Math Carry App — Project Handoff

**File:** `math-carry.html` (single-file, no build step)
**Purpose:** Bilingual (Thai/English) interactive web app teaching a Grade 1 child two-digit addition/subtraction with carrying and borrowing, via animated concrete manipulatives (ten-frame dots, ten-bars). Built for a specific child who understood single-digit math but had no visual model for "carrying."

This doc is a handoff summary of everything designed and fixed so far, written so a fresh Claude Code session can continue without re-deriving context from scratch.

---

## 1. Tech stack & constraints

- **Single HTML file.** Vanilla JS, inline `<style>`, inline `<script>`. No React, no build step, no npm deps.
- Fonts: Google Fonts `Baloo Thai 2` (headings/numbers) + `Itim` (body) — both render Thai and Latin correctly, chosen specifically so TH/EN toggle doesn't need separate font stacks.
- No backend, no persistence — everything resets on page reload. Single session only.
- Target device: mobile/tablet, portrait, one child using touch.

---

## 2. Screens & lesson model

Two screens, toggled via `#home`/`#board` visibility:

- **Home**: hero header + 6 lesson cards (`LESSONS` object, keys: `add1`, `add2n`, `add2c`, `sub1`, `sub2n`, `sub2b` — easy→hard, no-carry before carry).
- **Board**: vertical sum display (`#sum`) + concrete manipulative area (`#frames` + `#grouplegend`) + narration bubble (`#say`) + controls, under two modes:
  - **Watch** (`mode='watch'`): scripted step-by-step animation with narration, driven by `buildSteps()` + `nextStep()`.
  - **Try it** (`mode='try'`): child taps answer boxes, types on an on-screen number pad, self-checks via `checkAnswer()`.

`state` object holds current lesson key, mode, language, and the two operands (`state.a`, `state.b`).

---

## 3. The step engine (core mechanic — read before touching Watch mode)

`buildSteps()` returns an array of step objects for the *current* `state.a`/`state.b`/`state.key`:

```js
{ say: '<narration html>', col: 'ones'|'tens'|'both'|null, pulse: '<elementId>'|undefined, fx: () => { /* render this step */ } }
```

`nextStep()` walks this array one at a time:
1. Sets narration text (`setSay`)
2. Applies column highlight (`focusCol(col)` / `clearFocus()`) — tints the relevant column of the vertical sum
3. Runs `fx()` — the actual DOM mutation for this step
4. Pulses a specific element if `pulse` is set (draws the eye to e.g. the carry box)
5. Updates the step progress bar (`renderStepbar`)
6. Debounce-locks for ~750ms to prevent double-tap skipping a step

**Important:** `renderStepbar`'s total step count is always `state.watchSteps.length` — never hardcode a step count anywhere. Step count differs by lesson type and by whether carry/borrow triggers.

---

## 4. Visual language — established through iteration, please preserve

These color/shape conventions were arrived at through several rounds of user feedback (a parent testing with their child) and represent real pedagogical decisions, not arbitrary style. If asked to touch visuals, check this section first.

| Concept | Visual | CSS var / class |
|---|---|---|
| Ones place | Orange dot in a 5×2 ten-frame | `--ones` (#FF8A3D) |
| "Second group" of ones (2nd addend, or borrowed-in ones) | Lighter orange dot | `--ones-b` (#FFC98A), class `.groupB` |
| Tens place | Small blue bar (`.tenbarmini`) | `--tens` (#3B7DD8) |
| "Second group" of tens (2nd addend's tens) | Lighter blue bar | `--tens-b` (#8FB8E8), class `.groupB` |
| **A carried value** (just bundled from 10 ones, moving to tens) | **Green** bar (`.tenbarv.ascarry`) | `--carry` (#12B981) — matches the green carried digit already shown in the sum |
| **A borrowed value** (an existing ten breaking apart into ones) | **Blue** bar (`.tenbarv`, no `.ascarry`) | stays `--tens` blue — it's not a new carried value, it's an existing tens-place value |

**Why carry=green but borrow=blue:** the carried digit in the vertical sum is rendered green (`.ascarry` class on the digit). The bundled-ten bar during carrying was originally blue, which didn't visually connect to the green digit it becomes — this was reported as confusing and fixed. Borrowing doesn't produce a "new" green value; it's just an existing blue ten changing form, so blue was kept there deliberately.

**Grouping + legend pattern:** whenever two counts combine into one ten-frame/bar-row (e.g., 5+3 ones, or 2 original + 10 borrowed ones), the two counts get two shades of the same color AND a numeric "legend" pill row underneath (`showLegend(a,b)` / `#grouplegend`) so the color split maps to actual numbers a child can read. This pattern must be applied consistently — a past bug was the subtraction/borrow flow showing the two-tone dots but *no* legend, unlike addition. Fixed; keep both in sync if either changes.

**Column highlight (`focusCol`):** the cell(s) in `#sum` matching `data-col="ones"`/`"tens"` get a tinted background + border when a step is "about" that column. Must match what the narration text is actually describing — a past bug had the "split the carry" step highlighting `tens` when the narration talked about the ones leftover first; fixed to `col:'ones'`.

**Concrete label (`#concreteLabel`) must never go stale.** Every `fx()` that meaningfully changes the dot/bar picture should also update this label to describe what's currently on screen. A confirmed bug (see §6) was labels freezing at the lesson's initial description while dots visually changed underneath — always update label + dots together, never one without the other.

---

## 5. i18n (TH/EN toggle)

All user-facing strings live in `const T = { th: {...}, en: {...} }` (~line 302). Access via `S()` which returns `T[state.lang]`. Rules:

- **Never** hardcode a Thai or English string anywhere else in the JS — always add to both `T.th` and `T.en` and call through `S()`.
- Functions with parameters (e.g. `addOnes(oA,oB,oSum)`) exist in both languages with matching signatures.
- `setLang(l)` re-renders static chrome (`applyStatic()`) + rebuilds the home grid + re-renders the current board if one is open. If you add new static UI text, wire it into `applyStatic()` too or it won't update on language switch.

---

## 6. Testing methodology (read before making changes — avoids repeat mistakes)

This is a single HTML file with no test framework, so verification was done three ways. **Use all three for any non-trivial change**, in this order:

### a) Syntax check
Extract the `<script>` contents and run `node --check` on it. Catches typos before anything else.

### b) Headless logic harness (Node + a tiny DOM shim)
A minimal `document`/`window` shim (no real rendering) lets `buildSteps()`, `nextStep()`, `makeProblem()` etc. run in plain Node. Loop over many random problems × all 6 lessons × both languages, drive every step via `nextStep()` in a `while` loop, and assert the sequence completes without throwing. This catches crashes and logic errors but **cannot** catch stale-label/visual-mismatch bugs, since the shim doesn't compute real layout or verify a human would perceive it correctly — it only proves the code runs.

Key shim gotchas already solved (reuse this pattern, don't rebuild from scratch):
- `document.getElementById` must return a **persistent** element per id (memoize in a registry object), not a fresh one every call — otherwise `innerHTML` writes are invisible to later reads.
- `setTimeout`/`setInterval` should be stubbed to either run synchronously (simple case) or queue into an array you flush manually (needed when you want to inspect an *intermediate* state before a delayed callback fires, e.g. the bar-then-dots two-phase borrow animation).

### c) Real browser via Playwright (this is what actually caught the visual bugs)
A pre-installed headless Chromium binary exists in this sandbox at:
```
/opt/pw-browsers/chromium_headless_shell-1194/chrome-linux/headless_shell
```
Playwright's Python package is already installed (`pip show playwright` succeeds), but `playwright install chromium --with-deps` **fails** in this sandbox (network is restricted, can't reach `deb.nodesource.com` for apt deps). Use the pre-installed binary directly via `executable_path=` instead of trying to install.

```python
from playwright.sync_api import sync_playwright
browser = p.chromium.launch(executable_path="/opt/pw-browsers/chromium_headless_shell-1194/chrome-linux/headless_shell")
```

**Critical gotcha — forcing a specific test problem:** `setMode(m)` internally calls `newRound()` → `newProblem()`, which **randomizes** `state.a`/`state.b` via `makeProblem()`. If you set `state.a = 32; state.b = 15` and then call `setMode('watch')`, your values get overwritten by a random draw. To force a specific problem for testing:

```js
document.getElementById('board').classList.add('show');
state.key = 'sub2b'; state.mode = 'watch';
renderControls();           // sets up buttons, does NOT randomize
state.a = 32; state.b = 15; // force AFTER renderControls
renderRound();               // re-renders WITHOUT calling newProblem() — safe
```

`renderRound()` (not `newRound()`) is the one that doesn't re-randomize. This was the source of an entire false debugging trail earlier in this project — a test that appeared to show a bug was actually testing the wrong (randomly-substituted) problem.

Once a specific problem is loaded, call `nextStep()` in a loop with real waits (`page.wait_for_timeout(900)`, enough to cover any internal delayed callbacks — some steps chain multiple `setTimeout`s up to ~800ms), then inspect via `page.evaluate()` (read `textContent`, `className`, `getComputedStyle(...).backgroundColor`, or take `page.screenshot()`). Screenshots + computed styles are what actually caught: a frozen label bug and a missing-legend bug that the Node harness could never have detected, since both were about *content correctness*, not crashes.

---

## 7. Change log (chronological, high-level)

1. Initial single-file app: 6 lessons, Watch/Try modes, ten-frame dot manipulative, carry/borrow narration.
2. Fixed race conditions in dot-fill animations (were `setInterval`-staggered, causing timing bugs) — switched to instant CSS-driven renders.
3. Split into TH-only / EN-only files per request, then **merged into one file with a TH/EN toggle** (`T` dictionary + `S()` pattern established here). One real bug hit during the merge: dictionary entries were inserted *before* call-site string replacements ran, so replacements matched their own dictionary literals and created self-referential values (`hintGeneric: S().hintGeneric`) — a TDZ crash. Fixed by inserting the dictionary last. Worth remembering if doing large regex/string-replace refactors on this file again.
4. Redesigned the concrete manipulative: removed a confusing simultaneous "multiple blue tens-bars + dots" display, consolidated to one ten-frame + a single bundling bar.
5. Added dot **grouping** (two-tone + numeric legend) so addition visually shows "5 dots + 3 dots", not one undifferentiated blob of 8.
6. Added step-highlight system (`focusCol`, step progress bar) so a child stuck on a step can see exactly which column is being discussed.
7. Reworked the carry sequence per parent feedback: write the full two-digit sum (e.g. "16") aligned below the columns first, *then* a slow-motion step visually splits it into "6 stays, 1 flies up" — this two-phase reveal was specifically requested to build the "aha" moment.
8. Fixed carry-bar color (blue → green) to match the carried digit's existing green color; kept borrow-bar blue (different semantic, see §4).
9. Replaced a blank/empty concrete area during the tens-addition step with actual color-matched tens-bars (was previously just clearing the frame to nothing).
10. Merged two redundant steps (bundle-to-bar + write-the-sum) into one, since nothing visually changed between them and it wasted a tap.
11. Fixed step-highlight column mismatch on the "split carry" step (was highlighting tens, narration was about ones).
12. Extended two-tone shading to tens-bars (was previously a single flat blue for both addends' tens).
13. **Most recent fix** — subtraction/borrow flow: label text was frozen at the lesson's initial description throughout the entire borrow sequence (never reflected "borrowed 10, now have 12" or "7 left"), and the borrow's two-tone dot grouping had no numeric legend (unlike addition's equivalent). Both fixed: `borrowDown()` now updates label+legend once the borrowed dots land; the post-subtraction step now updates to a "leftover" label and hides the legend, mirroring how addition's `splitCarry()` already behaved.

---

## 8. Known-good state / what NOT to break

- All 6 lessons × both languages × both modes currently pass the Node harness with zero thrown errors across 40 random problems each.
- Playwright-verified: addition carry sequence (color-matched bar/digit, merged bundle+write step, grouped legend), subtraction borrow sequence (color-matched dots, dynamic label/legend), subtraction no-borrow sequence (leftover label still fires correctly).
- The TDZ/self-reference trap in §7.3 and the `setMode`-randomizes-the-problem trap in §6c are the two easiest ways to introduce a *phantom* bug or a *false positive* test result on this file. Watch for both.

---

## 9. Open items / possible next steps (not yet actioned — no user decision made)

- Animation pacing/speed — no final feedback given yet on whether current timings (750ms step-lock, ~550-800ms internal delays) feel right to the child.
- Idea floated but not implemented: having the green carry-bar visually *travel* from the ones zone to the tens zone (currently it fades in place / reappears, doesn't move across the layout).
- Thai TTS narration (would hook into Google Cloud TTS, already used elsewhere in this person's other project, HSK Journey) — discussed early, not built.
- No progress tracking / persistence — resets every page load. Not requested, but flag if scope grows.
- No colorblindness check on the two-tone orange/blue pairs — currently relies on shade + legend-number as a backup, not verified against actual color-vision-deficiency simulation.
- Only 6 lessons exist; no lesson beyond 2-digit ± 2-digit with single carry/borrow (e.g. no double-carry, no 3-digit).

---

## 10. Quick reference: key function names

| Function | Purpose |
|---|---|
| `buildSteps()` | Generates the Watch-mode step array for current `state.a/b/key` |
| `nextStep()` | Advances one step: narration, highlight, fx, pulse, debounce |
| `renderConcrete()` | Initial (pre-step) manipulative render, used by both Watch and Try |
| `fillOnes(x,y)` | Renders x+y grouped ones dots |
| `bundleToBar(oSum)` | Renders the full-frame → green bar bundling visual (carry) |
| `borrowDown()` | Renders the blue bar → broken-into-dots visual (borrow), then updates label+legend |
| `writeSixteen(oSum)` | Writes a full two-digit sum aligned into the answer boxes |
| `splitCarry()` | Flies the carried digit up, fades the bar, updates leftover label (addition) |
| `removeOnes(count)` | Animates removing `count` dots (subtraction) |
| `showTensBarsAdd/Sub(...)` | Renders the tens-place bar visual for each operation |
| `focusCol(col)` / `clearFocus()` | Column highlight in the vertical sum |
| `showLegend(a,b)` / `hideLegend()` | Numeric legend pills under the manipulative |
| `S()` | Returns the current language's string dictionary |
| `setLang(l)` / `setMode(m)` | Language / mode switches (⚠️ `setMode` randomizes the problem) |
| `renderRound()` vs `newRound()` | `renderRound` re-renders current state; `newRound` = `newProblem()` (randomizes) + `renderRound()` |
