# Implementation Plan: SQL-to-Pandas Translator

## Overview

Build a single-page, fully client-side web application in TypeScript/React (or plain TypeScript + HTML) that parses SQL queries and emits equivalent pandas code with per-clause explanations. All logic runs in the browser — no backend required. The build uses Vite + Vitest + fast-check + Playwright.

## Tasks

- [x] 1. Initialise project structure and tooling
  - Scaffold a Vite + TypeScript project (`npm create vite@latest`)
  - Install and pin exact versions: `node-sql-parser`, `prismjs`, `fast-check`, `vitest`, `@playwright/test`, and optionally `tailwindcss`
  - Configure `tsconfig.json`, `vite.config.ts`, and `vitest.config.ts`
  - Create the `src/` directory layout: `components/`, `core/`, `data/`, `styles/`
  - _Requirements: 7.1 (static deployable site implied by client-side architecture)_

- [x] 2. Define core data models and types
  - [x] 2.1 Create `src/core/types.ts` with `ClauseType`, `ClauseNode`, `SqlAST`, `ClauseOutput`, `TranslationResult`, `TranslationError`, and `ExampleQuery` type definitions
    - Implement all TypeScript interfaces and enums from the design document
    - _Requirements: 2.1, 2.2, 4.1, 4.2, 4.3_

- [ ] 3. Implement the Input Validator
  - [x] 3.1 Create `src/core/inputValidator.ts` implementing `validateInput(text: string): ValidationResult`
    - Return error for empty / whitespace-only input with message "Please enter a SQL query before translating."
    - Return error for input exceeding 10,000 characters with message including actual count
    - Return success for valid input
    - _Requirements: 1.3, 1.6_

  - [ ] 3.2 Write property test for Input Validator
    - **Property 5: Character limit** — Input strings longer than 10,000 characters never reach the parser (validator always blocks them first)
    - **Validates: Requirements 1.6**

  - [-] 3.3 Write unit tests for Input Validator
    - Empty string → error, whitespace-only → error, exactly 10,000 chars → valid, 10,001 chars → error
    - _Requirements: 1.3, 1.6_

- [ ] 4. Implement the SQL Parser
  - [x] 4.1 Create `src/core/sqlParser.ts` wrapping `node-sql-parser` to produce a `SqlAST`
    - Tokenise raw SQL and populate `clauses: ClauseNode[]` for all supported `ClauseType` values
    - Populate `unsupported: string[]` for unrecognised constructs (CTEs, window functions, UNION, etc.)
    - On parse failure, throw a structured `ParseError` with `lineNumber` and `token`
    - _Requirements: 2.3–2.13, 4.1_

  - [-] 4.2 Write unit tests for SQL Parser
    - Valid SELECT → correct AST; missing FROM → parse error; nested subquery → child ClauseNode
    - Each supported clause type produces the correct `ClauseNode.type`
    - _Requirements: 2.3–2.13, 4.1_

- [ ] 5. Implement the Clause Translator
  - [~] 5.1 Create `src/core/clauseTranslator.ts` with `translateClause(node: ClauseNode): string`
    - Implement SELECT → bracket notation / `.loc[]`, with aggregate mapping (COUNT→`.count()`, SUM→`.sum()`, AVG→`.mean()`, MIN→`.min()`, MAX→`.max()`)
    - Implement WHERE → boolean mask with operator mapping (`=`→`==`, AND→`&`, OR→`|`, IS NULL→`.isna()`, IN→`.isin()`, LIKE→`.str.contains()`)
    - Implement GROUP BY → `groupby()` + `.agg()` + `.reset_index()`
    - Implement HAVING → post-aggregation boolean mask
    - Implement ORDER BY → `sort_values()` with correct `ascending` parameter(s)
    - Implement LIMIT → `head(n)`
    - Implement JOIN (INNER/LEFT/RIGHT/FULL OUTER) → `pd.merge()` with correct `how`
    - Implement DISTINCT → `drop_duplicates()`
    - Implement SUBQUERY_FROM → intermediate DataFrame assignment
    - Implement SUBQUERY_WHERE_HAVING → nested inline expression
    - Replace unsupported clauses with `# [UNSUPPORTED: <ClauseName>]` comment
    - _Requirements: 2.3–2.14_

  - [~] 5.2 Write property test for Clause Translator — round-trip completeness
    - **Property 1: Round-trip completeness** — For any SQL query composed of supported clauses, the translator always produces non-empty `fullPandasCode`
    - **Validates: Requirements 2.1, 2.2**

  - [~] 5.3 Write property test for Clause Translator — error isolation
    - **Property 2: Error isolation** — If a query contains any unsupported clause, the result always includes a `TranslationError` of type `UNSUPPORTED_CLAUSE`
    - **Validates: Requirements 4.2**

  - [~] 5.4 Write property test for Clause Translator — partial output consistency
    - **Property 4: Partial output consistency** — When `hasPartialOutput` is `true`, every unsupported clause appears exactly once as a `# [UNSUPPORTED: ...]` comment in `fullPandasCode`
    - **Validates: Requirements 4.3**

  - [~] 5.5 Write unit tests for Clause Translator
    - One test per clause type: SELECT with aliases / * / aggregates; WHERE with AND/OR/IN/IS NULL/LIKE; GROUP BY single + multiple columns; all four JOIN types; ORDER BY mixed ASC/DESC; LIMIT; HAVING; DISTINCT; FROM subquery; WHERE subquery
    - _Requirements: 2.3–2.14_

