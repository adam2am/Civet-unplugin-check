# Civet Sourcemap Accuracy: String Literals in Conditionals

## Overview

Civet's sourcemap generation for string literals within `if`/`else` blocks shows inaccuracies that affect tooling like IDEs and Svelte Language Tools. This document summarizes these issues and the findings from our test script (`sourcemap-accuracy-pr.civet`). The goal is to contribute fixes to improve Civet's sourcemaps.

## Key Findings & Bugs

String literals in conditional assignments exhibit specific mapping errors:

1.  **Indentation-Style `if/else`:**
    *   **`"InsideIf"` (example):** Maps with a consistent column offset (e.g., original Col 2 maps to Col 25, an offset of +23).
    *   **`"InsideElse"` (example):** Often results in a missing mapping (original position maps to `null`).
2.  **Bracketed-Style `if/else`:**
    *   **`"BracketedIf"` (example):** Maps with significant line and column offsets (e.g., original L4,C2 maps to L5,C27; +1 line, +25 cols).
    *   **`"BracketedElse"` (example):** Also severely mis-mapped (e.g., original L6,C2 maps to L3,C6).
3.  **Ternary Operators (`? :`)**
    *   String literals in ternary expressions (e.g., `check ? "Yes" : "No"`) appear to be mapped **correctly** by the original Civet `sourcemap.civet` logic, based on our targeted tests.

These issues are reproducible and demonstrable with the provided test script, which logs detailed VLQ segment decoding for analysis.

## Root Cause & Proposed Fix

The primary cause of these misalignments is the `colOffset` parameter passed by `generate.civet` to `SourceMap#updateSourceMap`. This `colOffset` already contains the error offset. Our fix within `updateSourceMap` involves:
1.  Calculating a `columnAdjustment` that subtracts this incoming `colOffset`'s effect from the source column (`srcCol`).
2.  For bracketed `if` statements, decrementing the `srcLine` by 1 to correct the line number.
3.  No special adjustment seems needed for ternary operators based on current test results.

## Test Script (`sourcemap-accuracy-pr.civet`)

*   Uses Mocha and `@jridgewell/trace-mapping`.
*   Contains focused scenarios for indented `if/else`, bracketed `if/else`, and ternary operators.
*   Asserts expected original Civet line/column, highlighting incorrect behavior until fixes are applied.
*   The script's verbosity (logging decoded VLQs, etc.) is intentional for debugging aid.

## Validation of Bugs

These are genuine Civet sourcemap bugs, confirmed by:
*   Consistent failure patterns across syntax variations.
*   Direct VLQ segment analysis.
*   Impact on runtime developer experience in tools consuming these sourcemaps.

## svelte2tsx Integration Strategy

*   **Short-term:** Implement verification in the `svelte2tsx` preprocessor to detect and log Civet mapping gaps. Augment sourcemap chaining to be more resilient to incomplete mappings from Civet.
*   **Long-term:** Contribute the refined patch for `updateSourceMap` to the main Civet project. Expand test coverage for more complex conditional structures.

## Conclusion & Path Forward

The `colOffset` parameter passed to `updateSourceMap` is the main source of error for string literals in `if/else` contexts.
The fix involves adjusting `srcCol` to negate this incoming offset and correcting `srcLine` for bracketed `if`. Ternary operators seem correctly mapped by original Civet.
The `sourcemap-accuracy-pr.civet` test script validates these issues.
Next steps:
1.  Refine and apply the proposed patch to `source/sourcemap.civet`.
2.  Ensure all "StringLiteral" tests in `sourcemap-accuracy-pr.civet` pass.
3.  Prepare a Pull Request for Civet with the fix and the test script.
