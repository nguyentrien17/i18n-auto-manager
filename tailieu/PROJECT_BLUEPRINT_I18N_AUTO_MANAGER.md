# i18n Auto Manager — VS Code Extension
## Product Requirements, Technical Architecture, Implementation Workflow, and Vibe-Coding Guide

**Document version:** 1.0  
**Status:** Implementation blueprint  
**Product type:** Local-first Visual Studio Code extension  
**Primary objective:** Help developers internationalize existing software projects safely, without requiring AI or a cloud service.

---

## 1. Product Vision

Build **i18n Auto Manager**, a VS Code extension that scans an existing codebase, identifies user-facing text, helps developers convert that text into localization messages, generates translation resources, and validates that all configured locales remain consistent.

The extension must be useful in **offline mode**. It must not require an AI provider, API key, account, remote server, or network connection for its core workflow.

The product should automate deterministic engineering tasks while clearly identifying cases that require human judgment.

### 1.1 Core promise

A developer should be able to:

1. Open an existing project in VS Code.
2. Run a workspace localization scan.
3. Review detected user-facing strings and their source locations.
4. Select a source locale and one or more target locales.
5. Choose or confirm the project's localization strategy.
6. Preview proposed source-code and translation-file changes.
7. Apply approved changes safely.
8. Validate missing keys, unused keys, interpolation variables, and resource consistency.
9. Continue detecting new strings as the project evolves.

### 1.2 Product principles

- **Local-first:** source code stays on the developer's machine.
- **No AI dependency:** core functionality is deterministic.
- **Safe by default:** never silently rewrite an entire project.
- **Reviewable changes:** every proposed edit must be inspectable.
- **Framework-aware:** use dedicated adapters rather than one universal rewrite rule.
- **Incremental:** support scanning and updating only changed files.
- **Extensible:** allow new framework and localization adapters to be added.
- **Honest detection:** distinguish confirmed strings from uncertain candidates.
- **Reversible:** support preview, backups or workspace edits, and undo where possible.
- **Non-destructive:** preserve existing localization systems and project conventions whenever possible.

---

## 2. Goals and Non-Goals

### 2.1 Goals

The extension must:

- Detect the workspace's languages, frameworks, and existing i18n libraries.
- Scan supported source files for likely user-facing text.
- Exclude generated files, dependencies, build output, and configured paths.
- Extract candidate strings with source locations and context.
- Classify findings by confidence and explain why each was detected.
- Generate stable translation keys.
- Convert supported source patterns into calls to the project's localization API.
- Generate or update locale resource files.
- Preserve interpolation variables and message structure.
- Detect missing, unused, duplicate, and inconsistent translation keys.
- Provide a review interface before modifying files.
- Support incremental rescans.
- Provide diagnostics and a workspace report.
- Work offline for all core features.

### 2.2 Non-goals for the first release

Do not attempt to:

- Guarantee linguistically perfect translations.
- Automatically understand every runtime-generated message.
- Translate arbitrary database content or user-generated content.
- Rewrite every framework or programming language.
- Guarantee visual correctness without running the application.
- Infer business meaning from ambiguous strings.
- Replace professional localization management platforms.
- Execute arbitrary project scripts automatically.
- Modify files outside the selected workspace.
- Use AI or remote translation services in the core product.

Optional translation providers may be considered in a later release, but must remain opt-in and isolated from the core engine.

---

## 3. Target Users and Main Use Cases

### 3.1 Target users

- Individual software developers.
- Frontend and backend developers.
- Small development teams.
- Maintainers of legacy projects.
- Developers preparing products for international markets.

### 3.2 Main use cases

**UC-01: Add localization to a project that has none**
- Detect the project technology.
- Recommend a supported localization integration.
- Let the developer choose or confirm it.
- Extract eligible strings.
- Generate locale resources and source changes.

**UC-02: Add another locale to an existing localized project**
- Detect the current i18n setup.
- Read existing locale resources.
- Identify missing target-locale keys.
- Generate empty or dictionary-assisted entries.
- Do not replace existing translations.

**UC-03: Audit an existing localized project**
- Find hard-coded text candidates.
- Find missing translation keys.
- Find unused keys.
- Find duplicate or suspiciously similar keys.
- Compare interpolation variables across locales.

**UC-04: Keep localization synchronized**
- Watch relevant file changes.
- Mark the scan as stale when source files change.
- Offer an incremental scan.
- Report newly introduced hard-coded text and missing keys.

**UC-05: Review and safely apply changes**
- Show source and proposed code side by side.
- Show locale resource changes.
- Let the developer accept or reject each finding.
- Apply accepted edits as a single workspace operation where feasible.

