# i18n Auto Manager — VS Code Extension
## Product Requirements, Technical Architecture, Implementation Workflow, and Vibe-Coding Guide

**Document version:** 1.1 (Harmonized Blueprint)  
**Status:** Implementation blueprint  
**Product type:** Local-first Visual Studio Code extension  
**Companion document:** `I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md` (Normative Contract)  
**Primary objective:** Help developers internationalize existing software projects safely, deterministically, and offline—without requiring AI or cloud services.

---

## 1. Product Vision

Build **i18n Auto Manager**, a VS Code extension that scans an existing codebase, identifies user-facing text using language-aware AST parsers, helps developers convert that text into localization lookups via surgical range edits, generates and merges translation resources, and validates that all configured locales remain consistent.

The extension must be fully functional in **offline mode**. It must not require an AI provider, API key, account, remote server, or network connection for its core workflow.

### 1.1 Core promise

A developer can:

1. Open an existing project in VS Code.
2. Run a workspace localization scan.
3. Review detected user-facing strings, confidence levels, and exact source locations in the Activity Bar.
4. Configure a source locale, target locales, and localization adapter strategy.
5. Preview exact side-by-side diffs of proposed source-code and JSON resource changes using VS Code's native diff viewer.
6. Apply approved changes safely with full Undo and stale-file protection.
7. Validate missing keys, unused keys, interpolation variables, and resource consistency.
8. Continue detecting new strings incrementally as the project evolves.

### 1.2 Product principles

- **Local-first & Offline:** Source code never leaves the developer's machine.
- **Deterministic (No AI dependency):** Core extraction and transformation rely on compilers/ASTs, not probabilistic models.
- **Safe & Formatting-Preserving:** Use ASTs to find exact character ranges, then apply surgical range replacements so surrounding code formatting, comments, and style remain untouched.
- **Reviewable changes:** Every proposed edit is inspectable via native diffs before writing.
- **Idempotent:** Running the tool twice never duplicates keys or imports.
- **Honest detection:** Clearly separate `'high'` confidence static strings from `'medium'`, `'low'`, or `'unknown'` candidates.

---

## 2. Goals, Scope, and Non-Goals

### 2.1 Version 1.0 Goals

- Detect workspace languages, frameworks (Vue 3, React, JS/TS, HTML), and existing i18n libraries (`vue-i18n`, `i18next`, `react-i18next`, `react-intl`).
- Scan supported files while respecting `.gitignore`, default exclusions (`node_modules`, `dist`, `.env*`, etc.), and file-size limits.
- Extract candidate strings with exact 0-based line/character ranges, byte/character offsets, and file SHA-256 hashes.
- Classify findings using the unified taxonomy (`ConfidenceLevel`, `CandidateCategory`, `CandidateStatus`).
- Generate deterministic, collision-safe translation keys.
- Transform approved source patterns into localization calls using range-based edits.
- Generate and update nested or flat JSON locale resource files while preserving existing translations and file indentation.
- Preserve and validate interpolation variables across locales.
- Provide native VS Code TreeViews, QuickPicks, and `vscode.diff` previews.

### 2.2 Explicit Non-Goals for Version 1.0

- Do not claim universal language support (Laravel PHP/Blade and YAML/PO formats are deferred to **Version 1.1**).
- Do not use regular expressions as the primary parser or code rewriter.
- Do not re-serialize entire ASTs back to files (which ruins user formatting).
- Do not execute `npm install` or shell scripts automatically.
- Do not use AI or remote translation APIs in the core engine.

---

## 3. Supported Technology Strategy

Implement a capability-based adapter architecture. Do not claim support for a framework until its AST parser and transformation fixtures pass all quality gates.

### 3.1 Version 1.0 Support Matrix

| Technology | Parser Strategy | Initial Capability | Target i18n Integration |
|---|---|---|---|
| **JSON Resources** | `jsonc-parser` / native JSON + indent detection | Read, validate, merge, write (nested & flat) | Standard JSON locale files |
| **HTML** | `parse5` or `htmlparser2` (with source locations) | Static text & attribute extraction, configurable transform | Custom / template attributes |
| **JavaScript / TypeScript** | `@babel/parser` or TypeScript Compiler API | String literal candidate extraction & `t()` transform | `i18next` / generic `t()` |
| **Vue 3 SFC (`.vue`)** | `@vue/compiler-sfc` + `@babel/parser` | `<template>` text, attributes, interpolations & `<script setup>` | `vue-i18n` (`$t`, `t`, `useI18n`) |
| **React (`.jsx`, `.tsx`)** | `@babel/parser` (with `jsx`, `typescript` plugins) | `JSXText`, `JSXExpressionContainer`, props & hook injection | `react-i18next` (`useTranslation`, `t`) |

