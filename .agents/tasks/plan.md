# Implementation Plan: "ร้านยาของนักเล่นแร่แปรธาตุ" (Alchemist's Potion Shelf)

## Context (from exploration)

- `d:\Kiro Game` is empty: no git repo, no package.json, no docs. Greenfield. (Write permission was granted to the user account via icacls; verified.)
- Tooling: Node v24.13.1 (`node:test`, `node:vm`, `node --check`), git 2.56. Windows / PowerShell.
- Deliverables: `d:\Kiro Game\index.html` (ONE self-contained file: inline CSS + inline JS, no CDN, no network fonts, no build step, works from `file://`) and `d:\Kiro Game\README.md` (Thai).
- Test-only helpers live in `d:\Kiro Game\tests\` (not needed for hosting; README says only index.html must be uploaded).
- Review gate (unchanged): `d:\Kiro Game\.agents\tasks\review.json`, `{"verdict": "APPROVED"}`.
- Always use absolute paths under `d:\Kiro Game`.

## Design decisions

1. **One file, ordered sections inside one classic `<script>`** (no ES modules — they fail from `file://` in some browsers). Section banners, in order:
   `// ===== UTILS =====`, `// ===== SORTING ALGORITHMS =====` … `// ===== END SORTING ALGORITHMS =====`,
   `// ===== GAME LOGIC (PURE) =====` … `// ===== END GAME LOGIC (PURE) =====`, `// ===== STORAGE =====`,
   `// ===== AUDIO =====`, `// ===== RENDER: SHELF =====`, `// ===== SCREENS / NAV =====`, `// ===== MODE 1: TRAINING =====`,
   `// ===== MODE 2: RACE =====`, `// ===== MODE 3: PREDICT =====`, `// ===== CODEX =====`, `// ===== INIT =====`.
   Rationale: UTILS + the two PURE regions use no DOM/`window`, so tests extract the text from `// ===== UTILS =====` to `// ===== END GAME LOGIC (PURE) =====` and evaluate it in `node:vm`. No `module.exports` in the shipped file. INIT runs on `DOMContentLoaded`.