---

## 4. Supported Technology Strategy

Do not claim universal support. Implement a capability-based adapter system.

### 4.1 Initial support matrix

| Technology | Initial capability | Suggested integration |
|---|---|---|
| HTML | Static text extraction and review | Configurable |
| JavaScript | Candidate string extraction | Project-specific |
| TypeScript | Candidate string extraction | Project-specific |
| Vue 3 | Template extraction and selected script patterns | Vue I18n |
| React JSX/TSX | JSX text and selected attributes | i18next or react-intl |
| Laravel PHP | Blade and selected PHP patterns | Laravel lang files / `__()` |
| Laravel Blade | Text extraction and selected directive conversion | Laravel localization |
| JSON locale files | Read, validate, update | Native JSON |
| YAML locale files | Later capability | YAML parser |

### 4.2 Later support

Consider Angular, Flutter, Django, Java/Spring, Rails, Symfony, Svelte, and other ecosystems only after the adapter contract and test strategy are stable.

### 4.3 Adapter capability levels

Every adapter must declare capabilities explicitly:

- `detect`: detect project/framework.
- `scan`: identify candidate strings.
- `parseResources`: read existing locale resources.
- `transform`: produce safe source edits.
- `writeResources`: update locale resources.
- `validate`: validate integration-specific rules.
- `watch`: identify files that should trigger rescans.

A framework may initially support scan-only mode. Do not enable automatic transformation until tests prove it safe.

---

## 5. User Experience and Main Screens

The extension should provide:

1. Activity Bar icon and sidebar.
2. Workspace overview.
3. Project detection report.
4. Scan results table.
5. Localization setup wizard.
6. Locale selection interface.
7. Review and apply interface.
8. Translation resource explorer.
9. Validation report.
10. Extension settings.

### 5.1 Sidebar structure

Suggested navigation:

- Overview
- Scan Project
- Findings
- Translation Files
- Validation
- Settings

### 5.2 Overview screen

Show:

- Workspace name.
- Detected languages/frameworks.
- Detected localization library.
- Source locale.
- Configured target locales.
- Number of files scanned.
- Candidate strings found.
- High/medium/low confidence counts.
- Missing translation count.
- Last scan timestamp.
- Current scan status.

### 5.3 Setup wizard

The wizard should ask for:

1. Source locale.
2. Target locales.
3. Framework/integration choice, with auto-detection as a suggestion.
4. Resource format and destination.
5. Key naming strategy.
6. Whether to include selected file types.
7. Whether to scan tests, examples, and documentation.
8. Whether to enable file watching.

Allow users to cancel or return to previous steps.

### 5.4 Findings interface

Each finding should show:

- Original text.
- File path and line/column.
- Framework and syntax context.
- Suggested translation key.
- Confidence level.
- Detection reason.
- Proposed source transformation.
- Whether the finding is safe to transform.
- Exclusion/ignore controls.

Bulk actions must be limited to findings that pass adapter safety checks.

### 5.5 Review interface

Before applying changes, show:

- Files to be modified.
- Source-code diff.
- New or updated locale files.
- Existing translations that will remain unchanged.
- Warnings and unresolved findings.
- Number of accepted/rejected changes.

Require explicit user confirmation before writing files.

---

## 6. End-to-End Workflow

### Phase 1: Workspace discovery

1. Obtain the active workspace folders from VS Code.
2. If there are multiple roots, let the user choose one or scan all selected roots.
3. Read workspace settings and extension configuration.
4. Apply ignore rules.
5. Identify package and framework manifests.
6. Detect likely source roots and locale directories.
7. Detect existing localization libraries and conventions.
8. Generate a project profile.

Do not read outside the workspace unless the user explicitly selects another folder.

### Phase 2: Framework and localization detection

Inspect relevant files such as:

- `package.json`
- `composer.json`
- `pubspec.yaml`
- `angular.json`
- `vite.config.*`
- `webpack.config.*`
- Existing locale resource files
- Existing i18n initialization files
- Framework-specific configuration

Detection must be evidence-based. Return:
- Detected framework.
- Detected localization library.
- Evidence file paths.
- Confidence.
- Conflicts or ambiguity.

If multiple frameworks or localization libraries are present, ask the user to choose the relevant application/module.

### Phase 3: File discovery and scanning