### 3.2 Post-1.0 Roadmap (Version 1.1+)

| Technology | Planned Parser Strategy | Target Integration |
|---|---|---|
| **Laravel PHP & Blade** | `php-parser` + dedicated Blade directive tokenizer | Laravel `lang/` PHP/JSON files, `__()`, `@lang` |
| **YAML Locale Files** | `yaml` (CST-preserving parser) | YAML resource files |
| **Angular / Svelte** | Framework-specific template compilers | `@ngx-translate` / `svelte-i18n` |

### 3.3 Adapter Capability Flags

Every adapter must explicitly declare its supported capabilities:

```ts
export interface AdapterCapabilities {
  detect: boolean;
  scan: boolean;
  readResources: boolean;
  transform: boolean;
  writeResources: boolean;
  validate: boolean;
  watch: boolean;
}
```

---

## 4. User Experience Architecture

To keep the extension fast, theme-consistent, and maintainable, prioritize **native VS Code UI APIs** over heavy custom Webviews wherever possible:

1. **Activity Bar Container (`i18n Auto Manager`):**
   - **Overview & Profile View (`TreeDataProvider`):** Shows workspace name, detected frameworks & confidence, active i18n library, source/target locales, last scan timestamp, and quick action buttons (*Scan*, *Configure*, *Validate*, *Apply Approved*).
   - **Candidate Findings View (`TreeDataProvider`):** Hierarchical view (Filterable by Confidence, Category, Status, or File). Each item shows the string, suggested key, confidence badge, and inline action buttons (*Approve*, *Reject*, *Ignore*, *Preview Diff*).
   - **Validation & Locale Resources View (`TreeDataProvider`):** Lists missing keys, unused keys, interpolation mismatches, and JSON errors. Clicking an issue jumps directly to the source or locale file line.
2. **Native Side-by-Side Diff Preview (`TextDocumentContentProvider` + `vscode.diff`):**
   - Register a virtual URI scheme (e.g., `i18n-preview://`).
   - When the user clicks *Preview Change* on a file or candidate, compute the in-memory transformed content and invoke `vscode.commands.executeCommand('vscode.diff', originalFileUri, previewUri, title)`.
3. **Configuration Flow:**
   - Guided multi-step `vscode.window.createQuickPick` / `showInputBox` wizard (or lightweight configuration panel) that writes validated settings to `.vscode/settings.json` or `.i18n-auto-manager.json` upon user confirmation.

---

## 5. End-to-End Engineering Workflow

### Phase 1: Workspace Discovery
1. Enumerate opened `vscode.workspace.workspaceFolders`.
2. Load extension settings and `.gitignore` rules.
3. Exclude `node_modules`, `vendor`, `dist`, `build`, `out`, `.git`, `coverage`, `.env*`, and files larger than `maxFileSizeKb`.
4. Return a `DiscoverySummary` with counts of discovered, included, excluded, and unsupported files.

### Phase 2: Framework & i18n Detection
1. Inspect `package.json`, `tsconfig.json`, `vite.config.*`, and existing locale directories.
2. Evaluate adapter detection rules and compute `ConfidenceLevel` + evidence paths.
3. Inspect existing i18n initialization files and JSON locale structures (detecting flat vs. nested keys, indentation, and interpolation syntax).
4. If ambiguous or conflicting, surface options to the user rather than guessing.

### Phase 3: AST Scanning & Candidate Extraction
1. Read eligible source files asynchronously (supporting `CancellationToken`).
2. Compute the SHA-256 `fileContentHash` of each file's raw text.
3. Parse files into ASTs with character offsets and 0-based line/column coordinates enabled.
4. Traverse AST nodes to extract candidate strings, existing i18n calls, and interpolated expressions.
5. Classify each candidate with a `CandidateCategory`, `ConfidenceLevel`, and `reasonCodes`.

### Phase 4: Key Generation & Collision Resolution
1. For eligible candidates, generate deterministic keys using the configured strategy (`nested-dot`, `flat-dot`, `camelCase`, `snake_case`) and module/file path context.
2. Check existing locale resources for key collisions:
   - If the key exists with the **exact same source text**, offer to reuse the existing key.
   - If the key exists with **different source text**, append a disambiguating context suffix or flag a collision for review.

