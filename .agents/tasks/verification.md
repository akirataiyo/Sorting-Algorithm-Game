# Verification note (iteration 1, first build)

No `review.json` was present, so this iteration built the whole game. Deliverables: `d:\Kiro Game\index.html` (single file, one inline `<script>`, inline CSS, no external resources) and `d:\Kiro Game\README.md` (Thai).
Environment: Windows, Node v24.13.1 (`node --version`), Microsoft Edge (headless) at `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`.

All test scripts lived in a temp folder `d:\Kiro Game\_tmp_verify\` and were **deleted afterwards**. Only index.html and README.md remain (plus `.agents/`). There is no git repo in the workspace, so nothing was committed.

## 1. Static and syntax check: `node _tmp_verify\check.js`
- Exactly 1 inline `<script>` with no `src`. No `<link rel=stylesheet>`. No `http(s)` URLs in `src=`, `href=` or `url(`. No `console.*` calls in the game script. The `// ===== SORTING ALGORITHMS =====` … `// ===== END SORTING ALGORITHMS =====` section exists.
- Extracted the script to `os.tmpdir()` and ran `node --check` on it: exit 0. The temp file was removed.
- Result: **ALL CHECKS PASSED (7/7)**.

## 2. Algorithm and logic tests: `node --test _tmp_verify/sort.test.js _tmp_verify/logic.test.js`
These tests load the real code from index.html. They take the text between `// ===== UTILS =====` and `// ===== END GAME LOGIC (PURE) =====` and run it in `node:vm`. Nothing is re-implemented.
- sort.test.js: 824 inputs × 4 algorithms = **3296 runs**. Inputs cover n = 0..12: random, small-range duplicates, ascending, descending, all-equal, plus every shape generator. For each run the test asserts:
  - the output equals the numeric sort
  - replaying the step list with `applyStep` on a copy of the input gives exactly the algorithm's own output
  - the input is not mutated
  - comparisons, swaps and writes equal the counts of compare, swap and shift+insert steps; `countOps` totals are consistent
  - the last step is the only `done`; no swap has i==j; markSorted covers every index
- Specific checks:
  - Bubble on sorted input: n−1 comparisons, 0 swaps (early exit). On reversed input: n(n−1)/2 swaps.
  - Selection: always n(n−1)/2 comparisons, ≤ n−1 swaps, and `minFound` is the first minimum.
  - Insertion: stability verified by replaying the steps on `{v, idx}` objects. Sorted input gives n−1 comparisons and 0 writes.
  - Quick: sorted input gives n(n−1)/2 comparisons, and the pivot is always `hi`.
- logic.test.js simulates a training-mode player on every algorithm × 7 shapes × n = 1..12 × 6 seeds (**2016 games, 25061 decision points**). At every decision it checks that:
  - the live array equals the decision's snapshot
  - `expected` matches an independent check against the data: a[j] > a[j+1]; first min; stable insertion slot; < pivot; pivot index = lo + count(< pivot)
  - all **68176 wrong answers** return `ok:false` with a Thai message
  - `hintFor` points at the expected answer
  - the final array is sorted and equals the algorithm's output
- Other logic checks:
  - `minSwapsToSort` matches a BFS shortest path on all 873 permutations with n ≤ 6, and the cycle lengths sum correctly
  - `raceBudget` ≥ min swaps
  - Predict: winners are the true minimum; Insertion wins `total` on sorted data; Selection wins `moves` on reversed data; 30 seeds × 10 rounds generate valid data
  - All LEVELS generate unsorted data with decisions
  - Star, time-bonus, rank and streak rules
- Result: **12 tests, 12 pass, 0 fail**.

## 3. Real browser run: `node _tmp_verify\browser.js` (headless Edge + Chrome DevTools Protocol)
The script loads `file:///…/index.html` and records every `Runtime.exceptionThrown`, every console error/warning and every `Log` error. It then plays the game through the real DOM by clicking buttons:
- Title screen renders. **All 9 training levels** completed with correct play via UI clicks (swap/pass buttons, bottle clicks, insertion slots, < pivot buttons), each giving "ผ่านด่าน N" with 3 stars. Level 9 (4 stages) took about 31 s.
- 3 wrong answers lead to the fail screen. The hint highlights the correct button, and the keyboard shortcut answers correctly.
- After a page reload, progress persists in localStorage (unlocked 9, 3 stars on every level).
- Race:
  - Win using minimum swaps; the player's swap count equals the reported minimum.
  - Loss when the AI (Insertion) finishes first.
  - Loss when the swap budget runs out.
  - Selecting a bottle then pressing Esc cancels the selection.
  - Leaving mid-race stops the AI timer.
- Predict: 10 rounds, each showing a 4-bar chart, then the final screen with a rank title. Tied rounds were seen and accepted.
- Codex: every algorithm's demo, stepped to the end, finishes sorted and says "เรียงเสร็จ".
- The mute toggle flips `aria-pressed`.
- No horizontal overflow at 360 px on title, menu, insertion n=8 with slots, quick n=8, race n=10, predict n=12 plus reveal, codex and how-to. No overflow at 1366 px on race.
- **0 exceptions / console errors / log errors** across the whole run.
- Result: **ALL 25 BROWSER CHECKS PASSED**.

## Fixes made during verification
- `SHAPES.nearlySortedDup` with n=1 could swap with an undefined slot. It now only swaps when n ≥ 2. Caught by the test, though gameplay never uses n=1.
- Training view: while waiting for a decision, the previous compare highlight was still shown next to the current one (seen in a screenshot). The UI now shows only the current decision's highlight.
- Predict choices: disabled buttons no longer fade to 50% opacity after the reveal (contrast).

## Not verified
- A manual check with a real screen reader. ARIA labels and roles were added, but WCAG conformance needs manual assistive-technology testing.
- Sound output. WebAudio is created lazily and wrapped in try/catch; no errors were logged, but audio was not listened to.
- The live GitHub Pages deploy. Only `file://` was tested, and the file has no external dependencies.