1. Use VS Code workspace file discovery.
2. Respect `.gitignore` where feasible and extension-specific exclusions.
3. Exclude `node_modules`, `vendor`, `dist`, `build`, `.git`, coverage, generated code, and binary files by default.
4. Use language-aware parsers for supported syntax.
5. Extract candidate user-facing strings.
6. Preserve exact source ranges.
7. Deduplicate only when safe; never merge semantically different occurrences merely because text is identical.
8. Store findings in an in-memory scan model or a small workspace cache.

### Phase 4: Candidate classification

Classify each candidate as:

- `high`: clearly user-visible static text in a supported context.
- `medium`: likely user-visible but needs review.
- `low`: ambiguous string or context.
- `ignored`: excluded by rule or user decision.

Examples:
- HTML text node: usually high.
- Vue template text node: usually high.
- JSX text node: usually high.
- `placeholder`, `title`, `aria-label`: usually high.
- Log message: usually not user-facing.
- URL, route, CSS class, identifier: not a translation candidate.
- Database query, enum, status code: do not automatically translate.
- String used in both UI and logic: require review.
- Runtime-generated text: report as unsupported or uncertain.

Every classification must include a reason code.

### Phase 5: Localization setup

If no i18n integration exists:

1. Recommend an integration supported by the detected framework.
2. Show the required dependency and configuration changes.
3. Explain each proposed change.
4. Do not install packages automatically without explicit approval.
5. Generate setup changes only after confirmation.

If i18n already exists:

1. Detect its configuration.
2. Read its resource structure.
3. Reuse its function names, key style, namespace rules, and file layout.
4. Do not create a competing localization system.
5. Report unsupported or ambiguous configurations.

### Phase 6: Key generation

Support configurable strategies:

- `feature.group.name`
- `feature_group_name`
- `group.name`
- Existing project convention

Example:

```text
customer.title
customer.create
common.cancel
auth.login.success
```

Rules:

- Stable keys should not depend on line numbers.
- Avoid keys based solely on translated text when collisions are likely.
- Use file/module context where helpful.
- Detect collisions before proposing edits.
- Preserve existing keys.
- Do not rename existing keys automatically.
- Allow manual key editing.
- Record the source finding associated with each key.

### Phase 7: Source transformation

For each accepted finding:

1. Confirm that the adapter supports transformation for this syntax.
2. Confirm that the source range is still current.
3. Generate a minimal edit.
4. Preserve formatting and surrounding code.
5. Preserve interpolation variables.
6. Preserve escaping and template syntax.
7. Avoid changing control flow or business logic.
8. Run syntax validation where available.
9. Show the diff before applying.

If the source has changed since scanning, mark the finding stale and rescan it.

### Phase 8: Translation resource generation

Generate or update locale resources using the project's existing format.

Example:

```json
{
  "common": {
    "cancel": "Cancel",
    "save": "Save"
  },
  "customer": {
    "title": "Customer Management"
  }
}
```

Offline translation options:

- Create empty target-locale values.
- Copy source values as temporary placeholders only when clearly marked.
- Use a bundled, licensed dictionary or user-managed glossary.
- Reuse approved translation memory.
- Allow manual editing.
- Import/export CSV, JSON, or supported resource formats.

Never present copied source text as a completed translation.

### Phase 9: Validation

Validate:

- JSON/YAML/PHP resource syntax.
- Duplicate keys.
- Missing keys.
- Unused keys.
- Empty values.
- Key naming consistency.
- Interpolation variable consistency.
- Pluralization structure where supported.
- Invalid locale identifiers.
- Incorrect resource references.
- Source references to nonexistent keys.
- Unsupported transformation patterns.

Validation should distinguish errors, warnings, and informational findings.

### Phase 10: Apply and report

1. Revalidate file versions and source ranges.
2. Ask for confirmation.
3. Apply accepted edits.
4. Use VS Code WorkspaceEdit where possible.
5. Avoid partial writes where possible.
6. If a write fails, report exactly which files were changed.
7. Offer undo through VS Code and/or restore from a clearly documented backup mechanism.
8. Produce a report listing changed files, added keys, warnings, skipped findings, and validation results.

### Phase 11: Incremental maintenance

When files change:

1. Mark the previous scan as stale.
2. Debounce file watcher events.
3. Rescan only affected files and dependent resource files.
4. Reconcile changed findings with existing keys.
5. Report newly introduced hard-coded strings.
6. Report missing locale entries.
7. Never rewrite files automatically in the background.

The user must explicitly approve any source-code transformation.

---

## 7. Technical Architecture

### 7.1 High-level architecture