### Phase 5: Range-Based Transformation Planning
1. For each `'approved'` candidate where `transformSupported === true`:
   - Verify the adapter can safely transform the syntax node.
   - Create a surgical `PlannedTextEdit` using the exact `SourceRange` offsets.
   - If the file requires an i18n import or hook declaration (e.g., `import { useI18n } from 'vue-i18n'` or `const { t } = useI18n()`), compute a single idempotent insertion edit at the appropriate AST scope.
2. Compute the updated JSON locale resource content in memory, preserving existing keys and file indentation.
3. Bundle all source edits and resource updates into a `ChangePlan`.

### Phase 6: Diff Preview, Revalidation & Safe Apply
1. Present the `ChangePlan` to the user via the Findings View and native `vscode.diff` preview.
2. When the user clicks **Apply Approved Changes**:
   - Re-read all target files and verify their current SHA-256 hash matches `expectedFileHash` from the scan.
   - If any file changed, mark its edits as `'stale'`, block the apply for that file, and alert the user.
   - Validate that generated JSON resources parse cleanly without syntax errors.
   - Construct and apply a `vscode.WorkspaceEdit`.
   - Record an `OperationReport` detailing applied edits, added keys, skipped items, and any warnings.

### Phase 7: Validation & Incremental Watch
1. Run static validation across source references and JSON locale files (missing keys, unused keys, variable mismatches, duplicate keys).
2. Publish diagnostics to VS Code's `DiagnosticCollection` (Problems panel) and Validation TreeView.
3. Watch workspace files with a debounced watcher (`400ms`). When a file changes, mark its cached findings as `'stale'` and offer a fast incremental rescan.

---

## 6. Technical Architecture & Stack

### 6.1 Architectural Layers

```text
VS Code Extension Host
|
+-- Extension Entry (src/extension.ts)
|
+-- UI Layer (src/ui/)
|   +-- TreeView Providers (Overview, Findings, Validation)
|   +-- Virtual Diff Provider (TextDocumentContentProvider)
|   +-- QuickPick Setup Wizard & Commands
|   +-- DiagnosticCollection Manager
|
+-- Application Layer (src/application/)
|   +-- WorkspaceOrchestrator
|   +-- ScanWorkspaceUseCase
|   +-- PlanTransformationsUseCase
|   +-- ApplyChangePlanUseCase
|   +-- ValidateWorkspaceUseCase
|
+-- Core Domain Layer (src/domain/)  [Zero VS Code API dependencies]
|   +-- Canonical Enums & Types
|   +-- ProjectProfile, CandidateFinding, TranslationEntry
|   +-- ChangePlan, PlannedFileChange, PlannedTextEdit
|   +-- ValidationIssue, OperationReport
|
+-- Engine Layer (src/engines/)
|   +-- WorkspaceScanner (file filtering, hashing)
|   +-- FrameworkDetector
|   +-- CandidateClassifier
|   +-- KeyGenerator
|   +-- RangeTransformationEngine (offset validation, non-overlapping edits)
|   +-- JsonResourceEngine (indentation-preserving read/merge/serialize)
|   +-- ValidationEngine
|
+-- Adapter Layer (src/adapters/)
|   +-- HtmlAdapter
|   +-- JsTsAdapter
|   +-- Vue3Adapter
|   +-- ReactAdapter
|
+-- Infrastructure Layer (src/infrastructure/)
    +-- VsCodeFileSystemAdapter
    +-- WorkspaceConfigRepository
    +-- ScanCache
    +-- Logger (secret-safe OutputChannel)
```

### 6.2 Recommended Implementation Stack

- **Language:** TypeScript 5.x (`"strict": true`, `"noImplicitAny": true`, `"exactOptionalPropertyTypes": true`).
- **Bundler:** `esbuild` (fast builds, bundles parser dependencies into a single extension output).
- **Parsers:**
  - `@babel/parser` (with `typescript` and `jsx` plugins) for JS/TS/JSX/TSX ASTs with accurate `loc` and `range` offsets.
  - `@vue/compiler-sfc` for parsing Vue 3 `<template>` ASTs and `<script>` blocks.
  - `parse5` (with `sourceCodeLocationInfo: true`) for standalone HTML files.
  - `jsonc-parser` for fault-tolerant, formatting-aware JSON locale file reading and modification.
- **Testing:**
  - `vitest` for fast unit and AST fixture tests (testing domain, engines, and adapters outside VS Code).
  - `@vscode/test-electron` (`@vscode/test-cli`) for VS Code Extension Host integration tests.

