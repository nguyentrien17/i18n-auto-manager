# Requirements and Constraints — i18n Auto Manager

**Document type:** Product requirements, engineering constraints, and acceptance contract  
**Product:** i18n Auto Manager — a local-first Visual Studio Code extension  
**Status:** Normative specification for implementation agents  
**Language:** English  
**Purpose:** Prevent an AI coding agent or developer from making unsafe assumptions, implementing unsupported behavior, or claiming that localization is complete when it is not.

---

## 1. How to Interpret This Document

The words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative:

- **MUST / MUST NOT:** mandatory requirement. A release or implementation milestone is not complete if violated.
- **SHOULD / SHOULD NOT:** strong recommendation. Deviations require a documented reason.
- **MAY:** optional capability.

When requirements conflict, apply this priority order:

1. Preserve user data and source-code correctness.
2. Never make an unreviewed destructive change.
3. Preserve runtime behavior and application logic.
4. Produce valid, maintainable localization output.
5. Maximize automation and convenience.
6. Optimize speed and visual polish.

A safer partial result is preferable to an unsafe complete-looking result.

## 2. Product Definition

i18n Auto Manager is a VS Code extension that helps developers add, maintain, and validate internationalization in existing software projects.

The extension MUST operate locally by default. Its core workflow MUST NOT depend on an AI model, remote API, cloud account, or network connection.

The extension is an **assisted code transformation tool**, not a universal automatic translator and not a guarantee that every project can be internationalized without developer review.

### 2.1 Primary user outcome

A developer can:

1. Open an existing project in VS Code.
2. Ask the extension to inspect the workspace.
3. Review detected frameworks, source files, existing localization libraries, and candidate user-facing strings.
4. Configure a source locale, target locales, resource format, and framework-specific settings.
5. Review proposed code changes and translation resources.
6. Apply only approved changes.
7. Run validation and receive an actionable report.
8. Repeat the process after the project changes.

### 2.2 Core product principles

- Local-first and offline-capable.
- Explicit user approval before source changes.
- Framework-aware, parser-based transformations.
- Conservative detection: uncertainty must be visible.
- Idempotent operations: running the same operation twice must not create duplicate keys or repeatedly rewrite code.
- Reversible changes through source control and clear change previews.
- No false claims that untranslated source text is a completed translation.
- Incremental scanning rather than rescanning the entire workspace unnecessarily.

## 3. Scope and Non-Goals

### 3.1 Required initial scope

The first stable product should focus on a deliberately limited set of scenarios:

- VS Code desktop extension.
- TypeScript implementation.
- Local workspace inspection.
- JavaScript and TypeScript source analysis.
- Vue 3 single-file components, including template and script regions.
- React JSX/TSX analysis.
- HTML analysis.
- JSON localization resource files.
- Existing i18n configuration detection where it can be identified reliably.
- Candidate review, key generation, resource generation, safe source edits, and validation.

Laravel support MUST be treated as a separate adapter with its own parser and tests. It MUST NOT be claimed as supported merely because the extension can find PHP files.

### 3.2 Explicit non-goals for the initial release

The initial release MUST NOT attempt to:

- Understand every programming language or framework.
- Translate arbitrary text with guaranteed semantic accuracy.
- Automatically infer the correct meaning of every string.
- Rewrite database content, API responses, CMS content, or external service data.
- Modify application business logic to make a project fit the extension.
- Automatically install dependencies or execute project scripts.
- Change backend APIs, database schemas, or deployment configuration.
- Guarantee that translated text fits every screen or design.
- Automatically localize images, audio, video, PDFs, or other media.
- Make changes outside the opened workspace.
- Upload source code or translation content to external services.

Unsupported cases MUST be reported as unsupported or requiring manual work. They MUST NOT be silently skipped while reporting full success.

## 4. Functional Requirements

### FR-001 — Workspace discovery

The extension MUST:

