# Issue Resolution: Document.translate_row_col_to_index does not clamp negative rows to zero

## Metadata
- **Target Repository**: `prompt-toolkit/python-prompt-toolkit`
- **Private Repository**: `rajanjava/prompt-toolkit-private`
- **Upstream Issue**: [#2097: Document.translate_row_col_to_index does not clamp negative rows to zero](https://github.com/prompt-toolkit/python-prompt-toolkit/issues/2097)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
The docstring of `Document.translate_row_col_to_index(self, row: int, col: int) -> int` explicitly states:
> "Negative row/col values are turned into zero."

However, the implementation was:
```python
try:
    result = self._line_start_indexes[row]
    line = self.lines[row]
except IndexError:
    if row < 0:
        result = self._line_start_indexes[0]
        line = self.lines[0]
    else:
        result = self._line_start_indexes[-1]
        line = self.lines[-1]
```
Because Python lists support negative indexing, `self._line_start_indexes[-1]` accesses the last line of the document instead of raising `IndexError`. An `IndexError` was only raised when `row < -len(self.lines)`.

As a consequence:
- For a 3-line document `"one\ntwo\nthree"`, passing `row = -1` selected line 2 (`"three"`, index `8`), and `row = -2` selected line 1 (`"two"`, index `4`).
- Only when `row <= -4` did `IndexError` occur and trigger the `if row < 0:` branch to return `0`.
- This broke the documented contract that negative row indices are clamped to row `0`.

---

## Solution Approach
1. **Align Boundary Clamping Symmetry:** `Document.translate_row_col_to_index(row, col)` clamped oversized row indices to the last line, but when given a negative `row` (e.g., `-1`), Python's negative list indexing wrapped around to the end of the document instead of clamping to row `0`.
2. **Explicit Lower-Bound Clamping:** Added `row = max(0, row)` before indexing into `self.lines` in `src/prompt_toolkit/document.py`, ensuring symmetric clamping at both document boundaries (`0` and `line_count - 1`).
3. **TDD & Regression Safety:** Added `test_translate_row_col_to_index` cases for negative rows in `tests/test_document.py` verifying clamping to index `0`.

## Solution
Clamp `row` to non-negative values upfront via `row = max(0, row)` before indexing into `self._line_start_indexes` and `self.lines`:
```python
row = max(0, row)
try:
    result = self._line_start_indexes[row]
    line = self.lines[row]
except IndexError:
    result = self._line_start_indexes[-1]
    line = self.lines[-1]

result += max(0, min(col, len(line)))
result = max(0, min(result, len(self.text)))
return result
```

---

## Code Changes
```diff
diff --git a/src/prompt_toolkit/document.py b/src/prompt_toolkit/document.py
index 003a785..964fda9 100644
--- a/src/prompt_toolkit/document.py
+++ b/src/prompt_toolkit/document.py
@@ -325,16 +325,13 @@ class Document:
 
         Negative row/col values are turned into zero.
         """
+        row = max(0, row)
         try:
             result = self._line_start_indexes[row]
             line = self.lines[row]
         except IndexError:
-            if row < 0:
-                result = self._line_start_indexes[0]
-                line = self.lines[0]
-            else:
-                result = self._line_start_indexes[-1]
-                line = self.lines[-1]
+            result = self._line_start_indexes[-1]
+            line = self.lines[-1]
 
         result += max(0, min(col, len(line)))
```

---

## Verification
- Added reproduction test `test_translate_row_col_to_index` in `tests/test_document.py` covering valid coordinates, negative row clamping, negative column clamping, and out-of-range rows.
- Verified reproduction failure prior to fix:
  `AssertionError: assert 8 == 0 (where 8 = translate_row_col_to_index(-1, 0))`
- Verified test suite pass after fix:
  `13 passed in 0.13s`
- Verified `ruff check` on modified files:
  `All checks passed!`