---

## 7. Unified Core Data Models

All files in `src/domain/` MUST adhere to these exact interfaces (harmonized with `I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md`):

```ts
export type ConfidenceLevel = 'high' | 'medium' | 'low' | 'unknown';

export type CandidateCategory =
  | 'static-ui-text'
  | 'dynamic-ui-text'
  | 'technical-string'
  | 'existing-i18n-call'
  | 'uncertain';

export type CandidateStatus =
  | 'pending'
  | 'approved'
  | 'rejected'
  | 'ignored'
  | 'needs-manual-handling'
  | 'stale'
  | 'applied';

export type ValidationSeverity = 'error' | 'warning' | 'info' | 'not-checked';

export type OperationStatus =
  | 'completed-successfully'
  | 'completed-with-warnings'
  | 'partially-completed'
  | 'cancelled'
  | 'failed';

export interface SourceRange {
  startLine: number;      // 0-based line index
  startCharacter: number; // 0-based column index
  endLine: number;        // 0-based line index
  endCharacter: number;   // 0-based column index
  startOffset: number;    // 0-based character offset in file
  endOffset: number;      // 0-based character offset in file
}

export interface FrameworkDetection {
  adapterId: string;
  frameworkName: string;
  confidence: ConfidenceLevel;
  evidenceFiles: string[];
  detectedI18nLibraries: string[];
}

export interface ProjectProfile {
  workspaceUri: string;
  projectName: string;
  detections: FrameworkDetection[];
  selectedAdapterId?: string;
  sourceLocale: string;
  targetLocales: string[];
  resourcePaths: string[];
  ignoredPatterns: string[];
  lastScanAt?: number;
}

export interface CandidateFinding {
  id: string;
  fileUri: string;
  fileContentHash: string; // SHA-256 of file content at scan time
  range: SourceRange;
  originalText: string;
  normalizedText: string;
  syntaxContext: string;   // e.g., 'vue-template-text', 'jsx-attribute', 'ts-call-arg'
  category: CandidateCategory;
  confidence: ConfidenceLevel;
  reasonCodes: string[];
  adapterId: string;
  suggestedKey?: string;
  status: CandidateStatus;
  transformSupported: boolean;
  interpolationVariables: string[];
}

export interface TranslationEntry {
  key: string;
  locale: string;
  value: string;
  sourceLocaleValue?: string;
  status: 'translated' | 'missing' | 'needs-review' | 'placeholder';
  variables: string[];
}

export interface PlannedTextEdit {
  range: SourceRange;
  originalText: string;
  replacementText: string;
  associatedKey?: string;
  description: string;
}

export interface PlannedFileChange {
  fileUri: string;
  expectedFileHash?: string; // Required when operation === 'update'
  operation: 'create' | 'update';
  edits: PlannedTextEdit[];
  previewContent: string;    // Full computed content for vscode.diff preview
  reason: string;
}

export interface ChangePlan {
  id: string;
  createdAt: number;
  adapterId: string;
  ruleVersion: string;
  files: PlannedFileChange[];
  resourceEntriesAdded: TranslationEntry[];
  warnings: string[];
  requiresConfirmation: true;
}

export interface ValidationIssue {
  id: string;
  severity: ValidationSeverity;
  code: string;
  message: string;
  fileUri?: string;
  range?: SourceRange;
  key?: string;
  locale?: string;
  suggestedAction?: string;
}
```

---

## 8. Suggested Project Structure

```text
i18n-auto-manager/
+-- .vscode/
|   +-- launch.json
|   +-- tasks.json
+-- docs/
|   +-- I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md
|   +-- PROJECT_BLUEPRINT_I18N_AUTO_MANAGER.md
+-- src/
|   +-- extension.ts
|   +-- domain/
|   |   +-- models.ts
|   |   +-- enums.ts
|   |   +-- interfaces.ts
|   +-- engines/
|   |   +-- scanner/
|   |   +-- detection/
|   |   +-- classification/
|   |   +-- keys/
|   |   +-- transformation/
|   |   +-- resources/
|   |   +-- validation/
|   +-- adapters/
|   |   +-- base/
|   |   +-- json/
|   |   +-- html/
|   |   +-- jsts/
|   |   +-- vue3/
|   |   +-- react/
|   +-- application/
|   |   +-- orchestrator.ts
|   |   +-- useCases/
|   +-- ui/
|   |   +-- views/
|   |   +-- diff/
|   |   +-- commands/
|   +-- infrastructure/
|   |   +-- filesystem.ts
|   |   +-- config.ts
|   |   +-- logger.ts
|   +-- shared/
|       +-- hash.ts
|       +-- bcp47.ts
+-- test/
|   +-- unit/
|   +-- fixtures/
|   |   +-- vue3-app/
|   |   +-- react-app/
|   |   +-- html-app/
|   +-- integration/
+-- esbuild.js
+-- package.json
+-- tsconfig.json
+-- vitest.config.ts
+-- eslint.config.mjs
+-- README.md
```

