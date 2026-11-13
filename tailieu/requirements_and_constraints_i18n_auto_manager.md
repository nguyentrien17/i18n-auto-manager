# Requirements and Constraints — i18n Auto Manager

**Document type:** Product requirements, engineering constraints, and acceptance contract  
**Version:** 1.1 (Harmonized Specification)  
**Product:** i18n Auto Manager — a local-first Visual Studio Code extension  
**Status:** Normative specification for implementation agents  
**Language:** English  
**Purpose:** Prevent an AI coding agent or developer from making unsafe assumptions, implementing unsupported behavior, corrupting source formatting, or claiming that localization is complete when it is not.

---

## 1. How to Interpret This Document

The words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative (RFC 2119):

- **MUST / MUST NOT:** mandatory requirement. A release or implementation milestone is not complete if violated.
- **SHOULD / SHOULD NOT:** strong recommendation. Deviations require a documented architectural reason.
- **MAY:** optional capability.

When requirements conflict, apply this strict priority order:

1. Preserve user data and source-code correctness.
2. Never make an unreviewed destructive change.
3. Preserve runtime behavior, application logic, and surrounding source formatting.
4. Produce valid, maintainable localization output.
5. Maximize automation and convenience.
6. Optimize speed and visual polish.

A safer partial result is always preferable to an unsafe complete-looking result.

---

## 2. Product Definition

**i18n Auto Manager** is a VS Code desktop extension that helps developers add, maintain, and validate internationalization in existing software projects.

The extension MUST operate locally by default. Its core workflow MUST NOT depend on an AI model, remote API, cloud account, or network connection.

The extension is an **assisted, deterministic code transformation tool**, not a universal automatic translator and not a guarantee that every project can be internationalized without developer review.

### 2.1 Primary user outcome

A developer can:

1. Open an existing project in VS Code.
2. Ask the extension to inspect the workspace.
3. Review detected frameworks, source files, existing localization libraries, and candidate user-facing strings.
4. Configure a source locale, target locales, resource format, and framework-specific settings.
5. Preview proposed code changes (via native VS Code diffs) and translation resource updates.
6. Apply only explicitly approved changes.
7. Run validation and receive an actionable report.
8. Repeat the process incrementally after the project changes.

### 2.2 Core product principles

- **Local-first and offline-capable:** Zero external network calls in core workflows.
- **Explicit user approval:** Mandatory human-in-the-loop before any file write.
- **Framework-aware, AST-based range transformations:** Parse with ASTs, edit with targeted character ranges to preserve formatting.
- **Conservative detection:** Uncertainty must be visible (`unknown` or `low` confidence).
- **Idempotent operations:** Running the same operation twice must not create duplicate keys, duplicate imports, or repeatedly rewrite code.
- **Reversible changes:** Use native `vscode.WorkspaceEdit` and virtual document diffing for seamless preview and Undo support.
- **Honest translation status:** Never claim that copied source text or placeholder text is a completed translation.
- **Incremental scanning:** Avoid rescanning the entire workspace unnecessarily.

---

## 3. Scope and Non-Goals

### 3.1 Required initial scope (Version 1.0)

Version 1.0 MUST focus on a strictly bounded, well-tested JavaScript/TypeScript ecosystem scope:

- VS Code desktop extension (bundled with `esbuild`).
- Strict TypeScript implementation.
- Local workspace inspection (single-root and multi-root).
- HTML static text and attribute analysis.
- JavaScript and TypeScript (`.js`, `.ts`) string candidate analysis.
- Vue 3 Single-File Components (`.vue`), including `<template>` and `<script>` / `<script setup>` regions (targeting `vue-i18n`).
- React JSX/TSX (`.jsx`, `.tsx`) analysis (targeting `i18next` / `react-i18next` or `react-intl`).
- JSON localization resource files (nested and flat structures).
- Existing i18n configuration detection where it can be identified reliably.
- Candidate review, deterministic key generation, resource merging, range-based source edits, and validation.

### 3.2 Deferred post-1.0 scope (Version 1.1+)

- **Laravel / PHP / Blade support:** MUST be treated as a separate post-1.0 milestone (v1.1) requiring a dedicated PHP/Blade AST parser and test suite. It MUST NOT be claimed as supported in v1.0 merely because the extension can find `.php` files.
- **YAML / PO / XLIFF resource formats:** Deferred to v1.1+.

