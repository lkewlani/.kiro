# Requirements Document

## Introduction

This feature is a SQL-to-Pandas Python learning app targeted at data analysts who already know SQL and want to learn Python. Users paste a SQL query into a text input, and the app instantly returns the equivalent pandas code alongside plain-language explanations of how each SQL concept (SELECT, WHERE, GROUP BY, JOIN, ORDER BY, LIMIT, etc.) maps to a pandas operation. The goal is to accelerate Python learning by anchoring new concepts to familiar SQL knowledge.

## Glossary

- **App**: The SQL-to-Pandas translator web application.
- **User**: A data analyst learning Python who has existing SQL knowledge.
- **SQL_Input**: The raw SQL query text entered by the User.
- **Translator**: The component that parses SQL_Input and produces Pandas_Output.
- **Pandas_Output**: The generated Python/pandas code equivalent to the SQL_Input.
- **Concept_Explanation**: A short, human-readable description of how a specific SQL clause maps to a pandas operation.
- **Clause**: A distinct SQL keyword section within a query (e.g., SELECT, WHERE, GROUP BY, JOIN, ORDER BY, LIMIT).
- **Supported_Clause**: A Clause that the Translator is capable of converting to pandas.
- **Unsupported_Clause**: A Clause present in the SQL_Input that the Translator cannot convert.
- **Code_Block**: A formatted, syntax-highlighted display area for showing code.

---

## Requirements

### Requirement 1: SQL Input Entry

**User Story:** As a User, I want to paste or type a SQL query into a text box, so that I can submit it for translation to pandas code.

#### Acceptance Criteria

1. THE App SHALL provide a multi-line text input area with a minimum of 10 visible rows for entering SQL_Input.
2. THE App SHALL accept SQL_Input of up to 10,000 characters in length.
3. WHEN the SQL_Input area is empty or contains only whitespace and the User attempts to submit, THE App SHALL display an inline validation message indicating that a query is required.
4. THE App SHALL provide a submit control labelled "Translate" (or equivalent explicit action label) that triggers translation when activated.
5. WHEN the User activates the submit control, THE App SHALL pass the SQL_Input to the Translator.
6. IF the SQL_Input length exceeds 10,000 characters, THEN THE App SHALL display an inline error message stating that the maximum input length has been exceeded and SHALL NOT pass the SQL_Input to the Translator.

---

### Requirement 2: SQL-to-Pandas Translation

**User Story:** As a User, I want the app to convert my SQL query into equivalent pandas code, so that I can see exactly how to write the same logic in Python.

#### Acceptance Criteria

1. WHEN valid SQL_Input is submitted, THE Translator SHALL produce Pandas_Output such that executing the Pandas_Output against the same input data as the SQL_Input returns the same columns, same rows, and same values.
2. THE App SHALL display the Pandas_Output in a Code_Block with Python syntax highlighting.
3. WHEN the SQL_Input contains a SELECT clause, THE Translator SHALL convert column selection to a pandas DataFrame column-selection expression.
4. WHEN the SQL_Input contains a WHERE clause, THE Translator SHALL convert the filter condition to a pandas boolean-indexing expression.
5. WHEN the SQL_Input contains a GROUP BY clause, THE Translator SHALL convert the grouping to a pandas `groupby()` call.
6. WHEN the SQL_Input contains an aggregate function (COUNT, SUM, AVG, MIN, MAX) in the SELECT clause, THE Translator SHALL convert the aggregate to the corresponding pandas aggregation method.
7. WHEN the SQL_Input contains an ORDER BY clause, THE Translator SHALL convert the ordering to a pandas `sort_values()` call with the correct `ascending` parameter.
8. WHEN the SQL_Input contains a LIMIT clause, THE Translator SHALL convert the row limit to a pandas `head()` call.
9. WHEN the SQL_Input contains a JOIN clause (INNER, LEFT, RIGHT, or FULL OUTER), THE Translator SHALL convert the join to a pandas `merge()` call with the correct `how` parameter.
10. WHEN the SQL_Input contains a HAVING clause, THE Translator SHALL convert the post-aggregation filter to a pandas boolean-indexing expression applied after `groupby()`.
11. WHEN the SQL_Input contains a DISTINCT keyword, THE Translator SHALL convert it to a pandas `drop_duplicates()` call.
12. WHEN the SQL_Input contains a subquery in the FROM clause, THE Translator SHALL convert it to a named intermediate DataFrame assigned before the outer query expression.
13. WHEN the SQL_Input contains a subquery in a WHERE or HAVING clause, THE Translator SHALL convert it to a nested pandas expression inline within the outer boolean-indexing expression.
14. IF the SQL_Input contains a SQL construct that the Translator does not support, THEN THE Translator SHALL identify the unsupported construct by name and SHALL NOT produce partial Pandas_Output for that construct.

---

### Requirement 3: Per-Clause Concept Explanations

**User Story:** As a User, I want to see an explanation for each SQL clause in my query, so that I understand how SQL concepts map to pandas operations.

#### Acceptance Criteria