---

## 9. Implementation Roadmap (Milestones 0 to 10)

Implement in small, independently verifiable milestones. **Never ask or allow the coding agent to implement multiple milestones in one pass.**

### Milestone 0 — Repository Foundation & Build Pipeline
1. Initialize VS Code extension project with TypeScript (`"strict": true`) and `esbuild`.
2. Configure `vitest` for unit testing and ESLint for strict type-checked linting.
3. Add `src/domain/models.ts` containing the unified canonical types.
4. Register a basic activation entry (`src/extension.ts`) and a health-check command (`i18nAutoManager.showStatus`).
5. **Done when:** `npm run compile`, `npm run lint`, and `npm run test` execute and pass with zero warnings/errors.

### Milestone 1 — Workspace Scanner & File Hashing
1. Implement `WorkspaceScanner` with `.gitignore` and default exclusion patterns (`node_modules`, `.env*`, binaries, >1024KB files).
2. Implement SHA-256 `fileContentHash` utility (`src/shared/hash.ts`).
3. Support `CancellationToken` and progress reporting.
4. Return `DiscoverySummary` (counts of discovered, included, excluded, unsupported files).
5. **Done when:** Unit tests verify accurate file filtering, secret file exclusion, cancellation, and SHA-256 hashing on fixture directories.

### Milestone 2 — Project, Framework & Existing i18n Detection
1. Implement `FrameworkDetector` inspecting `package.json`, config files, and directory structures for Vue 3, React, JS/TS, and HTML.
2. Detect existing `vue-i18n`, `i18next`, `react-i18next`, and existing JSON locale directories.
3. Output `FrameworkDetection[]` with `ConfidenceLevel` and evidence paths.
4. Handle monorepos and ambiguous configurations without modifying files.
5. **Done when:** Unit tests against Vue, React, mixed, and unknown fixture manifests return accurate evidence and confidence levels.

### Milestone 3 — HTML & Vue 3 AST Extraction + Classification
1. Integrate `parse5` (for HTML) and `@vue/compiler-sfc` + `@babel/parser` (for Vue 3 `.vue` files).
2. Extract static text nodes, translatable attributes, simple `{{ expr }}` interpolations, and existing `$t()` / `t()` calls.
3. Record exact 0-based `SourceRange` (lines, columns, and character offsets) and classify with `CandidateCategory`, `ConfidenceLevel`, and `reasonCodes`.
4. Exclude comments, URLs, CSS classes, and technical identifiers.
5. **Done when:** Fixture tests prove accurate extraction, offset accuracy, and zero false positives on technical attributes.

### Milestone 4 — Activity Bar UI & Candidate Findings TreeView
1. Register Activity Bar container and TreeViews (*Overview*, *Candidate Findings*, *Validation*).
2. Wire `i18nAutoManager.scanWorkspace` to populate the TreeViews.
3. Add click-to-reveal source range in the active editor.
4. Add commands to Approve, Reject, Ignore, and Filter findings by confidence/category/status.
5. **Done when:** Developer can run a scan in the Extension Development Host, browse findings, jump to lines, and toggle candidate states.

### Milestone 5 — Configuration, BCP 47 Validation & JSON Resource Engine
1. Implement BCP 47 locale tag validator (`src/shared/bcp47.ts`).
2. Add QuickPick configuration wizard for source locale, target locales, key style, and resource directory.
3. Implement `JsonResourceEngine` (using `jsonc-parser`) to read, merge, and serialize nested and flat JSON files while preserving existing keys, translations, and indentation.
4. Implement deterministic `KeyGenerator` with collision detection.
5. **Done when:** Unit tests verify collision-safe key generation and non-destructive JSON merging with preserved formatting.