### 3.3 Explicit non-goals for Version 1.0

The initial release MUST NOT attempt to:

- Understand every programming language or framework.
- Translate arbitrary text with guaranteed semantic accuracy.
- Automatically infer the correct business meaning of ambiguous strings.
- Rewrite database content, API responses, CMS content, or external service data.
- Modify application business logic or control flow to make a project fit the extension.
- Automatically install dependencies (`npm`, `pnpm`, `yarn`) or execute project scripts.
- Change backend APIs, database schemas, or deployment configuration.
- Guarantee that translated text fits every screen layout or CSS design.
- Automatically localize images, audio, video, PDFs, or other media.
- Make changes outside the opened workspace folders.
- Upload source code or translation content to external services.

Unsupported cases MUST be reported as `unsupported` or `needs-manual-handling`. They MUST NOT be silently skipped while reporting full success.

---

## 4. Unified Domain Taxonomy

To prevent mismatches between the core engine, adapters, and UI, all modules MUST use the exact canonical types and enumerations defined below:

- **Confidence Level (`ConfidenceLevel`):**
  - `'high'`: Clearly user-visible static text in a supported UI syntax node.
  - `'medium'`: Likely user-visible text, or text with simple interpolation, requiring developer verification.
  - `'low'`: Ambiguous string literal where UI vs. technical usage is unclear.
  - `'unknown'`: Syntax or context cannot be classified deterministically.
- **Candidate Category (`CandidateCategory`):**
  - `'static-ui-text'`: Pure static user-facing text node or UI attribute.
  - `'dynamic-ui-text'`: User-facing text containing interpolated expressions.
  - `'technical-string'`: Identifiers, paths, URLs, CSS classes, log messages, internal constants (excluded from transformation).
  - `'existing-i18n-call'`: Already localized string reference.
  - `'uncertain'`: Context cannot be confirmed safely.
- **Candidate Review State (`CandidateStatus`):**
  - `'pending'`: Newly discovered, awaiting user decision.
  - `'approved'`: Explicitly accepted by the user for transformation.
  - `'rejected'`: Explicitly declined by the user for the current run.
  - `'ignored'`: Permanently excluded via rule or user configuration.
  - `'needs-manual-handling'`: Complex expression or unsupported syntax that cannot be auto-transformed safely.
  - `'stale'`: Source file was modified after the scan; requires rescan before applying.
  - `'applied'`: Transformation has been written to the workspace.
- **Validation Severity (`ValidationSeverity`):**
  - `'error'` | `'warning'` | `'info'` | `'not-checked'`
- **Operation Report Status (`OperationStatus`):**
  - `'completed-successfully'` | `'completed-with-warnings'` | `'partially-completed'` | `'cancelled'` | `'failed'`

---

## 5. Functional Requirements

### FR-001 — Workspace discovery

The extension MUST:

- Operate only on the workspace folders explicitly opened by the user.
- Support single-root and multi-root workspaces.
- Respect `.gitignore` where applicable and provide extension-specific exclude patterns.
- Exclude generated, dependency, build, cache, and vendor directories by default (`node_modules`, `vendor`, `dist`, `build`, `out`, `.next`, `.nuxt`, `.git`, `coverage`).
- Allow users to inspect and change include/exclude glob patterns.
- Avoid reading binary files or files exceeding `maxFileSizeKb` (default `1024` KB) as source text.
- Avoid following symbolic links outside the workspace unless explicitly enabled.
- Display the exact count of discovered, included, excluded, and unsupported files.

### FR-002 — Project and framework detection

The extension SHOULD identify project technologies using multiple evidence sources:

- `package.json` dependencies and devDependencies.
- Framework configuration files (`vite.config.*`, `vue.config.*`, `next.config.*`, `tsconfig.json`).
- File extensions and directory patterns.
- Existing localization initialization files (`i18n.ts`, `i18n.js`, locale folders).

Detection MUST include a `ConfidenceLevel` and an array of evidence file paths.

If multiple frameworks are present (e.g., a monorepo with Vue and React packages), the extension MUST report all detected frameworks per sub-project rather than selecting one arbitrarily.

If evidence is insufficient, the extension MUST display **Unknown / Needs confirmation** and allow the user to choose the adapter manually.

The extension MUST NOT modify project files during framework detection.