- Operate only on the workspace folders explicitly opened by the user.
- Support single-root and multi-root workspaces.
- Respect `.gitignore` where applicable and provide extension-specific exclude patterns.
- Exclude generated, dependency, build, cache, and vendor directories by default, including common examples such as `node_modules`, `vendor`, `dist`, `build`, `.git`, and coverage folders.
- Allow users to inspect and change include/exclude patterns.
- Avoid reading binary files as source text.
- Avoid following symbolic links outside the workspace unless the user explicitly enables that behavior.
- Display the number of discovered, included, excluded, and unsupported files.

The extension MUST NOT assume that every file in the workspace belongs to the application.

### FR-002 — Project and framework detection

The extension SHOULD identify project technologies using multiple evidence sources, such as:

- `package.json`
- framework configuration files
- file extensions and directory patterns
- dependency declarations
- existing localization configuration
- source syntax

Detection MUST include a confidence level and evidence.

If multiple frameworks are present, the extension MUST report all detected frameworks rather than selecting one arbitrarily.

If evidence is insufficient, the extension MUST display **Unknown / Needs confirmation** and allow the user to choose the adapter.

The extension MUST NOT modify project files merely because a framework was detected.

### FR-003 — Existing i18n configuration inspection

Before proposing a new localization setup, the extension MUST inspect for existing configuration and resource files.

It MUST attempt to identify:

- Existing localization libraries.
- Existing locale directories and files.
- Default or fallback locale.
- Existing key naming conventions.
- Existing interpolation conventions.
- Existing pluralization conventions.
- Existing namespace or domain organization.

If an existing i18n system is detected, the extension SHOULD integrate with it instead of creating a competing system.

If two or more plausible configurations conflict, the extension MUST ask the user to select one or mark the setup as unresolved.

The extension MUST NOT replace an existing i18n library or configuration automatically.

### FR-004 — Candidate string extraction

The extension MUST extract candidate user-facing strings only from supported syntax regions.

Examples include:

- Vue template text and relevant attributes.
- JSX/TSX text nodes and relevant attributes.
- JavaScript/TypeScript string literals in recognized UI contexts.
- HTML text nodes and relevant attributes.
- Explicit localization calls and existing translation keys.

The extractor MUST distinguish at least:

1. Static user-facing text.
2. Dynamic text or expressions.
3. Technical strings, such as identifiers, paths, log messages, API values, and internal constants.
4. Existing localization calls.
5. Uncertain candidates.

The extractor MUST preserve source locations, syntax context, file path, and enough structural information to propose a safe edit.

The extension MUST NOT classify every string literal as translatable.

### FR-005 — Candidate classification

Every candidate MUST have:

- A stable candidate ID for the current scan result.
- File path and source range.
- Original text or a safe representation of it.
- Syntax context.
- Candidate category.
- Confidence level: High, Medium, Low, or Unknown.
- A reason explaining why it was classified.
- A review state: Pending, Approved, Rejected, or Needs manual handling.

Default behavior:

- High confidence: may be preselected for review, but MUST still be visible before application.
- Medium confidence: requires explicit approval.
- Low or Unknown: MUST NOT be auto-applied.

The extension MUST provide filters for confidence, file, category, and review state.

### FR-006 — Dynamic content and interpolation

The extension MUST preserve dynamic expressions and their evaluation order.

For example, a source expression conceptually equivalent to:

`Hello, {user.name}`

must become a localized message with an explicit interpolation variable, not a literal string containing source-code syntax.

The extension MUST:

- Identify interpolated values where supported by the parser.
- Generate stable variable names where they can be inferred safely.
- Ask for review when variable names or expression boundaries are ambiguous.
- Preserve escaping and type-sensitive behavior.
- Avoid moving expressions across scopes.
- Avoid evaluating expressions during scanning.

If a string combines static text and complex expressions that cannot be transformed safely, the extension MUST leave it unchanged and report the reason.

### FR-007 — Key generation

Generated keys MUST:

- Be deterministic for the same approved candidate and configuration.
- Follow the configured naming convention.
- Avoid collisions with existing keys.
- Avoid overwriting existing values.
- Preserve existing keys even if they are not generated by this extension.
- Support namespaces or file-based grouping when the selected adapter supports them.
- Be stable across repeated scans when the source context has not materially changed.