```text
VS Code Extension Host
|
+-- Extension Entry / Activation
|
+-- UI Layer
|   +-- Activity Bar
|   +-- Tree Views
|   +-- Webview Panels
|   +-- Commands
|
+-- Application Layer
|   +-- Workspace Orchestrator
|   +-- Scan Use Cases
|   +-- Setup Use Cases
|   +-- Review / Apply Use Cases
|   +-- Validation Use Cases
|
+-- Core Domain
|   +-- Project Profile
|   +-- Candidate Finding
|   +-- Translation Key
|   +-- Locale Resource
|   +-- Transformation Plan
|   +-- Validation Issue
|
+-- Engine Layer
|   +-- Workspace Scanner
|   +-- Framework Detector
|   +-- Parser Registry
|   +-- Candidate Classifier
|   +-- Key Generator
|   +-- Transformation Engine
|   +-- Translation Resource Engine
|   +-- Validation Engine
|   +-- Incremental Scan Engine
|
+-- Adapter Layer
|   +-- Vue Adapter
|   +-- React Adapter
|   +-- Laravel Adapter
|   +-- HTML Adapter
|   +-- Generic JS/TS Adapter
|
+-- Infrastructure
    +-- VS Code API
    +-- File System
    +-- Configuration
    +-- Cache
    +-- Logging
```

### 7.2 Recommended implementation stack

- TypeScript.
- VS Code Extension API.
- Node.js runtime provided by the extension host.
- `@vscode/test-electron` for extension integration tests.
- Vitest or another compatible test runner for unit tests.
- TypeScript compiler in strict mode.
- ESLint and Prettier.
- `web-tree-sitter`, Babel parser, TypeScript compiler API, or framework-specific parsers as appropriate.
- JSON parser and carefully selected YAML/PHP parsing libraries where needed.

Choose parsers based on licensing, maintenance, runtime compatibility, and syntax coverage. Avoid implementing complex language parsing with regular expressions.

### 7.3 Parser strategy

Use a parser registry:

```ts
interface SourceParser {
  id: string;
  supportedLanguages: string[];
  canParse(filePath: string, content: string): boolean;
  parse(input: ParseInput): Promise<ParseResult>;
}
```

Parser output should contain source ranges and structured syntax context.

Do not let parsers write files. Parsing and transformation must be separate responsibilities.

### 7.4 Framework adapter contract

```ts
interface LocalizationAdapter {
  id: string;
  displayName: string;

  detect(context: ProjectContext): Promise<DetectionResult>;

  scan(context: ProjectContext): Promise<CandidateFinding[]>;

  readResources(context: ProjectContext): Promise<LocaleResource[]>;

  createSetupPlan(
    context: ProjectContext,
    options: LocalizationOptions
  ): Promise<ChangePlan>;

  createTransformation(
    finding: CandidateFinding,
    options: LocalizationOptions
  ): Promise<TransformationResult>;

  createResourcePlan(
    resources: LocaleResource[],
    options: LocalizationOptions
  ): Promise<ChangePlan>;

  validate(context: ProjectContext): Promise<ValidationIssue[]>;
}
```

The contract may evolve, but keep adapter-specific behavior out of the core domain.

---

## 8. Suggested Project Structure

```text
i18n-auto-manager/
|
+-- .vscode/
|   +-- launch.json
|   +-- tasks.json
|
+-- docs/
|   +-- PRODUCT_SPEC.md
|   +-- ARCHITECTURE.md
|   +-- ADAPTER_GUIDE.md
|   +-- SECURITY.md
|   +-- TESTING.md
|
+-- src/
|   +-- extension.ts
|   |
|   +-- commands/
|   |   +-- registerCommands.ts
|   |   +-- scanWorkspace.ts
|   |   +-- setupLocalization.ts
|   |   +-- applyChanges.ts
|   |   +-- validateWorkspace.ts
|   |
|   +-- ui/
|   |   +-- providers/
|   |   +-- views/
|   |   +-- webviews/
|   |   +-- messages/
|   |
|   +-- application/
|   |   +-- orchestrators/
|   |   +-- useCases/
|   |   +-- services/
|   |
|   +-- domain/
|   |   +-- models/
|   |   +-- enums/
|   |   +-- interfaces/
|   |   +-- errors/
|   |
|   +-- engines/
|   |   +-- scanner/
|   |   +-- detection/
|   |   +-- parser/
|   |   +-- classification/
|   |   +-- keys/
|   |   +-- transformation/
|   |   +-- resources/
|   |   +-- validation/
|   |   +-- incremental/
|   |
|   +-- adapters/
|   |   +-- base/
|   |   +-- html/
|   |   +-- javascript/
|   |   +-- typescript/
|   |   +-- vue/
|   |   +-- react/
|   |   +-- laravel/
|   |
|   +-- infrastructure/
|   |   +-- filesystem/
|   |   +-- config/
|   |   +-- cache/
|   |   +-- logging/
|   |
|   +-- shared/
|       +-- types/
|       +-- utils/
|       +-- constants/
|
+-- test/
|   +-- unit/
|   +-- integration/
|   +-- fixtures/
|   |   +-- vue-project/
|   |   +-- react-project/
|   |   +-- laravel-project/
|   |   +-- html-project/
|   +-- e2e/
|
+-- package.json
+-- tsconfig.json
+-- eslint.config.js
+-- .gitignore
+-- README.md
+-- CHANGELOG.md
+-- LICENSE
```