### FR-003 — Existing i18n configuration inspection

Before proposing a new localization setup, the extension MUST inspect for existing configuration and resource files:

- Existing localization libraries (`vue-i18n`, `i18next`, `react-i18next`, `react-intl`).
- Existing locale directories and resource files.
- Default/source or fallback locale.
- Existing key naming conventions (`nested-dot`, `flat-dot`, `camelCase`, `snake_case`).
- Existing interpolation conventions (`{var}`, `{{var}}`).
- Existing pluralization conventions.

If an existing i18n system is detected, the extension MUST integrate with it instead of creating a competing system. If two or more plausible configurations conflict, the extension MUST prompt the user to select one.

### FR-004 — Candidate string extraction

The extension MUST extract candidate user-facing strings only from supported syntax regions:

- HTML text nodes and translatable attributes (`title`, `placeholder`, `alt`, `aria-label`, `label`).
- Vue 3 `<template>` text nodes, mustaches `{{ }}`, translatable attributes, and `<script>` string literals in recognized UI contexts.
- React JSX/TSX `JSXText` nodes, string literals inside `JSXExpressionContainer`, and translatable props.
- JavaScript/TypeScript string literals in recognized UI contexts.
- Existing localization calls (`$t()`, `t()`, `<i18n-t>`, `<Trans>`, `formatMessage()`).

The extractor MUST preserve exact 0-based line/character ranges, byte/character offsets, a SHA-256 content hash of the source file at scan time, syntax context, and file URI.

### FR-005 — Candidate classification

Every candidate MUST have:

- A deterministic candidate ID for the current scan result.
- File URI, `SourceRange` (lines, columns, and character offsets), and `fileContentHash`.
- Original text and normalized preview text.
- Syntax context and `CandidateCategory`.
- `ConfidenceLevel` (`'high'` | `'medium'` | `'low'` | `'unknown'`).
- Structured `reasonCodes` explaining why it was classified.
- `CandidateStatus` (defaulting to `'pending'` or `'needs-manual-handling'`).

Default approval rules:

- `'high'` confidence: MAY be preselected in the review UI, but MUST still be visible and require the user's explicit "Apply" confirmation.
- `'medium'` confidence: MUST NOT be preselected; requires explicit per-item or filtered-group approval.
- `'low'` or `'unknown'` confidence: MUST NOT be auto-applied; transformation is disabled unless manually overridden and supported by the adapter.

### FR-006 — Dynamic content and interpolation

The extension MUST preserve dynamic expressions and their evaluation order.

For example, `Hello, {user.name}` in JSX or `Hello, {{ user.name }}` in Vue must become a localized message with an explicit interpolation parameter (e.g., `"Hello, {userName}"` or `"Hello, {{userName}}"` matching the library convention) and pass `{ userName: user.name }` at the call site.

The extension MUST:

- Identify interpolated identifiers and simple member expressions safely.
- Generate deterministic, collision-free variable names.
- Mark complex expressions (e.g., nested ternaries, function calls with side effects) as `'needs-manual-handling'`.
- Never evaluate expressions during scanning.

### FR-007 — Key generation

Generated keys MUST:

- Be deterministic for the same approved candidate and configuration.
- Follow the configured naming convention (e.g., `feature.component.textSlug`).
- Avoid collisions with existing keys who have different source values.
- Reuse existing keys if configured by the user when the source text and context match.
- Preserve all existing keys in resource files even if they were not generated by this extension.
- Never depend on volatile line or column numbers.

### FR-008 — Translation resource generation

For Version 1.0, JSON resource files (both nested objects and flat dot-notation keys) MUST be supported, preserving indentation (2 spaces, 4 spaces, or tabs) and trailing newlines.

The extension MUST:

- Preserve valid JSON structure and UTF-8 encoding.
- Preserve existing keys and translated values without reordering unrelated keys unnecessarily.
- Add only approved, missing entries.
- Keep source-locale values separate from target-locale values.
- Preserve interpolation variable names across all locales.

If no translation is available for a target locale, the extension MUST support configurable placeholder strategies:
1. Empty string `""` (marked as `missing`).
2. Omitted key in target file (marked as `missing`).
3. Source text copy with an explicit prefix (e.g., `"[UNTRANSLATED] Cancel"`) or companion metadata status (`placeholder`), ONLY when explicitly chosen by the user.