### Milestone 6 — Range-Based Transformation Engine & Native Diff Preview
1. Implement `RangeTransformationEngine` that takes approved `CandidateFinding` items, verifies non-overlapping offsets, and generates surgical `PlannedTextEdit` items (sorting edits bottom-to-top so offsets never shift).
2. Implement Vue 3 transformations (`{{ $t('key') }}`, `:placeholder="$t('key')"`, and script `t('key')` with idempotent `useI18n` import/hook insertion).
3. Implement `PreviewDiffProvider` (`TextDocumentContentProvider`) and wire `vscode.diff` so users can inspect exact source and JSON diffs before applying.
4. **Done when:** Fixture tests prove Vue 3 files are transformed accurately with 100% of surrounding formatting and comments preserved.

### Milestone 7 — Safe Apply Transaction & Stale-Edit Protection
1. Implement `ApplyChangePlanUseCase`.
2. Re-verify SHA-256 `expectedFileHash` and exact substring match at every `SourceRange` immediately before applying. Block and mark `'stale'` if modified.
3. Apply source and JSON resource changes via `vscode.WorkspaceEdit`.
4. Generate structured `OperationReport` and verify idempotency on immediate re-scan.
5. **Done when:** Integration/unit tests verify successful apply, stale-file blocking, and idempotency.

### Milestone 8 — Validation Engine & Incremental File Watcher
1. Implement `ValidationEngine` checking missing keys, unused keys, invalid JSON, duplicate keys, and interpolation variable mismatches across locales.
2. Publish issues to VS Code `DiagnosticCollection` and Validation TreeView.
3. Implement debounced `FileSystemWatcher` that marks modified files' findings as `'stale'` and offers incremental rescan.
4. **Done when:** Unit tests verify all validation rules and stale-marking behavior on file change.

### Milestone 9 — React JSX/TSX & Generic JS/TS Adapters
1. Implement `ReactAdapter` and `JsTsAdapter` using `@babel/parser` (`jsx`, `typescript` plugins).
2. Support `JSXText`, `JSXExpressionContainer` with interpolation, translatable JSX props (`placeholder`, `title`, `aria-label`, `alt`), and idempotent `useTranslation` hook / `import` insertion.
3. Pass all adapter quality gate fixture tests (static, dynamic, comments, existing `t()` calls, unsupported complex JSX).
4. **Done when:** React and JS/TS fixture suites pass extraction, range transformation, and idempotency tests.

### Milestone 10 — Packaging, Documentation & Release Readiness (v1.0)
1. Audit all 17 Acceptance Criteria from `I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md`.
2. Complete `README.md`, `CHANGELOG.md`, privacy statement, and adapter support documentation.
3. Package extension into a `.vsix` using `@vscode/vsce` and verify clean installation.
4. *(Post-1.0 Note: Laravel PHP/Blade adapter begins in Milestone 11 for Version 1.1).*

---

## 10. Suggested First Prompt for Your Vibe-Coding Agent

Copy and paste the prompt below into your coding agent (Cursor, Claude Code, Windsurf, or Copilot) after placing both `.md` files in your repository's `docs/` directory:

```text
You are a principal TypeScript and Visual Studio Code extension engineer.

First, read both specification documents completely:
1. `docs/I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md` (Normative Contract)
2. `docs/PROJECT_BLUEPRINT_I18N_AUTO_MANAGER.md` (Architecture & Roadmap)

Your task right now is **Milestone 0 ONLY: Repository Foundation & Build Pipeline**.

Inspect the existing repository first. Initialize a clean VS Code extension project using TypeScript and `esbuild`.

Requirements for Milestone 0:
1. Strict TypeScript configuration (`"strict": true`, `"noImplicitAny": true`).
2. `esbuild` build script for bundling the extension (`npm run compile` and `npm run typecheck`).
3. ESLint configuration (`npm run lint`) and Vitest setup (`npm run test`).
4. Implement the canonical domain types and enums in `src/domain/models.ts` exactly as specified in Section 7 of the Project Blueprint.
5. Implement a minimal extension entry point (`src/extension.ts`) that activates cleanly, initializes a secret-safe OutputChannel logger, and registers a status command (`i18nAutoManager.showStatus`).
6. Add a unit test in `test/unit/domain.test.ts` verifying the domain model helpers/types and run Vitest.
7. Include `.vscode/launch.json` and `.vscode/tasks.json` for running the Extension Development Host.
8. Do NOT implement Milestone 1 or any scanning/parsing logic yet.
9. Run `npm run typecheck`, `npm run lint`, `npm run compile`, and `npm run test` in the terminal and observe the actual outputs.

At the end, provide the mandatory Milestone Completion Report format defined in Section 13 of `I18N_AUTO_MANAGER_REQUIREMENTS_AND_CONSTRAINTS.md`.
```