# Alchemist's Potion Shelf: single-file sorting game with four instrumented algorithms

This is a greenfield build of `index.html`, a Thai-language browser game where players sort potion bottles by the number printed on each one, plus a Thai `README.md`. Four instrumented sorting functions (Bubble with early exit, Selection, Insertion, and Lomuto Quick Sort) record a step list and counters. Those same functions drive every mode: Training has the player act as the algorithm one decision at a time, Race pits the player against an AI replaying the steps, Predict asks which algorithm wins on a given data shape, and the Codex is a step-through demo. The verification note reports 3296 algorithm runs, 2016 simulated training games, BFS-checked min-swaps, and a headless-Edge playthrough of every mode with zero console errors. Nothing in the code contradicts that evidence.

Watch for: the Predict round-10 explanation (nearly sorted, "moves" metric) talks about Insertion Sort being near O(n), but Insertion structurally loses that metric (confirmed). The 360 px layout was checked, but no real screen reader was used. The plan's `tests\` folder was deleted, so the regression suite can't be re-run.

**Verdict**: APPROVED

## High-level view

One registry, `ALGORITHMS`, holds the four `fn`s, and every mode calls through it. Training turns the step list into decision points (`buildDecisions`). Race and Codex replay it with the shared `applyStep`. Predict reads the real counters through `countOps`. No mode has its own copy of the sorting logic, so training validation can't drift from the algorithm the player is being taught.

Training validation follows each algorithm's real rules. In Bubble, the swap/pass answer comes from whether the next recorded step is a swap. The pass boundary shrinks by one each pass, and early exit simply ends the decision list. Selection's answer is `minFound.min`, which uses strict `<` and so is the first minimum on ties. Insertion's answer is the recorded `insert.i`, which uses strict `>` and so is the stable slot. Quick asks a `< pivot` question per compare, then a pivot-placement question at `q = i+1`. The placement lookup correctly skips the in-loop swap that can follow the last compare. The Thai error messages explain why an answer is wrong (a tie rule, stability, or a strict comparison), and this is where most of the learning happens.

Race uses distinct values, so the budget (`minSwaps + 2`, computed from cycle decomposition) is exact. The summary shows the cycles and all four algorithms' swap counts on the same data. Predict runs 10 fixed-shape rounds, accepts ties, and finishes with a score and rank title. The game outcomes are correct everywhere. The remaining gaps are pedagogical (one misleading explanation) and process-related (deleted tests).

<details>
<summary>Issues (4)</summary>

1. **Predict round-10 explanation misleads** (confirmed, non-blocking): on nearly sorted data scored by moves, Insertion uses one shift per inversion plus one insert per key, so it always loses to Bubble/Selection. Yet the `nearlySorted` insight in `explainPrediction` praises Insertion and mentions Selection's comparisons, which don't count toward this metric. The end-of-game lesson ("เกือบเรียง → Insertion") reinforces the same idea. Fix: add a metric-aware insight for `nearlySorted` + `moves` (for example, "นับเฉพาะการย้าย: Insertion ต้องเลื่อนและวาง key จึงเขียนมากกว่าการสลับคู่ติดกันครั้งเดียวของ Bubble/Selection").
2. **Regression tests removed** (confirmed, non-blocking): the plan put `tests\extract.js`, `check-script.js`, `sort.test.js`, and `logic.test.js` in the workspace. The coder ran them from `_tmp_verify\` and then deleted them, so the next iteration can't re-run them. Fix: restore them under `tests\`. They aren't needed for hosting.
3. **README has no slot for the submission link** (likely, non-blocking): the assignment is graded by a link, but the README only gives a `https://<user>.github.io/<repo>/` template. Fix: add a "ลิงก์เกม:" line to fill in after deployment.
4. **Quick Sort space understated in Codex** (confirmed, cosmetic): `ALGORITHMS.quick.space` is `O(log n)`, but last-element Lomuto needs O(n) stack on sorted input, which the game itself demonstrates. Fix: change the value to `'O(log n)–O(n)'`.