The extension MUST detect duplicate or conflicting generated keys before writing resources.

The extension MUST NOT silently rename existing keys or rewrite every key in a project to enforce a preferred style.

### FR-008 — Translation resource generation

The extension MUST support the resource format selected by the active adapter.

For the initial generic resource engine, JSON SHOULD be supported.

The extension MUST:

- Preserve valid JSON structure and encoding.
- Preserve existing values.
- Add only approved, missing entries.
- Respect configured locale identifiers.
- Keep source-locale values separate from target-locale values.
- Make missing translations explicit.
- Preserve interpolation variable names across locales.
- Validate pluralization structures according to the selected i18n library.

If no translation is available, the extension MAY copy the source-language text into a target-locale entry only when the user explicitly chooses this behavior. Such entries MUST be marked as **untranslated / needs translation** in the report or metadata supported by the project.

The extension MUST NOT label copied source text as a completed translation.

### FR-009 — Translation input

The initial release MUST support manual translation entry and importing user-provided translation resources.

The initial release MUST NOT require an AI service.

Optional machine-translation providers MAY be added in the future, but they MUST be disabled by default and require explicit user configuration and consent.

If an external provider is added later, the extension MUST disclose what content will be sent, obtain confirmation, and provide a local-only mode that remains usable.

### FR-010 — Source-code transformation

Source edits MUST use syntax-aware transformations wherever supported.

The extension SHOULD use ASTs, parser node ranges, or framework-specific compiler/parser APIs. Regular expressions MUST NOT be the primary mechanism for rewriting JavaScript, TypeScript, Vue, JSX, or TSX source code.

Every proposed edit MUST include:

- Original source range.
- Original source text or a verifiable source hash.
- Replacement text.
- Reason for the transformation.
- Associated resource key.
- Adapter and transformation rule version.

Before applying an edit, the extension MUST verify that the source still matches the version that was scanned. If the file changed, the edit MUST be marked stale and rescanned.

The extension MUST:

- Preserve unrelated source code.
- Preserve comments where possible.
- Preserve formatting as much as practical.
- Avoid changing imports that already exist.
- Avoid adding duplicate imports.
- Avoid changing runtime behavior beyond replacing the approved text with a localization lookup.
- Use VS Code WorkspaceEdit or an equivalent supported editing API.
- Apply related source and resource changes as one coherent user-approved operation where possible.

### FR-011 — Review and application

The extension MUST provide a review step before applying changes.

The review interface MUST show:

- File name and location.
- Original code.
- Proposed code.
- Generated key.
- Source-locale resource value.
- Target-locale status.
- Confidence and classification reason.
- Any warnings.

The user MUST be able to approve or reject individual candidates and groups of candidates.

The extension MUST NOT apply changes during scanning, framework detection, or preview generation.

Before applying, the extension MUST revalidate all affected files and detect stale edits or key conflicts.

If any critical validation fails, the extension MUST stop the affected transaction and explain why. It MUST NOT report a partially applied operation as fully successful.

### FR-012 — Validation

The extension MUST validate at least:

- Missing keys referenced by source code.
- Missing locale entries.
- Unused resource keys, where reliable.
- Duplicate keys.
- Duplicate or conflicting resource paths.
- Invalid JSON resources.
- Interpolation variable mismatches.
- Invalid or unsupported locale identifiers.
- Source references that do not resolve to a resource entry.
- Syntax errors introduced by proposed transformations, to the extent supported by the parser.

Validation results MUST identify the affected file and key, explain the problem, and suggest a safe next action.

The extension MUST distinguish **Error**, **Warning**, **Information**, and **Not checked**.

The extension MUST NOT claim that an application is fully localized merely because all keys exist. Translation quality, context, cultural correctness, layout fit, and runtime behavior require separate review.

### FR-013 — Incremental maintenance

The extension SHOULD watch relevant workspace changes and mark affected scan results as outdated.

It MUST avoid silently applying edits in response to file changes.

It SHOULD rescan only affected files and dependencies where possible.

It MUST detect when a previously reviewed candidate no longer matches the source.

### FR-014 — Reporting