2. **Values**: integers 1–99. Bottle height ∝ value; hue mapped from value; the number is always printed on the bottle (never color-only).
3. **Race mode uses distinct values** so minimum swaps = n − (#cycles) is exact. Training/Predict may use duplicates.
4. **Seeded RNG** (`mulberry32(seed)`) for all data generation, so tests are deterministic and levels are reproducible per attempt seed.

## Shared instrumented sorting module (the engine)

Every algorithm is `function bubbleSort(input)` / `selectionSort` / `insertionSort` / `quickSort` → returns
`{ algo, input: copy, sorted: array, steps: Step[], comparisons, swaps, writes }`.
It copies `input` (never mutates it), sorts the copy, and pushes a step object every time it acts. Counters are incremented exactly where the step is pushed, so counting step types always equals the counters (tested).

Step schema (all indices absolute):

| type | fields | meaning | counter |
|---|---|---|---|
| `compare` | `i, j` | compare a[i] with a[j] (Insertion: `j` = index of the hole/key position, see below) | comparisons++ |
| `swap` | `i, j` | exchange a[i], a[j] (only recorded when i≠j) | swaps++ |
| `key` | `i, value` | Insertion: lift a[i] as key (hole at i) | – |
| `shift` | `from, to` | Insertion: a[to] = a[from], to = from+1 | writes++ |
| `insert` | `i, value` | Insertion: drop key into a[i] (only when position changed) | writes++ |
| `pivot` | `i, lo, hi` | Quick: pivot = a[hi] for partition [lo..hi] | – |
| `markSorted` | `from, to` | elements from..to are final (single index → from===to) | – |
| `passStart` | `pass, end` | Bubble: this pass compares j = 0..end−1 | – |
| `selectStart` | `i` | Selection: find min of [i..n−1] | – |
| `minFound` | `i, min` | Selection: chosen (first) minimum index for position i | – |
| `done` | – | finished (always last step) | – |

Shared helpers in the same section:
- `applyStep(arr, step, ctx)` — mutates an array for one step (swap; key stores `ctx.key = value`; shift copies; insert writes `ctx.key`). Used by race animation, codex demo, training auto-play AND tests (single replay source of truth). For display, the renderer shows the lifted key bottle above the shelf while a hole exists.
- `replaySteps(input, steps)` → final array.
- `countOps(result)` → `{ comparisons, swaps, writes, moves: swaps + writes, total: comparisons + swaps + writes }`.
- `ALGORITHMS = { bubble: {fn, nameTh, nameEn, ...}, selection, insertion, quick }` registry used by every mode.

Algorithm specifics (each with detailed Thai comments: idea, why each step is recorded, best/avg/worst Big-O, space, stability, how modes consume the steps):

- **Bubble Sort**: for `pass = 0..n-2`: push `passStart{pass, end:n-1-pass}`; `swapped=false`; for `j = 0..n-2-pass`: `compare(j,j+1)`; if `a[j] > a[j+1]` (strict) → `swap(j,j+1)`, swapped=true. After the pass: if `!swapped` → **early exit**: push `markSorted{from:0,to:n-1-pass}` and stop; else push `markSorted{from:n-1-pass,to:n-1-pass}`. If all passes ran, push `markSorted{0,0}`. n ≤ 1: `markSorted` (if n=1) + `done`. Best O(n) (sorted, thanks to early exit), avg/worst O(n²), stable.
- **Selection Sort**: for `i = 0..n-2`: `selectStart(i)`; `min=i`; for `j=i+1..n-1`: `compare(min, j)`; if `a[j] < a[min]` (strict → **first minimum on ties**) `min=j`. Push `minFound{i,min}`; if `min≠i` push `swap(i,min)`; push `markSorted{i,i}`. End: `markSorted{n-1,n-1}`. Always n(n−1)/2 comparisons, ≤ n−1 swaps, not stable (comment gives an example).
- **Insertion Sort**: for `i = 1..n-1`: `key(i, a[i])`; `j=i-1`; while `j >= 0`: push `compare{i:j, j:j+1}` (a[j] vs the key, whose hole is at j+1); if `a[j] > key` (strict → **stable**, key stays after equals) push `shift{from:j,to:j+1}`, `j--`; else break. `p=j+1`; if `p≠i` push `insert{i:p,value:key}` (no write when it stays). Then `markSorted{from:0,to:i}`. When j reaches −1 there is no extra compare. Best O(n) on sorted, worst O(n²), stable. n=1 → `markSorted{0,0}`.
- **Quick Sort (Lomuto, last element pivot)**: recursive `qs(lo,hi)`: if `lo>hi` return; if `lo===hi` push `markSorted{lo,lo}`, return. Push `pivot{i:hi,lo,hi}`; `p=a[hi]`; `i=lo-1`; for `j=lo..hi-1`: `compare(j,hi)`; if `a[j] < p` (strict; equals go right) → `i++`; if `i≠j` push `swap(i,j)`. Pivot placement `q=i+1`: if `q≠hi` push `swap(q,hi)`; push `markSorted{q,q}`; recurse `qs(lo,q-1)` then `qs(q+1,hi)`. Comment: avg O(n log n), worst O(n²) on sorted/reversed/all-equal input with last pivot; not stable; n ≤ 12 so recursion depth is safe.

How the step list drives each mode:
- **Training**: `buildDecisions(algo, input)` (pure) runs the algorithm, replays its steps with `applyStep`, and converts them into an ordered list of *decision points*, each storing the step-index range to auto-apply after a correct answer and an array snapshot. Player input is validated against the current decision; on success the steps up to the next decision are animated with `applyStep`.
- **Race**: AI shelf plays the chosen algorithm's `steps` one per tick with `applyStep`, highlighting `compare` / `swap` / `shift` / `pivot`; AI counters increase as steps play.
- **Predict**: runs all four `fn`s on the round data and reads `countOps` for the chart and the answer.
- **Codex**: mini demo replays steps on a 6-element sample.

## Training validation rules (pure, GAME LOGIC section)

`buildDecisions(algo, input)` returns `{ result, decisions: [{kind, expected, ctx, fromStep, toStep}] }` where `ctx` holds whatever the UI/messages need (pair values, pivot value, region, key value, snapshot array). `validate(decision, answer)` → `{ok, message}` (Thai). `hintFor(decision)` → Thai text of the algorithm's next move + the index/button to highlight. `explainStep(decision)` → Thai "ขั้นตอนตอนนี้: …" panel text.

- **Bubble** — one decision per `compare(j,j+1)`: `kind:'swapOrPass'`, `expected: 'swap'` iff the following step is `swap(j,j+1)`, else `'pass'`. UI highlights pair (j, j+1) and dims the sorted tail (> end). Wrong: swapped when left ≤ right → "Bubble Sort สลับเมื่อซ้ายมากกว่าขวาเท่านั้น (x ≤ y จึงไม่ต้องสลับ)"; passed when left > right → "x มากกว่า y ต้องสลับ เพื่อดันค่ามากไปทางขวา". When a pass finishes without swaps the panel says "รอบนี้ไม่มีการสลับเลย แปลว่าเรียงแล้ว — Bubble Sort หยุดก่อนได้ (early exit)" and the stage ends (follows the step list → consistent with the algorithm and counters).
- **Selection** — one decision per `selectStart(i)`: `kind:'pickIndex'`, valid clicks i..n−1, `expected = minFound.min`. Wrong: larger value → "ขวดนี้ไม่ใช่ค่าน้อยที่สุดในส่วนที่ยังไม่เรียง (มีค่า m ที่น้อยกว่า)"; equal value at later index → "มีค่าเท่ากันหลายขวด ให้เลือกขวดแรกที่เจอ (ซ้ายสุด) ตามที่ Selection Sort สแกน"; clicking the sorted zone → "ส่วนซ้ายเรียงเสร็จแล้ว เลือกเฉพาะส่วนที่ยังไม่เรียง". On success the scan's compare steps are applied (counter jumps by their count) then swap auto-plays, or the message "ค่าน้อยสุดอยู่ที่เดิมแล้ว ไม่ต้องสลับ" when min===i.
- **Insertion** — one decision per `key(i)`: `kind:'pickSlot'`, slots 0..i (slot p = "วางที่ตำแหน่ง p", slot i = "อยู่ที่เดิม"). `expected = insert.i` if an insert step exists for this key, else `i`. Wrong: slot left of/before an equal value → "Insertion Sort แบบ stable จะวางหลังค่าที่เท่ากัน ไม่แซงหน้า"; slot whose left neighbor... is smaller-than-required region (slot < expected, not equal case) → "ค่า x ทางซ้ายน้อยกว่าหรือเท่ากับ key แล้ว ต้องหยุดเลื่อนตรงนั้น"; slot > expected → "ยังมีค่ามากกว่า key อยู่ทางซ้าย ต้องเลื่อนต่อ". Success auto-plays compares/shifts/insert.
- **Quick (Lomuto)** — one decision per `compare(j,hi)`: `kind:'lessOrNot'`, `expected: snapshot[j] < pivotValue ? 'less' : 'notLess'`. Buttons "ไปฝั่ง < pivot" / "อยู่ฝั่ง ≥ pivot". After the last compare of a partition: decision `kind:'pickIndex'` (pivot placement) with `expected = q`, valid clicks lo..hi. UI marks pivot (★), region [lo..hi], tints the "< pivot" zone [lo..i]. Wrong: "ค่า x ไม่น้อยกว่า pivot p จึงอยู่ฝั่งขวา (Lomuto ใช้ < แบบเคร่งครัด ค่าเท่ากันไปขวา)" / "x น้อยกว่า pivot p ต้องย้ายไปฝั่งซ้าย"; pivot: "pivot ต้องไปอยู่ถัดจากกลุ่มที่น้อยกว่า pivot ทั้งหมด (ตำแหน่ง i+1)". Size ≤1 subarrays auto-mark sorted with a short message.

Scoring (pure, `scoreLevel`): correct decision +10; wrong → −1 heart, −5 points (floor 0), decision stays pending (retry); hint −15 points, counts as used. Completion time bonus `max(0, par − elapsedSec) * 2`, `par = 4 * totalDecisions`. Stars: 3 = 0 mistakes & 0 hints; 2 = mistakes + hints ≤ 2; 1 = completed. 0 hearts → fail screen (retry / menu).

Levels (`LEVELS`, each = list of stages sharing hearts/score):
1 Bubble n=5 random · 2 Bubble n=7 nearlySorted (shows early exit) · 3 Selection n=6 · 4 Selection n=8 manyDuplicates · 5 Insertion n=6 · 6 Insertion n=8 nearlySorted+duplicates · 7 Quick n=6 · 8 Quick n=8 · 9 Final "บททดสอบนักเล่นแร่แปรธาตุ": 4 stages n=5 Bubble → Selection → Insertion → Quick, shared 3 hearts.
Shape generators: `random`, `nearlySorted`, `reversed`, `sorted`, `manyDuplicates`, `distinctRandom` (generated data for training must not already be sorted, except where the stage says so).

## Other pure logic (same PURE section, tested)

- `minSwapsToSort(arr)` (distinct values): target positions from a sorted copy; count cycles; returns `{minSwaps: n − cycles, cycles: [[idx...]]}` (only cycles of length ≥2 listed). Thai comment explaining a k-cycle needs k−1 swaps.
- `raceBudget(arr)` = `minSwaps + 2`. `isSorted(arr)`.
- `evaluatePrediction(data, metric)` (metric `'total'` or `'moves'`) → per-algorithm `countOps`, `best` value, `winners` (all algos tied at the minimum). Correct if pick ∈ winners.
- `makePredictRounds(seed)` → 10 rounds: nearly sorted n=10 total; reversed n=8 total; random n=10 total; many duplicates n=10 total; already sorted n=12 total (Insertion wins, Quick worst); small random n=5 total; random n=12 total; reversed n=10 moves (Selection); random n=10 moves; nearly sorted n=10 moves. Each round has `shapeTh`, question text, and `explain(evaluation)` producing Thai text with real counts (Insertion ≈ O(n) on nearly sorted; Selection ≤ n−1 swaps; Quick last-pivot degrades on sorted).
- `rankTitle(score)`: ผู้ฝึกหัด / นักปรุงยา / นักเล่นแร่แปรธาตุ / ปรมาจารย์แห่งเวทเรียงลำดับ.

## Screens / state architecture

- Screens are `<section class="screen" id="screen-…" hidden>`: `title`, `howto`, `menu`, `levels`, `training`, `result` (shared complete/fail), `race-setup`, `race`, `predict`, `predict-result`, `codex`.
- `showScreen(id)`: calls `currentMode?.dispose()`, hides all, shows one, focuses its `<h2 tabindex="-1">`.
- Global `state = { screen, progress, muted }`; each mode object `{ start(opts), dispose() }`; timers go through a registry (`later(fn,ms)`, `every(fn,ms)`, `clearAllTimers()`) cleared on dispose.
- Storage key `alchemistShelf.v1` → `{ unlocked: 1, stars: {levelId: n}, best: {levelId: score}, predictBest: 0, raceWins: 0, muted: false }`. All `localStorage` access in try/catch with in-memory fallback. Completing level k unlocks k+1. Menu "รีเซ็ตความคืบหน้า" with `confirm()`.
- Audio: lazy `AudioContext` on first gesture; short oscillator beeps for compare/swap/correct/wrong/win/lose; header mute toggle (`aria-pressed`) persisted. Any audio failure is swallowed silently.

## UI / accessibility / responsive requirements

- `<html lang="th">`, viewport meta, `<title>`, font stack `"Sarabun","Noto Sans Thai","Leelawadee UI","Tahoma",sans-serif` (no network).
- Dark alchemy-shop theme via CSS variables (wooden shelf, glass bottles via CSS gradients/emoji only); text contrast ≥ 4.5:1; `:focus-visible` outlines; `prefers-reduced-motion` → instant changes.
- Bottles are `<button>`s with `aria-label="ขวดยาพลัง 42 ตำแหน่งที่ 3"` plus state words; states shown by color AND marker (✓ sorted, ▲ compared, ★ pivot, ⬆ key). Insertion slots are buttons "วางที่ตำแหน่ง k".
- `aria-live="polite"` region for feedback/explanations; visible counters (เปรียบเทียบ, สลับ, เขียน), hearts as text "♥ 2/3", score, timer.
- Keyboard: everything is a button; Training shortcuts S = สลับ, P = ไม่สลับ, H = ใบ้, also documented in how-to; Esc cancels race selection.
- Layout: flex/grid; bottle width via `clamp()`; n=12 fits 360px width; race shelves side by side ≥ 760px, stacked below.
- No uncaught errors: guard DOM lookups at init; no `console.error` calls.

## Mode details

- **Mode 1 ฝึกวิชา**: level grid (lock icon, stars, best score); level intro card (algorithm rule in Thai); play screen = shelf + explanation panel + action buttons per `kind` + counters + hint + quit; result screen with score, stars, mistakes, hints, player counters vs algorithm totals, buttons ด่านถัดไป / เล่นอีกครั้ง / เมนู.
- **Mode 2 แข่งกับเวทมนตร์**: setup: n (6/8/10), AI algorithm (สุ่ม/Bubble/Selection/Insertion/Quick), speed (ช้า 900ms / ปกติ 550ms / เร็ว 300ms per step). Both shelves get the SAME `distinctRandom` array (regenerate if sorted). Player clicks bottle A then B → swap (click A again deselects); "สลับแล้ว x / budget". 3-2-1 countdown, then AI ticks. Win: player sorted before AI's `done` and swaps ≤ budget. Lose: AI finishes first, or swaps reach budget while unsorted. Summary table: player swaps, minimum swaps with cycles (e.g. "วงจร (0→3→5)"), AI comparisons/swaps/writes, swaps each of the 4 algorithms would use on this data; Thai explanation. `raceWins` saved.
- **Mode 3 ทำนายเวทมนตร์**: 10 rounds; shelf + shape label + question ("อัลกอริทึมไหนใช้การดำเนินการรวม (เปรียบเทียบ+สลับ+เขียน) น้อยที่สุด?" or "…ย้ายข้อมูล (สลับ+เขียน) น้อยที่สุด?"); 4 choice buttons; reveal: CSS bar chart (labeled numbers) + table comparisons/swaps/writes/total, winners highlighted, explanation; +100 correct, +20 × streak bonus. Final: score, correct count, rank title, best saved.
- **สมุดตำรา Codex**: 4 algorithm tabs: idea, steps, Big-O best/avg/worst table, space, stable?, when to use; mini demo shelf replaying that algorithm's steps with เล่น/หยุด, ทีละขั้น, รีเซ็ต; disposed on exit.
- **วิธีเล่น**: rules for each mode, marker legend, keyboard shortcuts.

## Verification strategy (Node)

- `tests\extract.js`: reads `index.html`; `getScript()` returns the inline `<script>` text; `loadPure()` takes text from `// ===== UTILS =====` to `// ===== END GAME LOGIC (PURE) =====`, appends an expression collecting the named functions/constants into an object, runs it in `vm.createContext({})`, and returns it.
- `tests\check-script.js`: writes the full script to `os.tmpdir()`, runs `node --check` via `child_process.spawnSync(process.execPath, ['--check', file])`, deletes the temp file; also asserts exactly one inline `<script>` (no `src`), no `<link rel="stylesheet">`, and no `http://`/`https://` resource URLs in `src=`/`href=`/`url(`. Exits non-zero on any failure.
- `tests\sort.test.js` (node:test + assert): for each algorithm × input class (random, sorted, reversed, all-equal, many duplicates, nearly sorted) × length 0–12 × several seeds: `sorted` deep-equals `[...input].sort((a,b)=>a-b)`; `replaySteps(input, steps)` equals `sorted`; input not mutated; counters equal step-type counts; last step is `done`. Specific: Bubble on sorted → n−1 comparisons, 0 swaps; Selection swaps ≤ n−1 and `minFound.min` is the first minimum; Insertion stability (replay steps on objects `{v, idx}` and check equal values keep idx order) and sorted input → n−1 comparisons, 0 writes; Quick on sorted → n(n−1)/2 comparisons.
- `tests\logic.test.js`: a simulated player answering `expected` completes every algorithm on all input classes/lengths 1–12 with the final array sorted and equal to the algorithm output; each wrong answer gets `ok:false` with a non-empty Thai message; `hintFor` matches `expected`; `minSwapsToSort` equals brute-force BFS for all permutations of n ≤ 6; `raceBudget ≥ minSwaps`; `evaluatePrediction`: Insertion among winners for total on sorted/nearly sorted data, Selection among winners for moves on reversed; every LEVEL stage and every predict round generates valid data; star thresholds; `rankTitle` boundaries.
- Commands (cwd `d:\Kiro Game`): `node tests\check-script.js` and `node --test tests\`. Both must pass with zero failures.

---

# Implementation Plan

- [ ] 1. Scaffold `index.html`: full HTML skeleton (all screen sections, header with mute button, live region), complete CSS (theme, shelf/bottle styles, responsive, focus, reduced motion), and the ordered script section banners (including PURE markers); INIT shows the title screen. Create `tests\extract.js` and `tests\check-script.js`.
      Files: d:\Kiro Game\index.html, d:\Kiro Game\tests\extract.js, d:\Kiro Game\tests\check-script.js
      Verify: `node tests\check-script.js` exits 0 (cwd `d:\Kiro Game`).

- [ ] 2. Implement UTILS (copy, swap, mulberry32) and SORTING ALGORITHMS: the 4 instrumented algorithms exactly as specified, `applyStep`, `replaySteps`, `countOps`, `ALGORITHMS` registry, with detailed Thai comments. Write `tests\sort.test.js`.
      Files: d:\Kiro Game\index.html, d:\Kiro Game\tests\sort.test.js
      Verify: `node --test tests\` passes; `node tests\check-script.js` exits 0.

- [ ] 3. Implement GAME LOGIC (PURE): shape generators, `buildDecisions`/`validate`/`hintFor`/`explainStep` per algorithm with Thai messages, scoring/stars, `LEVELS`, `minSwapsToSort`, `raceBudget`, `isSorted`, `evaluatePrediction`, `makePredictRounds`, `rankTitle`. Write `tests\logic.test.js`. (Depends on 2.)
      Files: d:\Kiro Game\index.html, d:\Kiro Game\tests\logic.test.js
      Verify: `node --test tests\` all pass; check-script exits 0.

- [ ] 4. Implement STORAGE, AUDIO, RENDER: SHELF (accessible bottle buttons with number/height/hue/markers, highlight + swap/shift animation, key-lifted display, insertion slot buttons, reduced motion), SCREENS/NAV (`showScreen`, dispose, timer registry), plus title, how-to-play and main menu screens with navigation and reset progress.
      Files: d:\Kiro Game\index.html
      Verify: both commands pass; open index.html in a browser: title → วิธีเล่น → เมนู works, mute toggles and persists, no console errors.

- [ ] 5. Implement MODE 1 ฝึกวิชา: level select (locks/stars), level intro, play loop driven by `buildDecisions` (per-kind controls, auto-playing intermediate steps via `applyStep`), hearts/score/timer/counters/hint/explanations, keyboard shortcuts, multi-stage final level, result screen (complete/fail), unlock saved. (Depends on 3, 4.)
      Files: d:\Kiro Game\index.html
      Verify: both commands pass; browser: complete level 1 without mistakes → 3 stars, level 2 unlocked after reload; 3 wrong moves → fail screen; finish a Selection (duplicates), Insertion (duplicates) and Quick level.

- [ ] 6. Implement MODE 2 แข่งกับเวทมนตร์: setup, twin shelves with identical distinct data, click-two-to-swap with budget, countdown, AI tick animation from the chosen algorithm's steps with highlights/counters, win/lose rules, summary with min-swap cycles and per-algorithm swaps.
      Files: d:\Kiro Game\index.html
      Verify: both commands pass; browser: win a race on slow speed, lose one by letting AI finish, budget exhaustion ends the game, leaving mid-race stops the AI timer without errors.

- [ ] 7. Implement MODE 3 ทำนายเวทมนตร์: 10-round loop from `makePredictRounds`, choices, reveal with CSS bar chart + table from `evaluatePrediction`, Thai explanation with real counts, scoring/streak, final screen with rank title and saved best.
      Files: d:\Kiro Game\index.html
      Verify: both commands pass; browser: play all 10 rounds to the final screen; a tied winner is accepted.

- [ ] 8. Implement สมุดตำรา Codex: 4 algorithm tabs with idea, steps, Big-O best/avg/worst, space, stability, use cases, and a mini demo shelf replaying steps (play/pause/step/reset), disposed on exit.
      Files: d:\Kiro Game\index.html
      Verify: both commands pass; browser: each demo ends sorted and stops when leaving the screen.

- [ ] 9. Accessibility/responsive polish (aria-labels, focus management, live region text, contrast, 360px and 1366px layouts, reduced motion) and write the Thai `README.md` (game description, how sorting is used in each mode, algorithms, how to play, GitHub Pages steps: public repo → upload index.html → Settings > Pages > Deploy from a branch, `main` / root → `https://<user>.github.io/<repo>/`; Netlify Drop alternative; only index.html is needed).
      Files: d:\Kiro Game\index.html, d:\Kiro Game\README.md
      Verify: both commands pass; devtools responsive view at 360px and 1366px shows no horizontal overflow; keyboard-only path menu → training level works.

- [ ] 10. Final verification: run `node tests\check-script.js` and `node --test tests\` from `d:\Kiro Game`; confirm no external URLs/resources in index.html; play each mode start to finish with the console open (zero errors); remove temp files.
      Files: fixes as needed in d:\Kiro Game\index.html
      Verify: both commands exit 0; manual run-through clean.