Do not create every file as an empty placeholder on day one. Add modules as their implementation phase begins.

---

## 9. Core Data Models

Use typed models and keep source ranges explicit.

### 9.1 Project profile

```ts
interface ProjectProfile {
  workspaceUri: string;
  projectName: string;
  frameworks: FrameworkDetection[];
  localization: LocalizationDetection[];
  sourceLocale?: string;
  targetLocales: string[];
  resourcePaths: string[];
  ignoredPaths: string[];
  lastScanAt?: number;
}
```

### 9.2 Candidate finding

```ts
type Confidence = "high" | "medium" | "low";

interface SourceRange {
  startLine: number;
  startCharacter: number;
  endLine: number;
  endCharacter: number;
}

interface CandidateFinding {
  id: string;
  fileUri: string;
  range: SourceRange;
  originalText: string;
  syntaxKind: string;
  confidence: Confidence;
  reasonCodes: string[];
  adapterId: string;
  suggestedKey?: string;
  status: "new" | "accepted" | "rejected" | "ignored" | "stale";
  transformSupported: boolean;
  interpolationVariables: string[];
}
```

### 9.3 Translation resource entry

```ts
interface TranslationEntry {
  key: string;
  locale: string;
  value: string;
  sourceLocaleValue?: string;
  status: "translated" | "missing" | "needs-review" | "placeholder";
  variables: string[];
}
```

### 9.4 Change plan

```ts
interface ChangePlan {
  id: string;
  createdAt: number;
  files: PlannedFileChange[];
  warnings: string[];
  requiresConfirmation: true;
}

interface PlannedFileChange {
  fileUri: string;
  operation: "create" | "update";
  proposedContent?: string;
  edits?: PlannedTextEdit[];
  reason: string;
}
```

Avoid storing absolute machine-specific paths in portable project configuration.

---

## 10. Key Generation and Collision Handling

Key generation must be deterministic and configurable.

### 10.1 Recommended default

Prefer semantic keys with a module or feature namespace:

```text
auth.login
auth.loginSuccess
customer.title
customer.create
common.cancel
common.save
```

### 10.2 Rules

- Normalize text only for candidate key generation, not for source replacement.
- Use file/module context to reduce collisions.
- Detect duplicate keys.
- Preserve manually edited keys.
- Never overwrite an existing key with a different meaning.
- Offer a collision resolution screen.
- Support user-defined naming patterns.
- Keep a stable mapping between finding IDs and generated keys where possible.

---

## 11. Translation and Locale Resource Rules

### 11.1 Locale identifiers

Use standard BCP 47 language tags where appropriate, such as:

- `en`
- `en-US`
- `vi`
- `ja`
- `zh-CN`
- `zh-TW`
- `ar`

Validate locale identifiers. Do not assume a language is identical across all regions.

### 11.2 Interpolation

Preserve variables:

```text
Hello, {name}
You have {count} notifications
```

The validator must compare interpolation variables across locales.

### 11.3 Pluralization

Do not treat pluralization as simple string substitution. Support the plural syntax of the selected i18n library. If the adapter cannot safely generate plural messages, report the case for manual handling.

### 11.4 Right-to-left languages

For RTL locales, report that the application may need:
- `dir="rtl"` or equivalent runtime direction handling.
- CSS logical properties.
- Layout and icon review.
- Font coverage checks.

Do not automatically rewrite the entire CSS codebase.

### 11.5 Dates, numbers, and currencies

The extension may provide diagnostics or suggestions for locale-sensitive formatting APIs. It must not blindly replace every date or number formatter.

---

## 12. Safety, Privacy, and Reliability

This is a source-code modification tool. Safety is a core product requirement.

### 12.1 Mandatory safety rules