Each scan or apply operation MUST produce a report containing:

- Workspace and scan scope.
- Detected frameworks and confidence.
- Files scanned, excluded, unsupported, and failed.
- Candidates by category and confidence.
- Approved, rejected, skipped, and transformed candidates.
- Keys added or reused.
- Locale resources changed.
- Validation errors and warnings.
- Operations not performed and their reasons.

The report MUST distinguish between:
- Completed successfully.
- Completed with warnings.
- Partially completed.
- Cancelled.
- Failed.

## 5. Non-Functional Requirements

### NFR-001 — Offline operation

Core scanning, classification, key generation, transformation, resource writing, and validation MUST work without internet access.

### NFR-002 — Privacy

By default, source code, file paths, translation content, and scan reports MUST remain on the user's machine.

The extension MUST NOT transmit project content, telemetry containing source text, or file contents to external services.

If optional telemetry is introduced, it MUST be opt-in, documented, and exclude source text and secrets.

### NFR-003 — Performance

The extension MUST:

- Avoid blocking the VS Code UI thread with long-running scans.
- Support cancellation.
- Show progress for long operations.
- Avoid loading excluded or binary files.
- Use incremental scanning where practical.
- Handle large workspaces without assuming all source files fit in memory simultaneously.

Performance targets MUST be measured against a documented test workspace. Do not claim a speed target without measurement.

### NFR-004 — Reliability

- All file operations MUST handle read/write failures.
- Cancellation MUST leave the workspace in a consistent state.
- Errors MUST be actionable and logged without exposing secrets.
- The extension MUST not swallow parser or filesystem errors.
- Re-running the same operation MUST not duplicate keys or transformations.

### NFR-005 — Maintainability

- Framework-specific behavior MUST be isolated in adapters.
- Core domain logic MUST not depend directly on Vue, React, Laravel, or UI implementation details.
- Parser and transformation versions MUST be explicit.
- Types MUST be strict.
- Public interfaces MUST be documented.
- Tests MUST cover normal, edge, and failure cases.

### NFR-006 — Accessibility and usability

The extension SHOULD support keyboard navigation, clear labels, meaningful status messages, and VS Code theme conventions.

Warnings and confidence states MUST use text or icons in addition to color.

## 6. Mandatory Safety and Data-Integrity Constraints

These constraints are non-negotiable.

### SC-001 — Workspace boundary

The extension MUST NOT write outside the explicitly opened workspace folders.

Any exception requires a separate, explicit user action and a clear path preview.

### SC-002 — No destructive defaults

The extension MUST NOT:

- Delete source files.
- Delete existing locale files.
- Delete existing translation keys.
- Overwrite existing translations.
- Rename or move project files.
- Change package dependencies.
- Change build or deployment configuration.

Such operations are outside the initial scope. If a future feature needs them, it must use a separate explicit confirmation flow.

### SC-003 — No automatic execution

The extension MUST NOT automatically execute:

- Shell commands.
- Package-manager scripts.
- Build scripts.
- Project code.
- Database migrations.
- Network requests defined by the project.

It MAY use VS Code APIs and installed parser libraries within the extension process. Any future command execution feature requires explicit user approval and a visible command preview.

### SC-004 — Secrets and sensitive files

The scanner MUST exclude common secret, environment, credential, and private-key files by default, including `.env*`, private keys, credential files, and user-configured exclusions.

The extension MUST not display secret values in logs, reports, diagnostics, or telemetry.

### SC-005 — User approval

No source or resource modification may occur without explicit user approval for the proposed changes.

A setting that bypasses review MUST NOT be provided in the initial release.

### SC-006 — Stale edit protection

An edit generated from an old source snapshot MUST never be applied blindly.

If source text or relevant structure has changed since scanning, the extension MUST invalidate the edit and require a new scan or review.

### SC-007 — Failure recovery

If an operation fails after some files have been written, the extension MUST report exactly which files changed and which did not.

The implementation SHOULD stage changes and apply them in a controlled way to reduce partial-write risk.

It MUST NOT claim that the extension can roll back arbitrary user or external changes. Source control remains the recommended recovery mechanism.