The extension MUST NOT label copied source text as `'translated'`.

### FR-009 — Translation input

Version 1.0 MUST support manual translation editing and importing/exporting local JSON/CSV translation resources.

Version 1.0 MUST NOT require an AI service. Optional external translation providers MAY be added in post-1.0 versions, disabled by default, requiring explicit opt-in consent and data disclosure.

### FR-010 — Source-code transformation and formatting preservation

Source edits MUST use syntax-aware AST parsing combined with **range-based surgical string replacement**.

- Regular expressions MUST NOT be the primary mechanism for parsing or rewriting JS, TS, Vue, JSX, or TSX code.
- **Formatting Preservation Rule:** The transformation engine MUST NOT serialize/pretty-print the entire AST back to source text (which destroys user formatting, comments, quotes, and blank lines). Instead, the parser MUST identify exact start/end character offsets (`SourceRange`), and the transformation engine MUST generate targeted `vscode.TextEdit` replacements for only those specific ranges (plus a targeted import/hook insertion edit if needed).

Every proposed edit MUST include:

- Original file URI and `fileContentHash` (SHA-256) captured at scan time.
- Exact `SourceRange` (start/end line, character, and offset).
- Original substring at that range.
- Replacement text.
- Associated translation key and adapter ID.

Before applying an edit, the extension MUST verify that the target file's current SHA-256 hash and range substring still match the scanned snapshot. If the file changed, the edit MUST be marked `'stale'` and blocked until rescanned.

The extension MUST:

- Avoid adding duplicate `import` declarations or duplicate hook calls (e.g., `const { t } = useI18n()`).
- Apply related source and resource changes via `vscode.WorkspaceEdit`.

### FR-011 — Review and application (Native Diff Integration)

The extension MUST provide a review step before applying any changes:

- Use VS Code's built-in diff viewer (`vscode.commands.executeCommand('vscode.diff', originalUri, previewUri, title)`) backed by an in-memory `vscode.TextDocumentContentProvider` to show exact side-by-side diffs of proposed source and JSON resource modifications.
- Allow approving or rejecting individual candidates or filtered batches.
- Revalidate all affected files immediately prior to writing. If any critical validation fails (such as a stale file hash or syntax error in generated output), abort the write transaction and explain why.

### FR-012 — Validation

The extension MUST validate and report:

- Missing keys referenced in source code but absent in source/target locale files (`error`).
- Missing target-locale translations compared to the source locale (`warning`).
- Unused resource keys where static analysis can determine usage reliably (`info` / `warning`).
- Duplicate keys or conflicting key paths (`error`).
- Invalid JSON syntax in locale files (`error`).
- Interpolation variable mismatches between source locale and target locales (`error`).
- Invalid BCP 47 locale identifiers (`error`).

The extension MUST NOT claim that an application is fully localized merely because all keys exist.

### FR-013 — Incremental maintenance

The extension SHOULD watch relevant workspace file changes (with debouncing, e.g., `300ms`–`500ms`), mark affected findings as `'stale'`, and allow incremental rescanning of only modified files and locale resources. It MUST NEVER silently rewrite files in the background.

### FR-014 — Reporting

Each scan or apply operation MUST produce a structured report containing:

- Operation status (`OperationStatus`).
- Workspace and scan scope.
- Detected frameworks, i18n libraries, evidence, and confidence.
- File counts: discovered, scanned, excluded, unsupported, failed.
- Candidate counts by `CandidateCategory`, `ConfidenceLevel`, and `CandidateStatus`.
- Keys added, reused, or skipped.
- Locale resource files created or updated.
- Validation issues by `ValidationSeverity`.

---

## 6. Non-Functional Requirements

- **NFR-001 — Offline operation:** All core workflows MUST function with zero network connectivity.
- **NFR-002 — Privacy:** No source code, file paths, translation strings, or telemetry may leave the user's machine.
- **NFR-003 — Performance & Bundling:**
  - The extension MUST be bundled with `esbuild` for fast activation and minimal VSIX size.
  - Long-running scans MUST be asynchronous, support `vscode.CancellationToken`, and yield to avoid blocking the Extension Host UI thread.