- Never modify files during a scan.
- Never apply changes without explicit confirmation.
- Never execute project scripts automatically.
- Never install dependencies without explicit confirmation.
- Never send source code over the network in offline mode.
- Never write outside the selected workspace without explicit permission.
- Never transform unsupported syntax.
- Never silently discard existing translations.
- Never overwrite an existing key with a conflicting value.
- Detect stale source ranges before applying edits.
- Provide a clear list of modified files.
- Support cancellation of long-running scans.
- Handle symlinks and path traversal safely.
- Avoid logging source-code contents unless explicitly enabled.

### 12.2 Transaction-like apply process

1. Build a complete change plan.
2. Validate the plan.
3. Verify the original file versions or content hashes.
4. Display the diff.
5. Obtain user confirmation.
6. Apply edits.
7. Verify resulting files.
8. Report partial failures accurately.

Use VS Code WorkspaceEdit for editor-managed changes where appropriate. For multi-file operations, design rollback and failure reporting carefully; do not claim filesystem-wide atomicity unless actually guaranteed.

### 12.3 Privacy

The default product must be local-only. Any future network integration must:
- Be disabled by default.
- Clearly identify what data leaves the machine.
- Require explicit user consent.
- Allow users to configure provider and endpoint.
- Avoid uploading unrelated source files.

---

## 13. Performance Requirements

- Do not block the extension host with large synchronous scans.
- Use asynchronous file operations.
- Support cancellation tokens.
- Debounce file watcher events.
- Cache parser results using file URI, modification metadata, and content hash.
- Rescan only affected files when possible.
- Avoid scanning dependencies and build output by default.
- Show progress for large workspaces.
- Allow users to pause or cancel a scan.
- Provide configurable file-size limits.
- Handle monorepos and multiple workspace roots.

Performance targets should be measured against fixture projects and representative real-world workspaces. Do not promise fixed scan times before benchmarking.

---

## 14. VS Code Commands and Settings

### 14.1 Suggested commands

- `i18nAutoManager.scanWorkspace`
- `i18nAutoManager.setupLocalization`
- `i18nAutoManager.reviewFindings`
- `i18nAutoManager.applyAcceptedChanges`
- `i18nAutoManager.validateWorkspace`
- `i18nAutoManager.openLocaleFiles`
- `i18nAutoManager.rescanChangedFiles`
- `i18nAutoManager.ignoreFinding`
- `i18nAutoManager.exportReport`

### 14.2 Suggested settings

```json
{
  "i18nAutoManager.exclude": [
    "**/node_modules/**",
    "**/vendor/**",
    "**/dist/**",
    "**/build/**",
    "**/coverage/**",
    "**/.git/**"
  ],
  "i18nAutoManager.defaultSourceLocale": "en",
  "i18nAutoManager.keyStyle": "nested-dot",
  "i18nAutoManager.autoWatch": true,
  "i18nAutoManager.scanOnOpen": false,
  "i18nAutoManager.requireReviewBeforeApply": true,
  "i18nAutoManager.maxFileSizeKb": 1024
}
```

Settings must be validated and documented. Workspace-specific settings should override user settings only according to standard VS Code configuration precedence.

---

## 15. Testing Strategy

Testing is mandatory because incorrect transformations can break applications.

### 15.1 Unit tests

Test:
- File filtering.
- Framework detection.
- Parser output.
- Candidate classification.
- Confidence reasons.
- Key generation.
- Collision handling.
- Locale validation.
- Interpolation comparison.
- Resource merging.
- Diff generation.
- Stale finding detection.

### 15.2 Adapter tests

For each adapter, provide fixture projects with:
- Supported simple cases.
- Nested templates.
- Dynamic expressions.
- Escaped strings.
- Comments.
- Attributes.
- Existing translation calls.
- Unsupported patterns.
- Malformed source.
- Duplicate text with different contexts.

### 15.3 Integration tests

Verify:
- Extension activation.
- Commands and views.
- Workspace scanning.
- Review workflow.
- Applying edits.
- Resource creation.
- Undo behavior.
- File watcher behavior.
- Multi-root workspaces.

### 15.4 Regression tests

Every bug discovered in a real project must become a fixture and regression test.

### 15.5 Acceptance criteria

A release candidate must:
- Never modify files during scanning.
- Never apply edits without confirmation.
- Preserve existing locale entries.
- Detect stale findings.
- Correctly handle supported interpolation patterns.
- Pass all adapter fixture tests.
- Produce understandable reports.
- Work without network access.
- Avoid scanning excluded directories by default.

---

## 16. Implementation Roadmap

Implement in small, independently testable increments. Do not ask a coding agent to generate the entire extension in one pass.