## 7. Framework Adapter Contract

Every framework adapter MUST explicitly declare:

- Adapter ID and version.
- Supported file extensions.
- Framework detection evidence.
- Supported syntax regions.
- Supported candidate types.
- Parser/compiler dependencies.
- Supported localization library or resource format.
- Transformation capabilities.
- Known unsupported syntax.
- Validation capabilities.
- Test fixtures and compatibility range.

An adapter MUST return **Unsupported** or **Needs manual review** for syntax it cannot safely transform.

An adapter MUST NOT fall back to broad regular-expression rewriting when parsing fails.

### 7.1 Initial adapter priority

1. Generic JSON resource adapter.
2. JavaScript/TypeScript extraction adapter.
3. Vue 3 adapter.
4. React JSX/TSX adapter.
5. HTML adapter.
6. Laravel adapter only after a separately tested PHP parsing strategy is selected.

A framework being detected does not mean all of its syntax is supported.

### 7.2 Adapter quality gate

An adapter cannot be marked stable until it has:

- Parser-based extraction tests.
- Transformation tests.
- Resource-generation tests.
- Validation tests.
- Tests for comments, nested expressions, escaping, and malformed input.
- Tests proving unrelated source text is preserved.
- Documented unsupported cases.

## 8. Localization Correctness Rules

The extension MUST treat localization as more than replacing visible words.

### 8.1 Interpolation

- Variables MUST retain their values and evaluation behavior.
- Variable names MUST be consistent across locale resources.
- Missing or extra interpolation variables MUST be reported.
- Expressions with side effects MUST NOT be duplicated or evaluated more than once.

### 8.2 Pluralization

Plural forms MUST follow the selected localization library's rules and locale capabilities.

The extension MUST NOT assume that every language uses singular/plural as a simple two-form rule.

If the selected adapter cannot represent the required plural categories, the candidate MUST be marked unsupported or require manual handling.

### 8.3 Context and duplicate text

Identical source text can have different meanings in different contexts. For example, “Open” may be a verb or an adjective.

The extension SHOULD support context-aware keys and MUST NOT merge candidates solely because their visible text is identical.

### 8.4 Formatting

The extension SHOULD encourage locale-aware date, time, number, and currency formatting where it can identify relevant APIs safely.

It MUST NOT automatically change numeric or date logic based only on a string scan.

### 8.5 Right-to-left languages

RTL support requires application-level layout testing. The extension MAY report likely RTL considerations, but MUST NOT claim that changing locale resources alone makes an application RTL-compatible.

## 9. User Experience Requirements

The extension SHOULD provide these primary areas:

1. **Overview:** project detection, current localization status, and next recommended action.
2. **Scan:** scan scope, progress, cancellation, and scan summary.
3. **Findings:** candidates with filters and confidence indicators.
4. **Configuration:** source locale, target locales, adapter, resource format, key style, and exclusions.
5. **Review:** proposed code and resource diffs with per-item approval.
6. **Validation:** errors, warnings, missing keys, and unresolved candidates.
7. **History / Reports:** previous operations and their results.

The interface MUST clearly distinguish:
- Detected facts.
- Inferred suggestions.
- User-confirmed configuration.
- Proposed changes.
- Applied changes.

No screen may imply that a proposed change has already been written.

## 10. Configuration Requirements

Configuration SHOULD be stored in VS Code workspace settings or a project-level configuration file only after the user chooses to create one.

The configuration MUST include, where relevant:

- Source locale.
- Target locales.
- Selected adapter.
- Existing i18n library.
- Resource directory and naming convention.
- Key naming strategy.
- Include/exclude patterns.
- Interpolation convention.
- Pluralization convention.
- Review preferences.

The extension MUST validate configuration before using it.

It MUST NOT silently change configuration to make an operation pass.

## 11. Testing Requirements

### 11.1 Unit tests

Required for:

- File discovery and exclusion.
- Framework detection.
- Candidate classification.
- Key generation and collision detection.
- Interpolation extraction.
- Locale resource merge.
- Missing/unused key validation.
- Stale edit detection.