- **NFR-004 — Reliability & Idempotency:** Running Scan -> Approve -> Apply twice on the same unchanged project MUST produce zero additional edits on the second run.
- **NFR-005 — Maintainability:** Strict TypeScript (`"strict": true`, no `any`). Core domain logic MUST be decoupled from framework adapters and VS Code UI APIs.
- **NFR-006 — Accessibility:** Use standard VS Code TreeViews, QuickPicks, native Diff editors, and theme-aware Webviews (`vscode-codicons` and CSS variables). Never rely on color alone to convey confidence or severity.

---

## 7. Mandatory Safety and Data-Integrity Constraints

These constraints are non-negotiable:

- **SC-001 — Workspace boundary:** Never write or read outside the opened workspace folders.
- **SC-002 — No destructive defaults:** Never delete source files, delete locale files, delete existing keys, overwrite existing translations, or modify `package.json` dependencies automatically.
- **SC-003 — No automatic execution:** Never execute shell commands, `npm install`, build scripts, or project code automatically.
- **SC-004 — Secrets and sensitive files:** Exclude `.env*`, `*.pem`, `*.key`, `*credentials*`, and secret files by default. Never log their contents.
- **SC-005 — User approval:** No file write may occur without explicit user confirmation. Do not provide a "bypass review" setting in v1.0.
- **SC-006 — Stale edit protection:** Verify SHA-256 `fileContentHash` and exact substring match before applying any `PlannedTextEdit`. Block stale edits immediately.
- **SC-007 — Failure recovery:** Stage edits through `vscode.WorkspaceEdit`. If any part of an apply operation fails, report the exact per-file outcome.

---

## 8. Framework Adapter Contract

Every framework adapter MUST explicitly declare:

- Adapter ID, display name, and version.
- Supported file extensions and framework detection evidence rules.
- Supported AST syntax regions and candidate categories.
- Parser dependencies (e.g., `@vue/compiler-sfc`, `@babel/parser`, `typescript`).
- Supported localization library and resource format.
- Transformation capabilities and known unsupported syntax patterns.

### 8.1 Initial adapter priority (Version 1.0)

1. Generic JSON Resource Adapter (nested & flat JSON, indentation-preserving).
2. HTML Extraction & Transformation Adapter.
3. JavaScript/TypeScript Base AST Adapter.
4. Vue 3 Adapter (`@vue/compiler-sfc` + `vue-i18n`).
5. React JSX/TSX Adapter (`@babel/parser` / TypeScript AST + `react-i18next`).

*(Note: Laravel PHP/Blade Adapter is scheduled for Version 1.1 after v1.0 release gates are met.)*

### 8.2 Adapter quality gate

An adapter cannot be enabled for transformation until it passes unit and fixture tests covering:
- Static text extraction and transformation.
- Dynamic interpolation preservation.
- Preservation of comments, whitespace, quotes, and unrelated code outside the edited range.
- Detection of existing i18n calls (idempotency).
- Graceful fallback to `'needs-manual-handling'` on complex/malformed syntax.

---

## 9. Localization Correctness Rules

- **Interpolation:** Variables MUST retain evaluation semantics. Mismatched variable names across locales MUST be flagged as validation errors.
- **Pluralization:** Follow the target library's plural syntax (`vue-i18n` pipe syntax or `i18next` plural keys). Mark complex inline plural ternaries as `'needs-manual-handling'`.
- **Context & Homographs:** Identical strings in different components/contexts (e.g., "Open" as verb vs. adjective) MUST support context-prefixed keys and MUST NOT be merged blindly if the user selects context-scoped key generation.
- **Formatting & RTL:** Do not auto-rewrite date/number logic or CSS for RTL; report diagnostic suggestions instead.

---

## 10. User Experience & Configuration Requirements

The extension UI SHOULD prioritize native VS Code paradigms for reliability and speed:

1. **Activity Bar Container** with dedicated **TreeViews**:
   - *Overview & Project ProfileView* (detected frameworks, locales, status, quick actions).
   - *Candidate Findings TreeView* (grouped by file or confidence, with inline Approve/Reject/Ignore action icons and click-to-jump to source range).
   - *Locale Resources & Validation TreeView* (missing keys, unused keys, variable mismatches).
2. **Native Diff Preview:** Clicking a proposed transformation opens `vscode.diff` showing the exact before/after code and JSON changes.
3. **Configuration:** Stored in `.vscode/settings.json` or `.i18n-auto-manager.json` (only when created by user confirmation).

---

## 11. Testing Requirements