- [ ] 6. Implement the Explanation Generator
  - [x] 6.1 Create `src/core/explanationGenerator.ts` with `getExplanation(clauseType: ClauseType): string`
    - Return the explanation string for each `ClauseType` as defined in the design's `EXPLANATIONS` map
    - _Requirements: 3.1–3.10_

  - [-] 6.2 Write property test for Explanation Generator — explanation coverage
    - **Property 3: Explanation coverage** — For every `ClauseOutput` in `clauses`, a non-empty explanation string is always present
    - **Validates: Requirements 3.1, 3.3**

  - [ ] 6.3 Write unit tests for Explanation Generator
    - Each `ClauseType` value → returns a non-empty string referencing the correct pandas construct
    - _Requirements: 3.4–3.10_

- [ ] 7. Implement the Error Handler
  - [x] 7.1 Create `src/core/errorHandler.ts` implementing `buildErrorList(parseError?, unsupportedClauses?): TranslationError[]`
    - Produce `PARSE_ERROR` entries with `lineNumber` and `token`
    - Produce `UNSUPPORTED_CLAUSE` entries with `clauseName`
    - Produce `VALIDATION_ERROR` entries for input validation failures
    - _Requirements: 4.1–4.4_

  - [~] 7.2 Write unit tests for Error Handler
    - Parse error → includes line number and token; unsupported clause → lists clause name
    - _Requirements: 4.1, 4.2_

- [ ] 8. Wire core pipeline into a translation orchestrator
  - [~] 8.1 Create `src/core/translator.ts` with `translate(sqlText: string): TranslationResult`
    - Call `validateInput` → `sqlParser` → `clauseTranslator` → `explanationGenerator` → `errorHandler` in sequence
    - Assemble `fullPandasCode` as ordered concatenation of `ClauseOutput.pandasCode` segments
    - Order clauses per pandas execution order: DataFrame assignment → filter → groupby → having → select → sort → head
    - Set `hasPartialOutput` when some clauses translated and some did not
    - _Requirements: 2.1, 2.2, 4.1–4.4_

- [~] 9. Checkpoint — Ensure all core logic tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Build the SQL Input Panel component
  - [~] 10.1 Create `src/components/SqlInputPanel.tsx` (or `.ts` + template)
    - Render `<textarea rows="10" maxlength="10000" aria-label="SQL query input">` with `aria-describedby` pointing to char counter and validation message elements
    - Render real-time character counter below the textarea; debounce update to 100 ms on large pastes
    - Render "Translate" `<button type="submit">` with accessible label
    - Display `validationMessage` in an ARIA live-region when non-null; clear on any non-whitespace keystroke
    - Disable textarea and button while translation is in progress
    - Return focus to textarea after submission
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 7.4, 7.5_

  - [~] 10.2 Write unit tests for SqlInputPanel
    - Empty submit → validation message shown; non-empty submit → `onSubmit` called; input > 10,000 chars → error shown
    - _Requirements: 1.3, 1.6_

- [ ] 11. Build the Example Query Panel component
  - [~] 11.1 Create `src/components/ExampleQueryPanel.tsx` with the seven `EXAMPLE_QUERIES` from `src/data/examples.ts`
    - Render each example as a labelled button (label ≤ 60 chars) accessible by keyboard (Enter/Space)
    - On selection: replace SQL input content, immediately trigger translation, display errors in error section if translation fails (never blank the input)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [~] 11.2 Write property test for Example Query Panel — example query validity
    - **Property 6: Example query validity** — All seven pre-built example queries parse without errors and produce non-empty `fullPandasCode`
    - **Validates: Requirements 6.1, 6.3**

  - [~] 11.3 Write unit tests for ExampleQueryPanel
    - Selecting an example replaces input and triggers translation; each label ≤ 60 characters
    - _Requirements: 6.2, 6.3, 6.4_

- [ ] 12. Build the Copy Button component
  - [~] 12.1 Create `src/components/CopyButton.tsx`
    - Use `navigator.clipboard.writeText()` to copy `targetText`
    - Disable button (`aria-disabled="true"`) when `targetText` is empty
    - Show "Copied!" confirmation for 2–3 seconds using `aria-live="polite"`, then auto-dismiss
    - On failure: show inline error and set `style="user-select: all"` on the code block for manual copy
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [~] 12.2 Write unit tests for CopyButton
    - Disabled when no output; success → confirmation shown then dismissed; failure → fallback message shown
    - _Requirements: 5.2, 5.4, 5.5_

