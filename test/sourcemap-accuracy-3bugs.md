# Civet Sourcemap Accuracy Issues & Proposed Test Case

## Summary

This document outlines several sourcemap generation inaccuracies identified in Civet, particularly concerning string literals within conditional statements (`if`/`else`). These inaccuracies can impact the developer experience in tools that consume these sourcemaps, such as IDEs for features like "Go to Definition" or "Hover", and preprocessors like `svelte2tsx` used in Svelte Language Tools.

We have developed a test script (`sourcemap-accuracy-pr.civet` - to be provided alongside this) that demonstrates these issues. The goal is to contribute this understanding and the test cases to help improve Civet\'s sourcemap generation.

## Key Findings: Bugs in Literal Mapping in Conditionals

Our investigation (detailed in [Log11 of our internal notes](sourcemap-accuracy.md#log11-all-microtests-passed-definitive-understanding-of-conditional-mapping-bugs) and subsequent logs) has pinpointed three distinct, reproducible bugs related to how string literals are mapped when assigned within conditional blocks:

**Bug 1: Missing Literal Map (Assignments to intermediate variables)**

*   **Context:** When a string literal is assigned to an intermediate/temporary variable (e.g., `ref = "value"`) within an `if` or `else` block (both indentation-style and bracketed-style).
*   **Behavior:** Civet fails to generate a mapping segment with *any* original source information for the literal itself. The sourcemap segment for the literal often only carries a generated column delta.
*   **Result:** When querying the sourcemap for the literal\'s original position, tools may report "source is null/undefined" or map it incorrectly.
*   **Example Snippet (Illustrative):**
    ```civet
    condition := true
    message := if condition
      intermediate := "AssignedInIf" // This literal may not be mapped
      intermediate
    else
      "AssignedInElse"
    ```

**Bug 2: Incomplete/Borrowed Literal Map (Direct assignments in `IF` and Bracketed `ELSE`)**

*   **Context:** When a string literal is directly assigned to a non-temporary variable (e.g., `result = "value"`) within:
    *   An `IF` branch (any style: indentation or bracketed `if { ... }`).
    *   A bracketed `ELSE { ... }` branch.
*   **Behavior:** Civet *does* emit a mapping segment for the line containing the literal. However, this segment may be incomplete (lacking full original source information like `sourceColumnDelta`) or it maps the literal to an incorrect original column.
*   **Result:** The literal effectively "borrows" its original source line (which is usually correct) but often maps to the original source column of a preceding token (like the `=` operator or the space after it). This leads to a predictable column mismatch.
*   **Example Snippet (Illustrative for `if`):**
    ```civet
    condition := true
    outputVar := ""
    if condition
      outputVar = "InsideIf" // Maps to column of '=' instead of '"'
    ```
*   **Example Snippet (Illustrative for bracketed `else`):**
    ```civet
    condition := false
    outputVar := ""
    if condition {
      outputVar = "InsideIf"
    } else {
      outputVar = "InsideElse" // Maps to column of '=' instead of '"'
    }
    ```

**Bug 3: Severely Misaligned Literal Map (Direct assignments in Indentation-Style `ELSE` only)**

*   **Context:** When a string literal is directly assigned to a non-temporary variable specifically within an *indentation-style* `ELSE` branch.
*   **Behavior:** The mapping for the literal is severely misaligned.
*   **Result:** It incorrectly points to the original source line and column of the `else` keyword itself, not the line/column of the literal or even the assignment statement containing the literal.
*   **Example Snippet (Illustrative):**
    ```civet
    condition := false
    outputVar := ""
    if condition
      outputVar = "InsideIf"
    else
      outputVar = "ElseBranchValue" // Maps to the line/col of 'else'
    ```

## Proposed Test Script

We will provide `sourcemap-accuracy-pr.civet`, a dedicated test script using Mocha and `@jridgewell/trace-mapping`. This script contains minimal, focused test cases (`runScenario`) for each of the bugs described above. Each test:
*   Defines a small Civet code snippet.
*   Specifies target tokens in the generated TypeScript.
*   Asserts the expected original Civet line and column.
*   The test expectations are set to highlight the current incorrect behavior, thus failing until the bugs are addressed.
*   It also logs detailed decoded VLQ segments for failing mappings, aiding in debugging.

## Potential Areas for Investigation in Civet

Based on our analysis of `source/sourcemap.civet` 
these issues likely from how `generate.civet` (the code generator) calls `SourceMap#updateSourceMap`. Specifically:
*   For **Bug 1**, `updateSourceMap` might not be called at all for the literal, or called without a valid `inputPos`.
*   For **Bug 2**, `updateSourceMap` might be called with an `inputPos` that points to the assignment operator rather than the literal itself, or the logic for deriving original column might be subtly off for these cases.
*   For **Bug 3**, the `inputPos` passed to `updateSourceMap` for the literal seems to be incorrectly derived from the `else` keyword.

We believe that addressing these points will significantly improve the accuracy of Civet\'s sourcemaps, benefiting the wider ecosystem of tools that rely on them.

We are happy to provide further details or assist in refining these test cases.

## Why These Are Real Bugs, Not Testing Issues

1. **Consistency Across Varied Syntax**: The same patterns of mapping failures occur across different syntactic structures (if/else with and without brackets, direct assignments vs. intermediate variables).

2. **VLQ Analysis Confirmation**: The decoded VLQ segments from the sourcemap directly confirm the issues:
   - For Bug 2, segments exist for the line containing the literal, but they map the literal to an incorrect original column, often lacking a specific `sourceColumnDelta` and instead borrowing the column from a prior token.
   - For Bugs 1 and 3, segments exist but target incorrect source positions (Bug 1 might entirely lack source info for the literal itself, Bug 3 maps to the `else` keyword).

3. **Stable Test Methodology**: Our test harness:
   - Precisely targets specific positions in generated TypeScript
   - Verifies sourcemap structure (sources array, sourcesContent)
   - Decodes and analyzes the actual VLQ mapping data
   - Uses minimal test cases focused on specific patterns

4. **Runtime Integration Impact**: These mapping issues directly affect developer experience in tools like:
   - IDE features for "jump to definition" and hover
   - Svelte Language Tools when using Civet in `<script lang="civet">` tags
   - Sourcemap consumers for debugging and error reporting

### Angles

1. **Comprehensive Testing**:
   - Expand test suite to include complex nested conditionals, loops, and other structures
   - Create end-to-end tests with Svelte components using Civet
   - Verify accuracy of position mapping during hover, definition lookup, etc.

2. **Documentation**:
   - Document known limitations and workarounds

This dual approach allows us to deliver a workable solution in the short term while pursuing proper fixes in the core Civet compiler.

## Hypothesis

- The core issue is that the `colOffset` parameter passed into `updateSourceMap` already includes the incorrect offset for string literals in conditional contexts, causing all subsequent mappings to be misaligned.
- Fixing this requires subtracting the raw `colOffset` from the computed source column and applying precise line adjustments for bracketed conditionals.
- The `sourcemap-accuracy-pr.civet` test script will serve as a regression suite, explicitly failing until these fixes are applied.
- In the short term, integrate verification and fallback steps in the Svelte preprocessing pipeline to mitigate mapping gaps.
- Long term, contribute the upstream patch to Civet's `updateSourceMap` logic, ensuring accurate sourcemaps for conditional string literals and improving developer tooling across the ecosystem.