- **Unit Tests (Vitest):** File discovery/exclusions, SHA-256 hash verification, framework detection, AST range extraction, key generation, collision resolution, JSON resource merging (preserving indentation), interpolation validation, and stale-edit blocking.
- **Adapter Fixture Tests:** Real-world Vue 3 SFC and React TSX fixtures testing static text, props, interpolations, existing `$t`/`t` calls, comments, and unsupported syntax.
- **Integration Tests (`@vscode/test-electron`):** End-to-end flow: Scan -> Approve subset -> Preview Diff -> Apply -> Validate -> Re-scan (verifying idempotency).
- **Safety Tests:** Prove excluded files, `.env` files, rejected findings, and stale files are never modified.

---

## 12. Acceptance Criteria for Version 1.0 Release

Version 1.0 is acceptable only when all 17 criteria pass:

1. Extension activates cleanly in VS Code without network access.
2. User can scan a workspace offline with cancellation and progress reporting.
3. Extension reports detected frameworks (Vue 3, React, JS/TS, HTML) and existing i18n setup with evidence and confidence.
4. User can inspect candidates, filter by confidence/status, and see reason codes.
5. `'low'` and `'unknown'` confidence candidates are never auto-transformed.
6. User can configure source and target locales (validated against BCP 47).
7. Key generator produces deterministic, collision-safe keys.
8. User can preview exact source and JSON diffs via VS Code's diff viewer.
9. Changes are applied only after explicit user approval.
10. Range-based edits preserve surrounding code formatting, comments, and imports.
11. Existing translation values in JSON files are preserved 100%.
12. Interpolation variables are preserved in code and validated across locales.
13. Stale edits (files modified after scan) are detected via hash check and blocked.
14. Validation catches missing keys, invalid JSON, and variable mismatches.
15. Repeated runs on the same codebase are idempotent (no duplicate keys or imports).
16. Unsupported syntax is honestly marked as `'needs-manual-handling'`.
17. Structured operation reports accurately reflect `'completed-successfully'`, `'completed-with-warnings'`, `'partially-completed'`, `'cancelled'`, or `'failed'`.

---

## 13. Implementation-Agent Rules

Any AI coding agent working on this repository MUST follow these rules:

1. Read both `PROJECT_BLUEPRINT_I18N_AUTO_MANAGER.md` and this document before writing code.
2. Inspect the existing repository before proposing or creating files.
3. Implement ONLY the milestone explicitly requested by the user. Do not implement future milestones "for convenience".
4. Do not replace working architecture or dependencies without explaining why and obtaining approval.
5. Do not invent framework support, parser behavior, or test results.
6. Never use regular expressions as the primary source-code parser or transformation mechanism; use AST node offsets and range edits.
7. Never add AI APIs, telemetry, cloud services, or network dependencies to the core.
8. Do not modify sample project files outside test runs.
9. Never use TypeScript `any`, `@ts-ignore`, or disable ESLint rules just to make checks pass.
10. Never leave placeholder/stub functions that are reported as implemented.
11. Never claim a build, lint, or test command passed unless it was actually executed and its terminal output observed.
12. If a requirement cannot be implemented safely, stop that part, explain the limitation, and propose the smallest safe alternative.
13. Update documentation and unit tests whenever behavior or public types change.

### Required completion report format

Every milestone report MUST contain:

- **Milestone:** Exact milestone ID and title.
- **Implemented:** Concrete behavior completed.
- **Files changed:** Exact relative paths created or updated.
- **Tests run:** Exact commands executed and observed pass/fail outputs.
- **Not implemented:** Anything deferred or out of scope for the current milestone.
- **Known risks:** Remaining edge cases or limitations.
- **Acceptance checklist:** Pass/Fail status for each criterion of the current milestone.
- **Next step:** Name of the next milestone only; wait for user authorization before starting it.

---

## 14. Definition of Done

A requirement or milestone is **Done** only when:
- The implementation exists in strict TypeScript with zero compile or lint errors.
- The implementation adheres to all safety and formatting-preservation constraints.
- Unit/fixture tests cover normal, edge, and failure cases and pass when executed.
- The user-facing behavior is documented and actual test outputs are reported.

**Final rule:** When uncertain, do not guess and do not modify user code. Preserve the original project, surface the uncertainty in the UI/report, and let the developer decide.