1. WHEN Pandas_Output is displayed, THE App SHALL display a Concept_Explanation for each Supported_Clause present in the SQL_Input.
2. THE App SHALL display each Concept_Explanation within the same visual section as the Pandas_Output for its corresponding clause, not separated from it by another clause's output.
3. THE App SHALL include in each Concept_Explanation a reference to the specific pandas construct used for that clause (e.g., `groupby()` for GROUP BY, `merge()` for JOIN).
4. WHEN the SQL_Input contains a SELECT clause, THE App SHALL display a Concept_Explanation describing how pandas selects columns using bracket notation or `.loc[]`.
5. WHEN the SQL_Input contains a WHERE clause, THE App SHALL display a Concept_Explanation describing how pandas uses boolean masks for row filtering.
6. WHEN the SQL_Input contains a GROUP BY clause, THE App SHALL display a Concept_Explanation describing how `groupby()` works in pandas.
7. WHEN the SQL_Input contains a JOIN clause, THE App SHALL display a Concept_Explanation describing the `merge()` function and its `how` parameter.
8. WHEN the SQL_Input contains an ORDER BY clause, THE App SHALL display a Concept_Explanation describing `sort_values()` and the `ascending` parameter.
9. WHEN the SQL_Input contains a LIMIT clause, THE App SHALL display a Concept_Explanation describing the `head()` method.
10. WHEN the SQL_Input contains a HAVING clause, THE App SHALL display a Concept_Explanation describing post-aggregation filtering in pandas.
11. IF the SQL_Input contains no Supported_Clause, THEN THE App SHALL NOT display any Concept_Explanation section.

---

### Requirement 4: Error Handling for Invalid or Unsupported SQL

**User Story:** As a User, I want to receive clear feedback when my SQL query cannot be translated, so that I understand what needs to be corrected or simplified.

#### Acceptance Criteria

1. IF the SQL_Input cannot be parsed as valid SQL, THEN THE Translator SHALL return an error message that includes the line number or the offending clause token and the nature of the syntax problem, and SHALL NOT proceed to unsupported-clause detection.
2. IF the SQL_Input contains one or more Unsupported_Clauses, THEN THE App SHALL display a message listing all Unsupported_Clauses found and stating that each is not yet supported.
3. IF a partial translation is possible (some Supported_Clauses and some Unsupported_Clauses), THEN THE App SHALL display the partial Pandas_Output and replace each untranslatable section with an inline annotation that identifies the Unsupported_Clause by name.
4. IF the Translator returns any error or unsupported-clause message, THEN THE App SHALL display those messages in a dedicated labeled section that is structurally separate from the Pandas_Output Code_Block.

---

### Requirement 5: Copy-to-Clipboard for Generated Code

**User Story:** As a User, I want to copy the generated pandas code with one click, so that I can quickly paste it into my Python environment.

#### Acceptance Criteria

1. THE App SHALL provide a copy-to-clipboard control within or adjacent to the Code_Block displaying Pandas_Output.
2. THE App SHALL disable the copy-to-clipboard control when no Pandas_Output is present.
3. WHEN the User activates the copy-to-clipboard control, THE App SHALL copy the full Pandas_Output text to the system clipboard.
4. WHEN the copy operation succeeds, THE App SHALL display a confirmation message for 2–3 seconds and then remove it automatically.
5. IF the copy operation fails due to a browser permission error, THEN THE App SHALL display an inline error message and show the Pandas_Output text in a selectable format so the User can copy it manually.

---

### Requirement 6: Example Queries for Getting Started

**User Story:** As a User, I want to see pre-built example SQL queries, so that I can quickly explore translations without writing a query from scratch.

#### Acceptance Criteria

1. THE App SHALL provide at least five pre-built example SQL queries covering: basic SELECT, WHERE filtering, GROUP BY with aggregation, JOIN, and ORDER BY with LIMIT.
2. WHEN the User selects an example query, THE App SHALL replace any existing SQL_Input content with the selected example's SQL text.
3. WHEN an example query is loaded into the SQL_Input area, THE App SHALL immediately trigger translation and display the corresponding Pandas_Output and Concept_Explanations.
4. THE App SHALL display each example query with a descriptive label of no more than 60 characters (e.g., "Filter rows with WHERE", "Group and aggregate with GROUP BY").
5. IF the auto-triggered translation for a loaded example query fails, THEN THE App SHALL display the error in the dedicated error section and SHALL NOT leave the SQL_Input area blank.

---

### Requirement 7: Responsive and Accessible UI

**User Story:** As a User, I want the app to be usable on both desktop and tablet screens, so that I can access it from my preferred device.

#### Acceptance Criteria

1. THE App SHALL render on viewport widths from 768px to 2560px without producing a horizontal scrollbar or clipping any content outside the visible area.
2. THE App SHALL display both the SQL_Input area and the Pandas_Output area fully visible within the viewport on viewports of 1024px or wider, requiring no vertical scroll to see either panel.
3. THE App SHALL provide sufficient colour contrast for all text and code elements, meeting WCAG 2.1 AA contrast ratio requirements (minimum 4.5:1 for normal text, 3:1 for large text and UI components).
4. THE App SHALL associate all interactive controls with accessible names, roles, and states conforming to WCAG 2.1 AA Success Criterion 4.1.2.
5. WHEN the User navigates the App using keyboard controls only, THE App SHALL allow the User to reach and activate all interactive controls, and each focused control SHALL display a visible focus indicator with a contrast ratio of at least 3:1 against its adjacent background.