- [ ] 13. Build the Error Section component
  - [~] 13.1 Create `src/components/ErrorSection.tsx`
    - Render a visually distinct container (amber/red background, warning icon) with `role="alert"` and `aria-live="assertive"`
    - List each `TranslationError` with type label and message; include `clauseName` for `UNSUPPORTED_CLAUSE` and `lineNumber`/`token` for `PARSE_ERROR`
    - Render only when `errors.length > 0`
    - _Requirements: 4.2, 4.4_

  - [~] 13.2 Write unit tests for ErrorSection
    - Renders nothing when errors array is empty; renders correct labels and messages for each error type
    - _Requirements: 4.4_

- [ ] 14. Build the Output Panel component
  - [~] 14.1 Create `src/components/OutputPanel.tsx`
    - Render `fullPandasCode` in a `<code>` block with Python syntax highlighting via Prism.js (applied via `requestAnimationFrame`)
    - Render each `ClauseOutput.explanation` adjacent to its code fragment
    - Render `CopyButton` when `fullPandasCode` is non-empty
    - Render `ErrorSection` when `errors` is non-empty; keep it structurally separate from the code block
    - Render nothing when `result` is null
    - _Requirements: 2.2, 3.1, 3.2, 3.3, 4.4, 5.1_

  - [~] 14.2 Write unit tests for OutputPanel
    - Null result → blank panel; successful result → code block + explanations shown; errors present → error section shown separately
    - _Requirements: 2.2, 3.1, 4.4_

- [~] 15. Checkpoint — Ensure all component tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Implement responsive layout and global styles
  - [~] 16.1 Create `src/styles/layout.css` (or Tailwind config) implementing the two-column / single-column responsive grid
    - ≥ 1024 px: two-column layout, both panels visible without vertical scroll
    - 768 px – 1023 px: single-column stacked layout
    - Example Query Panel above SQL Input Panel at all widths
    - No horizontal scrollbar at any viewport from 768 px to 2560 px
    - _Requirements: 7.1, 7.2_

  - [~] 16.2 Apply accessibility and focus styles
    - Minimum 4.5:1 contrast ratio for body text; 3:1 for UI component borders and large text
    - Visible focus indicator: 2 px solid outline with ≥ 3:1 contrast against adjacent background
    - Focus order: Examples → SQL Input → Translate Button → Copy Button → Output → Error Section
    - Font sizes: 16 px body, 14 px code blocks
    - _Requirements: 7.3, 7.4, 7.5_

- [ ] 17. Wire everything together in the root App component
  - [~] 17.1 Create `src/App.tsx` (or `src/main.ts`) composing `ExampleQueryPanel`, `SqlInputPanel`, and `OutputPanel`
    - Manage shared state: `sqlText`, `translationResult`, `isTranslating`
    - Connect `ExampleQueryPanel.onSelect` → set `sqlText` + call `translate()`
    - Connect `SqlInputPanel.onSubmit` → call `translate()` + update `translationResult`
    - Pass `translationResult` to `OutputPanel`
    - Ensure Content Security Policy meta tag restricts `script-src` to `'self'` plus the syntax-highlighting CDN
    - _Requirements: 1.5, 6.2, 6.3_

- [ ] 18. Write end-to-end integration tests
  - [~] 18.1 Write Playwright integration tests for the main user flows
    - Page load → input empty, output blank, copy button disabled
    - Click "Translate" with empty input → inline validation message, output stays blank
    - Select "Filter rows with WHERE" example → input populated, pandas code and WHERE explanation shown
    - Type valid GROUP BY query → output shows `groupby()` code and GROUP BY explanation
    - Click copy button → "Copied!" confirmation appears for 2–3 seconds then disappears
    - Type unsupported clause (e.g., `WITH cte AS (...)`) → error section shown, no crash
    - Responsive check at 768 px viewport width → no horizontal scrollbar
    - _Requirements: 1.3, 2.2, 3.1, 4.2, 5.4, 6.3, 7.1_

- [~] 19. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- All dependencies must be pinned to exact versions in `package.json`
- Property tests use `fast-check` to generate arbitrary inputs against the six properties defined in the design's Testing Strategy
- Unit tests use `Vitest`; end-to-end tests use `Playwright`
- Syntax highlighting is applied after translation via `requestAnimationFrame` to avoid blocking the UI thread
- The app is entirely client-side — no server, no eval(), no database connections

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["3.1", "4.1", "6.1", "7.1"] },
    { "id": 2, "tasks": ["3.2", "3.3", "4.2", "6.2", "6.3", "7.2"] },
    { "id": 3, "tasks": ["5.1"] },
    { "id": 4, "tasks": ["5.2", "5.3", "5.4", "5.5"] },
    { "id": 5, "tasks": ["8.1"] },
    { "id": 6, "tasks": ["10.1", "11.1", "12.1", "13.1"] },
    { "id": 7, "tasks": ["10.2", "11.2", "11.3", "12.2", "13.2", "14.1"] },
    { "id": 8, "tasks": ["14.2", "16.1", "16.2"] },
    { "id": 9, "tasks": ["17.1"] },
    { "id": 10, "tasks": ["18.1"] }
  ]
}
```