</details>

<details>
<summary>Details</summary>

### Instrumented algorithms and the shared step list

Every counter is incremented on the same line as its `steps.push`, so the counter totals and the step counts can't diverge. Insertion emits `insert` only when the key actually moved, so already-sorted input gives n−1 comparisons and 0 writes. Predict's "sorted" round is therefore a deliberate Bubble/Insertion tie, and the explanation text says so.

The Thai comment blocks above each function cover the idea, how strict the comparison is and how that affects stability, best/average/worst Big-O, space, and how Training uses the steps. The one inaccuracy (confirmed, cosmetic) is that `ALGORITHMS.quick.space` is `O(log n)` while the comment says `O(log n)–O(n)`. The Codex table shows the registry value, so the O(n) worst-case stack depth on sorted input never appears in the game.

### Training decision mapping

```
steps:  pivot  cmp  cmp  [swap]  cmp(last)  [swap i,j]  [swap q,hi]  markSorted ...
dec:           LoN  LoN          LoN                    ^ pickIndex(q) placed here
```

For Quick, `part.left` counts down from `hi-lo`. When it reaches 0, `placeAt` is set to the step after the last compare, plus one more if that compare triggers an `i≠j` swap. That way the pivot-placement question comes before `swap(q,hi)` (or before `markSorted` when `q===hi`). This is the step most likely to go out of sync, and the verification note's independent `lo + count(<pivot)` check covers it.

Clicks on bottles outside the active region aren't disabled. That covers the sorted prefix in Selection and anything outside `[lo..hi]` during pivot placement. Such a click costs a heart and gets a Thai explanation. The plan does this deliberately, treating a click outside the region as a misunderstanding of the algorithm, not a misclick.

### Predict explanations vs. the metric

`evaluatePrediction` is correct, and winners always come from real counts. `explainPrediction` picks its insight text by data shape only, not by shape and metric together. Round 10 is `nearlySorted` with metric `moves`. With k disjoint adjacent inversions, Bubble and Selection each use k swaps. Insertion uses k shifts plus k inserts, which is 2k writes. Insertion can't win this round, yet the text the player reads after losing says Insertion "เลื่อนขวดเพียงไม่กี่ครั้ง … ใกล้เคียง O(n)". The closing lesson "ข้อมูลเกือบเรียง → Insertion Sort" repeats this without qualification. The game's results are correct. The explanation teaches the wrong takeaway for this metric.

### Single-file, accessibility, responsive

The file has one inline `<script>`, inline CSS, no `<link>`/`@import`/remote `src`, and a font stack with no network fonts. Every bottle is a `<button>` or `role="img"` with an `aria-label` like `ขวดยาพลัง N ตำแหน่งที่ k (สถานะ…)`, and the number and index are printed on it. State is shown with a glyph (▲ ✓ ★ ⬆ ◆ ●) as well as color. Feedback, explanations, and the countdown are in live regions. Training has S/P/L/R/H shortcuts, Race has Esc, and Codex tabs use arrow keys. The verification run found no 360 px overflow. A real screen reader wasn't used, so full WCAG conformance still needs manual assistive-technology testing.

### README

The README covers the four algorithms and where they live in the code, how sorting drives each mode, controls and markers, GitHub Pages steps (Public repo → upload `index.html` → Settings → Pages → `main` / root → URL pattern, with no login required to view), and Netlify Drop as an alternative. The only gap is that there's no place to record the actual deployed link.

</details>

<details>
<summary>File map</summary>

- `index.html`: the whole game. Inline CSS, and a script split into UTILS / SORTING ALGORITHMS / GAME LOGIC (PURE) / STORAGE / AUDIO / RENDER / SCREENS / MODE 1–3 / CODEX / INIT.
- `README.md`: Thai game description, how sorting is used in each mode, how to play, and GitHub Pages and Netlify hosting steps.

There's no git repo, so the full files are the diff.

</details>