### Milestone 0 — Repository foundation

Tasks:
1. Create a VS Code extension project using the official TypeScript extension template.
2. Configure strict TypeScript.
3. Add linting, formatting, and test scripts.
4. Add extension activation and one sample command.
5. Add README and development instructions.
6. Verify that the extension launches in an Extension Development Host.

Done when:
- The extension activates.
- The sample command works.
- Build and tests run successfully.

### Milestone 1 — Workspace scanner

Tasks:
1. Implement workspace-root selection.
2. Implement include/exclude rules.
3. Exclude generated and dependency directories.
4. Enumerate supported source files.
5. Add cancellation and progress reporting.
6. Add scanner unit tests.

Done when:
- The scanner lists eligible files.
- It does not modify any file.
- Large scans can be cancelled.

### Milestone 2 — Project and framework detection

Tasks:
1. Inspect manifests and configuration files.
2. Detect JavaScript/TypeScript projects.
3. Detect Vue, React, and Laravel evidence.
4. Detect existing localization libraries and resource directories.
5. Produce an evidence-based detection report.
6. Handle ambiguous and monorepo projects.

Done when:
- Detection results include evidence and confidence.
- Ambiguous results are shown to the user instead of guessed.

### Milestone 3 — First parser and candidate extraction

Start with HTML and Vue 3 templates.

Tasks:
1. Select and integrate appropriate parsers.
2. Extract static text nodes.
3. Extract selected user-visible attributes.
4. Exclude comments, identifiers, CSS classes, URLs, and obvious technical strings.
5. Capture exact source ranges.
6. Add confidence and reason codes.
7. Build parser fixture tests.

Done when:
- Findings are reviewable.
- No source file is modified.
- Unsupported syntax is reported safely.

### Milestone 4 — Findings UI

Tasks:
1. Add sidebar navigation.
2. Add scan progress and summary.
3. Add findings tree/table.
4. Add filters by file, confidence, adapter, and status.
5. Add ignore/reject controls.
6. Add navigation from a finding to its source location.

Done when:
- A developer can scan and inspect findings inside VS Code.

### Milestone 5 — Localization configuration

Tasks:
1. Add source locale selection.
2. Add target locale selection.
3. Add resource path and format configuration.
4. Detect existing i18n setup.
5. Add Vue I18n setup proposal.
6. Add setup preview and confirmation.

Done when:
- The extension can propose a localization setup without silently changing files.

### Milestone 6 — Key and resource engine

Tasks:
1. Implement key generation.
2. Implement collision detection.
3. Read existing JSON locale resources.
4. Merge new entries without deleting existing values.
5. Generate source-locale resources.
6. Generate target-locale placeholders clearly marked as untranslated.
7. Add resource validation.

Done when:
- Resource files are generated consistently and safely.

### Milestone 7 — Safe source transformation

Tasks:
1. Implement Vue I18n transformations for a limited, tested set of syntax patterns.
2. Preserve interpolation variables.
3. Create a complete change plan.
4. Show source and resource diffs.
5. Verify source ranges and file versions before apply.
6. Apply only accepted changes.
7. Add rollback/error reporting tests.

Done when:
- Supported Vue cases can be converted through an explicit review-and-apply workflow.

### Milestone 8 — Validation and maintenance

Tasks:
1. Detect missing keys.
2. Detect unused keys where static analysis is reliable.
3. Compare interpolation variables.
4. Detect duplicate keys and empty values.
5. Add file watcher with debouncing.
6. Add incremental scanning.
7. Add exportable reports.

Done when:
- Developers can maintain localization as the project changes.

### Milestone 9 — React and Laravel adapters

Implement one adapter at a time.

For each:
1. Detection.
2. Read existing localization configuration.
3. Scan supported syntax.
4. Resource handling.
5. Safe transformations.
6. Validation.
7. Fixture tests.
8. Documentation and known limitations.

Do not start the next adapter until the current one passes its acceptance tests.

### Milestone 10 — Packaging and release

Tasks:
1. Add extension icon and marketplace metadata.
2. Document supported and unsupported cases.
3. Add privacy statement.
4. Add changelog and license.
5. Build `.vsix`.
6. Test installation in a clean VS Code profile.
7. Test on supported operating systems.
8. Publish only after release checks pass.

---

## 17. Vibe-Coding Instructions for the Coding Agent

Use the following rules when asking an AI coding assistant to implement this repository.

### 17.1 General rules