### 11.2 Adapter tests

Each adapter MUST use real source fixtures and expected extraction/transformation results.

Fixtures MUST include:
- Simple static text.
- Nested templates or components.
- Dynamic expressions.
- Escaped quotes and special characters.
- Comments.
- Existing localization calls.
- Duplicate visible text in different contexts.
- Malformed or unsupported syntax.

### 11.3 Integration tests

The project MUST test a complete flow:

1. Open a fixture workspace.
2. Scan it.
3. Review candidates.
4. Approve a subset.
5. Preview changes.
6. Apply changes.
7. Validate resources and references.
8. Run the same operation again and verify idempotency.

### 11.4 Safety tests

Tests MUST prove that:

- Excluded files are not modified.
- Files outside the workspace are not modified.
- Existing resource values are preserved.
- Rejected candidates are not changed.
- Stale edits are blocked.
- Cancellation does not leave unreported partial changes.
- Secrets are not emitted in logs.

## 12. Acceptance Criteria for the First Usable Release

The first usable release is acceptable only when all of the following are true:

1. The extension activates in a supported VS Code version.
2. The user can scan a sample workspace without network access.
3. The extension reports detected project technologies with evidence and confidence.
4. The user can inspect candidates and understand why each was detected.
5. Low-confidence candidates are not silently transformed.
6. The user can configure source and target locales.
7. The user can generate deterministic, collision-safe keys.
8. The user can preview source and resource changes.
9. The user must explicitly approve changes before they are applied.
10. Existing translation values are preserved.
11. Interpolation variables are preserved or the candidate is blocked for manual review.
12. Validation identifies missing keys and invalid resource files.
13. Repeated runs do not create duplicate keys or repeated transformations.
14. The extension works offline for its core features.
15. Tests cover the supported adapters and mandatory safety constraints.
16. Unsupported syntax is reported honestly.
17. The report distinguishes success, warnings, partial completion, cancellation, and failure.

A feature is not complete merely because its UI exists. It is complete only when its behavior, failure handling, tests, and documentation satisfy this contract.

## 13. Implementation-Agent Rules

Any AI coding agent working on this repository MUST follow these rules:

1. Read the product blueprint and this requirements document before editing code.
2. Inspect the existing repository before proposing or creating files.
3. Implement only the milestone explicitly requested by the user.
4. Do not implement future milestones “for convenience”.
5. Do not replace working architecture or dependencies without explaining why and obtaining approval.
6. Do not invent framework support, parser behavior, or test results.
7. Do not use regex as the primary source-code transformation mechanism.
8. Do not add AI APIs, telemetry, cloud services, or network dependencies to the core.
9. Do not modify sample project files outside the extension's own test fixtures.
10. Do not suppress TypeScript, lint, parser, or test errors merely to make checks pass.
11. Do not leave placeholder functions that are reported as implemented.
12. Do not claim a test passed unless it was actually run and its result was observed.
13. If a requirement cannot be implemented safely, stop that part, explain the limitation, and propose the smallest safe alternative.
14. Update documentation and tests whenever behavior or public configuration changes.
15. At the end of each milestone, report changed files, implemented behavior, tests actually run, known limitations, and the next milestone's prerequisites.

### Required completion report format

Every milestone report MUST contain:

- **Milestone:** exact milestone ID and title.
- **Implemented:** concrete behavior completed.
- **Files changed:** actual paths.
- **Tests run:** exact commands and observed results.
- **Not implemented:** anything deferred or incomplete.
- **Known risks:** remaining limitations.
- **Acceptance checklist:** pass/fail for each applicable criterion.
- **Next step:** the next milestone only; do not start it without authorization.

## 14. Definition of Done

A requirement is Done only when:

- The implementation exists.
- The implementation follows the relevant constraints.
- The behavior is covered by tests.
- Failure cases are handled.
- The user-facing behavior is documented.
- The feature does not violate workspace, privacy, or approval boundaries.
- The actual test results are reported.

**Final rule:** When uncertain, do not guess and do not modify. Preserve the original project, show the uncertainty, and ask the developer to decide.