- Read this document before changing code.
- Inspect the current repository before proposing changes.
- Do not assume the repository is empty.
- Do not generate the entire product in one response.
- Work only on the current milestone and explicitly requested task.
- Keep modules small and responsibilities clear.
- Use strict TypeScript and avoid `any`.
- Prefer tested parsers over regular expressions for source-code transformations.
- Keep parsing, classification, planning, and writing separate.
- Never silently modify user source files.
- Never apply source transformations without an explicit user action.
- Never add AI or network dependencies to the offline core.
- Do not invent support for a framework that has not been tested.
- Add tests with every meaningful feature.
- Run relevant tests, type checks, and linting after changes.
- Report changed files, commands run, test outcomes, known limitations, and next steps.
- If requirements are ambiguous, ask a focused question rather than guessing.
- Do not refactor unrelated code.
- Do not leave placeholder implementations while claiming a feature is complete.

### 17.2 Required implementation response format

After each coding task, report:

1. **Implemented:** what is now functional.
2. **Files changed:** exact paths.
3. **Design decisions:** important implementation choices.
4. **Tests:** commands executed and actual results.
5. **Limitations:** unsupported cases or known risks.
6. **Next task:** the smallest logical next increment.

### 17.3 Milestone discipline

Before coding:
1. Read the relevant architecture and acceptance criteria.
2. Inspect current files.
3. Write a short implementation plan.
4. Identify dependencies and risks.

During coding:
1. Implement the smallest coherent unit.
2. Add tests.
3. Run tests.
4. Fix regressions.
5. Update documentation.

After coding:
1. Summarize actual results.
2. Do not claim untested functionality.
3. Wait for approval before moving to a materially different milestone.

---

## 18. Suggested First Prompt for a Vibe-Coding Agent

Copy the prompt below into your coding agent after placing this document in the repository as `docs/PROJECT_BLUEPRINT.md`.

> You are a senior TypeScript and Visual Studio Code extension engineer.
>
> Read `docs/PROJECT_BLUEPRINT.md` completely and treat it as the product and architecture specification.
>
> Your first task is **Milestone 0 only: Repository foundation**.
>
> Inspect the existing repository before making changes. If it is empty, initialize a professional VS Code extension project using TypeScript and the official extension development conventions.
>
> Requirements:
> - Strict TypeScript.
> - Clean extension activation and deactivation.
> - One sample command registered in `package.json`.
> - ESLint and formatting configuration.
> - A working test setup.
> - Development launch configuration for the Extension Development Host.
> - Clear README instructions for install, build, test, and debug.
> - No AI integration.
> - No network dependency.
> - No source-code scanning or transformation yet.
>
> Do not implement future milestones.
>
> Run the available build, type-check, lint, and test commands. Fix failures caused by your changes.
>
> At the end, report exact files changed, commands executed, test results, and any limitations. Do not claim success for a command that was not executed.

---

## 19. Definition of Done for Version 1.0

Version 1.0 should be considered complete only when a developer can:

1. Install the `.vsix` into VS Code.
2. Open a supported Vue 3, React, or Laravel project.
3. Run a workspace scan.
4. See detected framework and localization setup with evidence.
5. Review extracted user-facing text and confidence levels.
6. Configure source and target locales.
7. Generate or update locale resource files without losing existing translations.
8. Preview supported source transformations.
9. Explicitly approve and apply changes.
10. Validate missing keys and interpolation mismatches.
11. Undo or recover from applied changes using the documented mechanism.
12. Continue scanning for new strings without automatic background rewrites.
13. Use all core features without AI, an account, a remote server, or network access.

A smaller, reliable release is better than a broad extension that claims to support every framework but corrupts source code in edge cases.

---

## 20. Product Expansion After Version 1.0

Possible future capabilities:

- Additional framework adapters.
- More localization resource formats.
- Translation memory and glossary management.
- Import/export workflows.
- Visual preview integration with a running application.
- RTL and locale-format diagnostics.
- Team-shared localization conventions.
- Optional offline translation models.
- Optional user-configured translation providers.
- Optional AI assistant for ambiguous context and translation quality.
- Public adapter SDK so the community can add framework support.

Keep these features outside the initial core unless a concrete use case justifies them.

---

## Final Product Direction

Build **i18n Auto Manager** as a safe, local-first developer tool—not as an autonomous code-rewriting bot.

Its strongest value is not that it can translate every sentence. Its value is that it can reliably perform the repetitive engineering work of internationalization: discover strings, understand supported syntax, create stable keys, integrate with an existing localization system, update resource files, validate consistency, and present safe changes for developer approval.

The first release should prioritize correctness, transparency, and reversibility over the number of supported frameworks.
