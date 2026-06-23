# AGENTS.md — Coras Framework Migration Agent
## Prompt Stack Design Document (PSD) · Version 2.0

```
╔══════════════════════════════════════════════════════════════════╗
║           CORAS FRAMEWORK MIGRATION AGENT — PSD v2.0            ║
║    AI-powered migration of @frmk2sjs/* and coras-for-* packages  ║
║          across 75 Angular applications, every release           ║
╚══════════════════════════════════════════════════════════════════╝
```

> **Authored by:** Meenakshi Sundaram  
> **Document version:** 2.0  
> **Agent repository:** `coras-migration-agent/`  
> **Classification:** Internal — Engineering & Architecture  
> **Audience:** Application Engineers · Tech Leads · DevOps · Senior Management  

---

## Table of Contents

1. [Purpose](#1-purpose)  
2. [Context](#2-context)  
3. [Scope](#3-scope)  
4. [Limitations](#4-limitations)  
5. [Inputs](#5-inputs)  
6. [Expected Output](#6-expected-output)  
7. [File Paths — Full Repository Layout](#7-file-paths--full-repository-layout)  
8. [Extra Dynamic Rules — Complete Schema](#8-extra-dynamic-rules--complete-schema)  
9. [How to Use with Roo Code](#9-how-to-use-with-roo-code)  
10. [How to Use with OpenCode](#10-how-to-use-with-opencode)  
11. [CLI Reference & End-to-End Runbook](#11-cli-reference--end-to-end-runbook)  
12. [Rule Authoring Guide — Every Release](#12-rule-authoring-guide--every-release)  
13. [Management Justification](#13-management-justification)  
14. [Governance & Contacts](#14-governance--contacts)  

---

## 1. Purpose

### 1.1 One-Line Summary

The Coras Migration Agent applies all Coras Framework version changes — package updates, theming attributes, component API renames, and configuration file edits — to any Angular application automatically, using AI to handle code transformations that scripts cannot.

### 1.2 Problem It Solves

When the Coras team publishes a new version (e.g., `8.0.6 → 9.0.1`), every one of the **75 Angular applications** in the organisation must:

- Bump `@frmk2sjs/services`, `@frmk2sjs/core`, `@frmk2sjs/component`  
- Bump whichever `coras-for-*` adapter packages they use (conditionally — not all apps use all adapters)  
- Apply any new mandatory HTML attributes (`coras-theme` on `<body>` in 9.0.1)  
- Apply component API renames (e.g., `[popupItems]` → `[menuItems]`)  
- Apply new required bindings (e.g., `[showColumnMenu]`, `[retainSelection]`)  
- Update config files where necessary  

Doing this manually is 3–5 engineer-days per application. Multiplied by 75 applications, this is **225–375 engineer-days per release cycle** — a non-starter at scale.

This agent automates every step above. A human provides the rules file (authored once per release from the CHANGELOG and version announcement), points the agent at the target application, and reviews the dry-run report. The agent does the rest.

### 1.3 What the Agent Does — Layer by Layer

| Layer | What the Agent Automates |
|---|---|
| **Package versions** | Updates `@frmk2sjs/*` in `package.json` to the declared target version |
| **Conditional dependencies** | Updates `coras-for-*` adapters **only if they already exist** in the target app — no blind installs |
| **`index.html`** | Injects or validates theme attributes on `<body>` (e.g., `coras-theme="light"`) |
| **HTML templates** | Applies selector renames, new required bindings, deprecated attribute removal |
| **TypeScript source** | Applies import path changes, service API renames, new config properties |
| **`angular.json`** | Updates builder paths and architect targets via LLM-assisted merge |
| **`tsconfig.json`** | Merges `compilerOptions` changes deterministically |
| **Reports** | Produces JSON report + human-readable log + JIRA-ready summary after every run |
| **Backup** | Full project backup before any `apply` run — rollback is always possible |

### 1.4 What the Agent Does NOT Do

- Does **not** replace `ng update` for Angular core version bumps — run that first  
- Does **not** push to Git — a human reviews the diff and commits  
- Does **not** make any product or business logic decisions  
- Does **not** touch unit tests unless a specific test rule is declared  
- Does **not** apply any change not declared in the rules file  

---

## 2. Context

### 2.1 The Coras Framework Ecosystem

Coras is an internal UI framework maintained by our team. It wraps Angular Material, AG Grid, PrimeNG, and Highcharts with organisation-specific conventions, theming, and component APIs.

**Core packages (always present in Coras apps):**

```
@frmk2sjs/services      HTTP interceptors, authentication, application state services
@frmk2sjs/core          Base classes, lifecycle hooks, decorators
@frmk2sjs/component     Full component library: grids, forms, tree, popup menu, charts
```

**Adapter packages (present only in apps that need them):**

```
coras-for-angular-material   Angular Material styling adapter
coras-for-ag-grid-community  AG Grid Community edition wrapper
coras-for-ag-grid-enterprise AG Grid Enterprise edition wrapper
coras-for-primeNG            PrimeNG component adapter
coras-for-highcharts         Highcharts charting wrapper
```

**Support matrix:**

| Coras Version | Angular | AG Grid | Status |
|---|---|---|---|
| `8.x` | Angular 18 | AG Grid 18 | Supported |
| `9.x` | Angular 19 | AG Grid 18 or 19 | **Current / Recommended** |

### 2.2 Release Cadence

Coras ships approximately every 6–8 weeks. Every release includes two artefacts that feed this agent:

1. **Version announcement** — posted on the internal channel. Contains the new version number, summary of new features, and a link to the CHANGELOG.  
2. **CHANGELOG.md** — hosted in GitLab. The line-by-line source of truth for what changed, keyed to JIRA ticket IDs (TCAAS1443-xxx format).

Both were visible in the version 9.0.1 release. The announcement listed the packages, the required `coras-theme` body attribute, and compatibility notes. The CHANGELOG listed specific JIRA tickets (645, 640, 635, 622, 620) with one-line descriptions. Together they are everything needed to author a complete rules file.

### 2.3 Where This Agent Fits in the Larger Toolchain

```
[Angular core release]
        ↓
  ng update @angular/core @angular/cli   ← Angular's own tooling handles Angular layer
        ↓
  [This agent]                           ← We handle the Coras layer
        ↓
  npm install
        ↓
  ng build / tsc --noEmit               ← CI is the final acceptance gate
        ↓
  PR review → merge → deploy
```

### 2.4 Technology Stack

| Component | Technology | Why |
|---|---|---|
| Agent runtime | Node.js (≥ 18) | Same environment as Angular CLI; no extra install |
| LLM calls | Axios HTTP — provider-agnostic | Works with Anthropic, OpenAI, Azure, Ollama |
| File processing | Node `fs` + regex pre-filter | Fast pattern match before any LLM call |
| Rules | JSON | Human-readable, diff-able, versionable in Git |
| Reports | JSON + plain text log | Machine-readable for CI; human-readable for engineers |
| IDE integration | Roo Code + OpenCode | Two supported workflows — see Sections 9 and 10 |

---

## 3. Scope

### 3.1 In Scope

- All 75 Angular applications consuming `@frmk2sjs/*` or `coras-for-*` packages
- Migrations between **adjacent Coras major versions** within the support matrix
- File types the agent touches: `.ts`, `.html`, `package.json`, `angular.json`, `tsconfig.json`, `tsconfig.app.json`, `src/index.html`
- **Dry-run mode** (default): report produced, no files written
- **Apply mode**: files written after human approval

### 3.2 Out of Scope

| What | Why |
|---|---|
| Angular core runtime bumps | Handled by `ng update` |
| Applications not consuming Coras packages | Agent has nothing to do |
| Multi-hop migrations (e.g., v7 → v9) | Run step-by-step: v7→v8 then v8→v9 |
| Coras Framework source code itself | This agent targets consumers, not the framework |
| Backend / API layer | Out of jurisdiction |
| Git operations | Human reviews and commits |

---

## 4. Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| Rules are only as good as their author | Unlisted changes won't be applied | Rule authoring checklist in Section 12; peer review rules file before batch rollout |
| LLM context window (~128k tokens) | Very large files may be truncated or processed incorrectly | Agent flags files >1800 lines in the report; chunk large files manually if needed |
| No live compilation check during migration | Agent can't guarantee TypeScript compiles until build step | Build agent step runs `tsc --noEmit`; CI is the authoritative gate |
| Conditional dependency detection is `package.json`-based | Transitive-only Coras deps (rare) won't be detected | Run `npm ls @frmk2sjs/component` first to surface hidden dependencies |
| Theming conflict: existing `coras-theme` value | Agent will not overwrite an existing value | Warns in report; human resolves |
| Structural HTML/CSS changes | Layout changes may break app-specific CSS | Structural rules ship with `enabled: false` by default; human applies after visual QA |
| LLM non-determinism | Same file could produce slightly different output on two runs | Dry-run mode + Git diff review catch any surprises before apply |
| Parallel LLM calls (default: 5 files at once) | Rate limiting on some providers | Reduce `parallel_file_processing` in `llm-config.json` if you hit 429s |

---

## 5. Inputs

### 5.1 Primary User — Who Runs This Agent

**Role:** Angular Application Engineer or DevOps Engineer  
**When:** Once per Coras version release (approximately every 6–8 weeks)  
**Technical prerequisites:** Node.js ≥ 18, access to an LLM API key (Anthropic or OpenAI), the target application's source on disk  

**What the user does before running the agent:**
1. Reads the version announcement and CHANGELOG for the new Coras version
2. Authors (or receives from the Coras team) the `coras-migration-rules.json` for this release
3. Sets `project_root` in `llm-config.json` to point at the application being migrated
4. Confirms `dry_run: true` (it is the default — never changes for first run)
5. Runs the agent

### 5.2 Required Input Files

| File | Location in repo | Who authors it | Description |
|---|---|---|---|
| `llm-config.json` | `config/llm-config.json` | Engineer (once, then update `project_root` per app) | LLM credentials, project path, Node.js download URL, migration settings |
| `coras-migration-rules.json` | `rules/coras-migration-rules.json` | Coras team (once per release) | Version numbers, package updates, all transformation rules |
| Target Angular application | Anywhere on disk, pointed to by `project_root` | Existing app | The Angular project being migrated |

### 5.3 `llm-config.json` — Full Reference

```jsonc
{
  // ─── LLM Provider Configuration ───────────────────────────────────────────
  "llm": {
    "provider": "anthropic",
    // Options: "anthropic" | "openai" | "azure-openai" | "ollama"
    // For Roo Code: set to "anthropic" and leave api_key blank — Roo Code injects it
    // For OpenCode: set to "openai" with OpenCode's base_url

    "base_url": "https://api.anthropic.com/v1",
    // anthropic  → https://api.anthropic.com/v1
    // openai     → https://api.openai.com/v1
    // azure      → https://YOUR-RESOURCE.openai.azure.com/openai/deployments/YOUR-DEPLOYMENT
    // ollama     → http://localhost:11434/v1

    "api_key": "YOUR_API_KEY_HERE",
    // For Roo Code usage: leave as placeholder — the agent reads from
    // environment variable ANTHROPIC_API_KEY or OPENAI_API_KEY automatically
    // For OpenCode usage: leave as placeholder — opencode injects credentials

    "model": "claude-sonnet-4-6",
    // Recommended:
    //   Anthropic → claude-sonnet-4-6
    //   OpenAI    → gpt-4o
    //   Azure     → your deployment name

    "max_tokens": 8000,
    "temperature": 0.1,   // Low temperature = deterministic, safe for code
    "timeout_ms": 120000, // 2 minutes per file — allow for large files
    "retry_attempts": 3,
    "retry_delay_ms": 2000
  },

  // ─── Migration Settings ────────────────────────────────────────────────────
  "migration": {
    "from_version": "8",         // Coras major version you are migrating FROM
    "to_version": "9",           // Coras major version you are migrating TO
    "project_root": "../my-angular-app",
    // Absolute or relative path to the Angular project root
    // Must contain package.json and angular.json

    "dry_run": true,
    // ALWAYS start with true. Review the report. Set false only to apply.

    "backup_before_migration": true,
    // Creates timestamped backup under backup_dir before any writes

    "backup_dir": "./migration-backups",
    "parallel_file_processing": 5,
    // Number of files processed concurrently via LLM. Reduce to 2-3 if hitting rate limits.

    "skip_dirs": ["node_modules", ".git", "dist", ".angular"],
    "skip_files": []
    // Add specific filenames to skip (e.g. generated files)
  },

  // ─── Node.js Download (for Build Agent) ───────────────────────────────────
  "node": {
    "required_version": "18.19.0",
    "download_url": "https://nodejs.org/dist/v18.19.0/node-v18.19.0-win-x64.zip",
    // Find your platform's URL at https://nodejs.org/dist/
    // win  → node-vX.X.X-win-x64.zip
    // mac  → node-vX.X.X-darwin-arm64.tar.gz  (set platform: "mac")
    // linux → node-vX.X.X-linux-x64.tar.gz    (set platform: "linux")

    "install_dir": "./tools/node",
    "platform": "win"
    // Options: "win" | "linux" | "mac"
  },

  // ─── Post-Migration Commands ───────────────────────────────────────────────
  "post_migration": {
    "run_npm_install": true,
    "run_command": "ng serve",
    // Options: "ng serve" | "npm run build" | "npm run start" | "npm test"
    "open_browser": false
  }
}
```

### 5.4 CLI Flags (Override config file at runtime)

```bash
node index.js                          # Full pipeline: migrate → npm install → ng serve
node index.js --migrate-only           # Migration agent only (no build step)
node index.js --build-only             # Build agent only (skip migration)
node index.js --config ./path/to.json  # Use a different config file
node index.js --rules ./path/to.json   # Use a different rules file

# npm script shortcuts (defined in package.json)
npm run migrate                        # migration only
npm run build                          # build only
npm run dry-run                        # migration only (dry_run must be true in config)
npm start                              # full pipeline
```

---

## 6. Expected Output

### 6.1 What You See in the Console

```
╔══════════════════════════════════════════════════════╗
║        Angular AI Migration Agent  v1.0              ║
╚══════════════════════════════════════════════════════╝

  Config : ./config/llm-config.json
  Rules  : ./rules/coras-migration-rules.json
  Mode   : migrate

============================================================
  Angular Migration Agent Starting
============================================================
[INFO]  Project: /path/to/my-angular-app
[INFO]  Migration: Coras 8.0.6 → 9.0.1
[INFO]  Dry run: true

============================================================
  Updating package.json
============================================================
[INFO]    @frmk2sjs/services: 8.0.6 → 9.0.1
[INFO]    @frmk2sjs/core: 8.0.6 → 9.0.1
[INFO]    @frmk2sjs/component: 8.0.6 → 9.0.1
[INFO]    coras-for-ag-grid-community: 8.0.6 → 9.0.1 (conditional — found in project)
[INFO]    coras-for-primeNG: not present in project, skipping (conditional)
[OK]    package.json updated

============================================================
  Updating index.html
============================================================
[INFO]  Applying CORAS-THEME-001: coras-theme="light" on <body>
[OK]    src/index.html updated

============================================================
  Migrating Source Files (.ts and .html)
============================================================
[INFO]  Found 89 TypeScript files, 54 HTML files
[INFO]  Processing src/app/grid/grid.component.html (CORAS-HTML-001, CORAS-HTML-002)
[DRY RUN] Would write: src/app/grid/grid.component.html
[INFO]  Processing src/app/menu/menu.component.html (CORAS-HTML-003)
[DRY RUN] Would write: src/app/menu/menu.component.html
...

┌─────────────────────────────────────┐
│         Migration Summary           │
├─────────────────────────────────────┤
│ Files processed : 143               │
│ Files modified  : 38                │
│ Files skipped   : 105               │
│ Errors          : 0                 │
└─────────────────────────────────────┘

ℹ  DRY RUN complete. No files were changed.
   Set "dry_run": false in llm-config.json to apply changes.
```

### 6.2 Generated Artefacts

| Artefact | Location | Created by |
|---|---|---|
| Migration log (plain text) | `logs/migration-TIMESTAMP.log` | Every run |
| Migration report (JSON) | `reports/migration-report-TIMESTAMP.json` | Every run |
| Project backup | `migration-backups/backup-TIMESTAMP/` | Apply-mode runs only |
| JIRA summary (optional) | Copy from report, paste to JIRA ticket | Engineer |

### 6.3 Migration Report — Full JSON Schema

```json
{
  "startTime": "2026-06-19T09:00:00.000Z",
  "endTime":   "2026-06-19T09:08:44.123Z",
  "filesProcessed": 143,
  "filesModified":   38,
  "filesSkipped":   105,
  "errors": [],
  "changes": [
    {
      "file": "src/app/grid/grid.component.html",
      "rules": ["CORAS-HTML-001", "CORAS-HTML-002"],
      "summary": "Applied 2 rule(s)"
    },
    {
      "file": "src/app/menu/menu.component.html",
      "rules": ["CORAS-HTML-003"],
      "summary": "Applied 1 rule(s)"
    }
  ],
  "warnings": [
    "CORAS-HTML-005: coras-tree detected in 3 files — manual review required (rule disabled)"
  ],
  "manual_review_required": [
    "src/app/org-chart/org-chart.component.html"
  ]
}
```

---

## 7. File Paths — Full Repository Layout

```
coras-migration-agent/                    ← Agent repository root
│
├── AGENTS.md                             ← THIS FILE — Prompt Stack Design Document
│
├── index.js                              ← Entry point. Parses CLI flags, runs agents in sequence.
├── package.json                          ← Agent's own dependencies (just axios)
│
├── config/
│   └── llm-config.json                   ← FILL IN: LLM credentials, project path, settings
│                                            Do NOT commit API keys. Use .env or CI secrets.
│
├── rules/
│   ├── coras-migration-rules.json        ← ACTIVE rules file. Replaced each release.
│   └── archive/
│       ├── coras-v8.0.5-to-v8.0.6.json  ← Previous release rules (kept for audit)
│       └── coras-v8.x-to-v9.0.1.json    ← 9.0.1 release (example — see Section 8)
│
├── agents/
│   ├── migration-agent.js                ← Agent 1. Reads rules → updates package.json,
│   │                                        angular.json, tsconfig, index.html,
│   │                                        all .ts and .html source files.
│   └── build-agent.js                    ← Agent 2. Downloads Node.js, npm install, ng serve.
│
├── utils/
│   ├── llm-client.js                     ← Axios wrapper. Handles Anthropic/OpenAI/Azure/Ollama.
│   │                                        Retry logic, provider-specific payload formatting.
│   ├── file-utils.js                     ← collectAngularFiles, readJson, writeJson, writeText,
│   │                                        backupProject, findProjectFiles, chunk.
│   └── logger.js                         ← Console + file logging. Produces migration report JSON.
│
├── logs/                                 ← AUTO-CREATED. One .log file per run.
├── reports/                              ← AUTO-CREATED. One .json report per run.
└── migration-backups/                    ← AUTO-CREATED. One directory per apply-mode run.
```

**Target application paths the agent reads and writes:**

```
{project_root}/                           ← Set in llm-config.json → migration.project_root
│
├── package.json                          ← Dependencies bumped (deterministic merge)
├── angular.json                          ← Builder/architect changes (LLM-assisted)
├── tsconfig.json                         ← compilerOptions merge (deterministic)
├── tsconfig.app.json                     ← compilerOptions merge (deterministic)
│
└── src/
    ├── index.html                        ← Theme attribute injection (coras-theme on body)
    └── app/
        ├── **/*.ts                       ← Source files matched by typescript_rules
        └── **/*.html                     ← Template files matched by html_rules
```

---

## 8. Extra Dynamic Rules — Complete Schema

The `coras-migration-rules.json` is authored **once per Coras release** from the CHANGELOG and version announcement. This section defines every field.

### 8.1 Top-Level Structure

```json
{
  "_doc": "Human note — what this file is",
  "migration_meta":        { ... },   // Who, what, when, why
  "package_json_updates":  { ... },   // Dependency version bumps
  "index_html_updates":    { ... },   // <body> attribute changes  (optional block)
  "angular_json_updates":  { ... },   // angular.json path patches (optional block)
  "tsconfig_updates":      { ... },   // tsconfig compilerOptions  (optional block)
  "code_migration_rules":  { ... },   // Per-file LLM rules
  "validation_rules":      { ... }    // Post-migration checks
}
```

### 8.2 `migration_meta`

```json
"migration_meta": {
  "coras_from_version": "8.0.6",
  "coras_to_version":   "9.0.1",
  "angular_base_version": "19",
  "ag_grid_version":      "19",
  "created_by":    "Meenakshi Sundaram",
  "created_date":  "2026-06-19",
  "announcement_ref": "Internal channel post — VERSION RELEASE 9.0.1 — 08/07/2025",
  "changelog_ref": "https://gitlab.internal/devtools/ssjs_frmk/-/blob/develop/CHANGELOG.md",
  "notes": "9.0.1 adds Dark/Light theme, AG Grid 19 compat, column hamburger hide,
            lazy pagination selection retention, i18n text, popup menu API cleanup,
            and tree structural improvements."
}
```

### 8.3 `package_json_updates`

```json
"package_json_updates": {
  "_doc": "Core packages are always updated. Conditional packages are only updated
           if they already exist in the target project's package.json.",

  "dependencies": {
    "@frmk2sjs/services":   "9.0.1",
    "@frmk2sjs/core":       "9.0.1",
    "@frmk2sjs/component":  "9.0.1"
  },

  "conditional_dependencies": {
    "_doc": "Agent inspects the target package.json first. Updates version only
             if the package is already declared. Never adds a new dependency.",
    "coras-for-angular-material":    "9.0.1",
    "coras-for-ag-grid-community":   "9.0.1",
    "coras-for-ag-grid-enterprise":  "9.0.1",
    "coras-for-primeNG":             "9.0.1",
    "coras-for-highcharts":          "9.0.1"
  }
}
```

> **Why conditional?** An app using only the community grid should not suddenly have enterprise grid installed. The agent checks `Object.keys(pkg.dependencies)` before updating.

### 8.4 `index_html_updates` *(optional block)*

```json
"index_html_updates": {
  "_doc": "Changes to src/index.html. Only include this block when a Coras release
           requires structural changes to index.html — not every release does.",

  "rule_id": "CORAS-THEME-001",
  "jira_ticket": "CORAS-THEME-2025",

  "body_attributes": {
    "coras-theme": "light"
    // Key-value pairs of attributes to add to the <body> element.
    // Each attribute is only added if it does not already exist.
  },

  "ai_instruction": "Locate the <body> tag in src/index.html. Add the attribute
    coras-theme='light' if it does not already exist. If a coras-theme attribute
    already exists with any value (even 'dark'), do NOT overwrite it — instead
    add a WARNING to the report stating 'coras-theme already set to X, preserved
    as-is — human review required'. Do not change any other attributes or content.",

  "example_before": "<body>\n  <app-root></app-root>\n</body>",
  "example_after":  "<body coras-theme=\"light\">\n  <app-root></app-root>\n</body>"
}
```

### 8.5 `angular_json_updates` *(optional block)*

```json
"angular_json_updates": {
  "_doc": "Dot-notation paths in angular.json to update. LLM applies these as
           a targeted merge — all other content is preserved exactly.",

  "projects.*.architect.build.builder":
    "@angular-devkit/build-angular:application",

  "projects.*.architect.build.options.outputPath": "dist/{{projectName}}",

  "_migration_notes": [
    "{{projectName}} is automatically resolved to the actual project name by the agent",
    "Only add entries here for paths that must change — leave everything else untouched"
  ]
}
```

### 8.6 `tsconfig_updates` *(optional block)*

```json
"tsconfig_updates": {
  "_doc": "compilerOptions merged into tsconfig.json AND tsconfig.app.json.
           Agent does a deterministic key-by-key merge — no LLM needed for this.",
  "compilerOptions": {
    "target":                   "ES2022",
    "module":                   "ES2022",
    "lib":                      ["ES2022", "dom"],
    "useDefineForClassFields":  false,
    "experimentalDecorators":   true,
    "emitDecoratorMetadata":    true,
    "strictTemplates":          true
  }
}
```

### 8.7 `code_migration_rules` — Per-Rule Schema

This is the most powerful section. Each rule is an object with these mandatory fields:

```json
{
  "id":             "CORAS-HTML-001",
  // Convention: CORAS-TS-NNN for TypeScript rules, CORAS-HTML-NNN for templates.
  // Must be unique within this file.

  "name":           "Short human-readable rule name",

  "enabled":        true,
  // Set to false for risky or layout-sensitive rules — agent flags files
  // in manual_review_required instead of applying the change.

  "jira_ticket":    "TCAAS1443-645",
  // The CHANGELOG JIRA ticket that drove this rule. Provides full traceability.

  "detect_pattern": "coras-grid",
  // A string that must appear literally in the file for the LLM to be invoked.
  // Files where this string is absent are skipped with zero LLM cost.
  // Choose something specific enough to avoid false positives.

  "ai_instruction": "Full instruction to the LLM. Must include:
    1. What to find (specific selector, attribute name, import path, class name)
    2. What to change (exact rename, new attribute, new value)
    3. What NOT to change (most important — prevents over-reaching)
    4. Edge cases (what if the attribute already exists? what if the file has both old and new?)
    5. Safe default if a value cannot be inferred from context",

  "example_before": "<!-- real code snippet from an actual Coras app before the change -->",
  "example_after":  "<!-- real code snippet from an actual Coras app after the change -->"
}
```

### 8.8 Complete v9.0.1 Rules — All Code Rules

**TypeScript rules:**

```json
"typescript_rules": [
  {
    "id": "CORAS-TS-001",
    "name": "i18n config on CorasGridConfig",
    "enabled": true,
    "jira_ticket": "TCAAS1443-635",
    "detect_pattern": "CorasGridConfig",
    "ai_instruction": "Find TypeScript objects typed as CorasGridConfig (or assigned to a
      variable used as a CorasGridConfig). If the object does not already have an 'i18n'
      or 'locale' property, add 'i18n: { enabled: true }' as a new property at the end
      of the object literal. Do not change objects that already have i18n settings.
      Preserve all existing properties and their values exactly. Do not add this to
      any object that is not a CorasGridConfig.",
    "example_before": "const gridConfig: CorasGridConfig = {\n  pagination: true,\n  pageSize: 25\n};",
    "example_after":  "const gridConfig: CorasGridConfig = {\n  pagination: true,\n  pageSize: 25,\n  i18n: { enabled: true }\n};"
  },
  {
    "id": "CORAS-TS-002",
    "name": "CorasPopupMenuService import path rename",
    "enabled": true,
    "jira_ticket": "TCAAS1443-620",
    "detect_pattern": "CorasPopupMenuService",
    "ai_instruction": "Find import statements that import CorasPopupMenuService from
      '@frmk2sjs/component/popup'. Rename the import path to '@frmk2sjs/component/menu'.
      The imported symbol name stays the same. If the import is already from
      '@frmk2sjs/component/menu', skip this file. Do not change any other imports.",
    "example_before": "import { CorasPopupMenuService } from '@frmk2sjs/component/popup';",
    "example_after":  "import { CorasPopupMenuService } from '@frmk2sjs/component/menu';"
  },
  {
    "id": "CORAS-TS-003",
    "name": "retainSelection on lazy grid configs",
    "enabled": true,
    "jira_ticket": "TCAAS1443-640",
    "detect_pattern": "lazyLoad",
    "ai_instruction": "Find TypeScript configuration objects that have 'lazyLoad: true' set.
      These are objects used to configure Coras grids. If 'retainSelection' is not
      already defined in the same object, add 'retainSelection: true' immediately after
      the 'lazyLoad: true' line. Only modify objects where lazyLoad is explicitly true.
      Do not add retainSelection to objects where lazyLoad is false or absent.",
    "example_before": "const config = {\n  lazyLoad: true,\n  pageSize: 50\n};",
    "example_after":  "const config = {\n  lazyLoad: true,\n  retainSelection: true,\n  pageSize: 50\n};"
  }
]
```

**HTML template rules:**

```json
"html_rules": [
  {
    "id": "CORAS-HTML-001",
    "name": "Column hamburger hide on coras-grid",
    "enabled": true,
    "jira_ticket": "TCAAS1443-645",
    "detect_pattern": "coras-grid",
    "ai_instruction": "Find all <coras-grid> elements. In v9.0.1 a new [showColumnMenu]
      input enables the column-level hamburger hide option. If a <coras-grid> element
      does not already have [showColumnMenu] or showColumnMenu as an attribute, add
      [showColumnMenu]=\"true\" as a new attribute. Preserve all existing attributes
      and their order. Do not add the attribute if it is already present with any value.",
    "example_before": "<coras-grid [columnDefs]=\"cols\" [rowData]=\"rows\"></coras-grid>",
    "example_after":  "<coras-grid [columnDefs]=\"cols\" [rowData]=\"rows\" [showColumnMenu]=\"true\"></coras-grid>"
  },
  {
    "id": "CORAS-HTML-002",
    "name": "Retain selection on lazy-loaded grids",
    "enabled": true,
    "jira_ticket": "TCAAS1443-640",
    "detect_pattern": "[lazy]",
    "ai_instruction": "Find <coras-grid> or <coras-table> elements that have
      [lazy]=\"true\" or [lazy]='true'. If [retainSelection] is not already present
      on the element, add [retainSelection]=\"true\" as a new attribute immediately
      after the [lazy] binding. Only modify elements that have the lazy binding.
      Elements without [lazy]='true' must not be touched.",
    "example_before": "<coras-grid [lazy]=\"true\" [columnDefs]=\"cols\"></coras-grid>",
    "example_after":  "<coras-grid [lazy]=\"true\" [retainSelection]=\"true\" [columnDefs]=\"cols\"></coras-grid>"
  },
  {
    "id": "CORAS-HTML-003",
    "name": "Popup menu — rename [popupItems] to [menuItems]",
    "enabled": true,
    "jira_ticket": "TCAAS1443-620",
    "detect_pattern": "popupItems",
    "ai_instruction": "Find all occurrences of [popupItems] on <coras-popup-menu> elements.
      Rename the input binding from [popupItems] to [menuItems]. The bound expression
      value must remain unchanged — only the input name changes. Do not modify any
      element that is not coras-popup-menu. Do not change (menuItemClick) or any
      other attributes.",
    "example_before": "<coras-popup-menu [popupItems]=\"actions\" (menuItemClick)=\"onAction($event)\"></coras-popup-menu>",
    "example_after":  "<coras-popup-menu [menuItems]=\"actions\" (menuItemClick)=\"onAction($event)\"></coras-popup-menu>"
  },
  {
    "id": "CORAS-HTML-004",
    "name": "i18n locale binding on coras-grid",
    "enabled": true,
    "jira_ticket": "TCAAS1443-635",
    "detect_pattern": "coras-grid",
    "ai_instruction": "Find <coras-grid> elements. If the element does not already have
      a [locale] or [i18nConfig] binding, add [locale]=\"currentLocale\" as a new
      attribute. If the surrounding component TypeScript file (same name, .ts extension)
      does not appear to declare currentLocale, add [locale]=\"'en-US'\" as a safe
      default and add an HTML comment immediately above: <!-- TODO: wire up dynamic
      locale (TCAAS1443-635) -->. Do not change elements that already have locale
      or i18nConfig bindings.",
    "example_before": "<coras-grid [columnDefs]=\"cols\" [rowData]=\"data\"></coras-grid>",
    "example_after":  "<coras-grid [columnDefs]=\"cols\" [rowData]=\"data\" [locale]=\"currentLocale\"></coras-grid>"
  },
  {
    "id": "CORAS-HTML-005",
    "name": "Coras Tree structural improvements — MANUAL REVIEW",
    "enabled": false,
    "jira_ticket": "TCAAS1443-622",
    "detect_pattern": "coras-tree",
    "ai_instruction": "RULE DISABLED — Tree layout changes require human visual review.
      Do NOT apply any automatic changes. Add an HTML comment immediately above each
      <coras-tree> element: <!-- TODO: Review for v9.0.1 tree improvements
      (TCAAS1443-622) - manual review required -->. Log the file path in the
      manual_review_required report section.",
    "example_before": "<coras-tree [nodes]=\"data\"></coras-tree>",
    "example_after":  "<!-- TODO: Review for v9.0.1 tree improvements (TCAAS1443-622) -->\n<coras-tree [nodes]=\"data\"></coras-tree>"
  }
]
```

### 8.9 `validation_rules`

```json
"validation_rules": {
  "_doc": "Automated checks after all transforms complete.",
  "checks": [
    "package.json: @frmk2sjs/services equals 9.0.1",
    "package.json: @frmk2sjs/core equals 9.0.1",
    "package.json: @frmk2sjs/component equals 9.0.1",
    "package.json: all present coras-for-* packages equal 9.0.1",
    "src/index.html: <body> has coras-theme attribute",
    "No .html files contain [popupItems] binding (CORAS-HTML-003)",
    "No .ts files import from @frmk2sjs/component/popup (CORAS-TS-002)",
    "tsc --noEmit passes (run by build agent)"
  ],
  "manual_review_flags": [
    "Files containing <coras-tree> (CORAS-HTML-005 — layout change, disabled rule)",
    "Files where existing coras-theme differed from 'light'"
  ]
}
```

---

## 9. How to Use with Roo Code

**Roo Code** is the AI coding assistant extension for VS Code (formerly Roo Cline). It connects directly to your LLM provider using your configured API key and can orchestrate multi-step agentic tasks from within the IDE.

### 9.1 Prerequisites

- VS Code with the **Roo Code** extension installed  
- An **Anthropic API key** (or OpenAI) configured in Roo Code's settings  
- The `coras-migration-agent` repository cloned locally  
- The target Angular application available on your file system  
- Node.js ≥ 18 installed  

### 9.2 Setup — One Time

**Step 1: Open the agent repo in VS Code**

```
File → Open Folder → select coras-migration-agent/
```

**Step 2: Install agent dependencies**

Open the VS Code terminal (Ctrl+`) and run:

```bash
npm install
```

**Step 3: Configure your credentials**

Edit `config/llm-config.json`. For Roo Code, set:

```json
{
  "llm": {
    "provider": "anthropic",
    "base_url": "https://api.anthropic.com/v1",
    "api_key": "YOUR_ANTHROPIC_KEY_HERE",
    "model": "claude-sonnet-4-6",
    "temperature": 0.1,
    "max_tokens": 8000
  }
}
```

> **Security:** Do not commit `llm-config.json` with your key. Add it to `.gitignore`. Alternatively, export `ANTHROPIC_API_KEY` as an environment variable and the agent will pick it up automatically.

**Step 4: Set your project path**

In `config/llm-config.json`, set `migration.project_root` to the path of the Angular app:

```json
"migration": {
  "project_root": "../path/to/your-angular-app",
  "dry_run": true
}
```

### 9.3 Workflow A — Run the Agent Directly from Roo Code Chat

Open the Roo Code panel (click the Roo Code icon in the activity bar), then type:

```
Run the Coras migration agent in dry-run mode against the project at ../my-angular-app.
The rules file is at ./rules/coras-migration-rules.json.
Use the config at ./config/llm-config.json.
Show me the migration summary when done.
```

Roo Code will:
1. Open a terminal and run `node index.js --migrate-only`
2. Stream the console output back to you
3. Summarise the report

To apply changes after reviewing the dry-run:

```
The dry-run report looks good. Set dry_run to false in the config and run the migration.
```

### 9.4 Workflow B — Roo Code Edits the Rules File, Then Runs the Agent

This is the **recommended Roo Code workflow** for authoring a new rules file from a version announcement. Paste the announcement text into Roo Code:

```
I have a new Coras version announcement. Based on the information below, create a 
coras-migration-rules.json file in the ./rules/ directory for migrating from 
Coras 8.0.6 to 9.0.1. Follow the schema in AGENTS.md Section 8.

[paste the full announcement text and CHANGELOG entries here]
```

Roo Code will:
1. Read `AGENTS.md` to understand the schema
2. Parse the announcement for version numbers, package names, and CHANGELOG entries
3. Create a complete, correctly structured rules file in `./rules/`
4. Ask you to review it before proceeding

Then prompt it to run:

```
The rules file looks correct. Run the agent in dry-run mode now.
```

### 9.5 Workflow C — Roo Code as Batch Orchestrator (All 75 Apps)

Create a file `config/app-registry.json`:

```json
[
  { "name": "wholesale-portal",   "path": "/projects/wholesale-portal" },
  { "name": "cib-dashboard",      "path": "/projects/cib-dashboard" },
  { "name": "bp2s-reports",       "path": "/projects/bp2s-reports" }
]
```

Then prompt Roo Code:

```
Using the app-registry.json, run the Coras migration agent in dry-run mode against 
each application in sequence. After each app, capture the report summary. Once all 
apps have been processed, produce a consolidated report table showing which apps 
need the most changes and any errors encountered.
```

Roo Code will loop through each app, update `project_root` in the config, run the agent, and collect results.

### 9.6 Roo Code `.roo/` Custom Mode (Advanced)

For repeatable, structured Roo Code behaviour, add a custom mode file:

**`.roo/modes/coras-migration.json`:**

```json
{
  "name": "Coras Migration Agent",
  "description": "Runs and manages the Coras Framework Migration Agent",
  "instructions": "You are assisting with Coras Framework migrations. Always read AGENTS.md first to understand the rules schema. Never modify source files directly — always go through the agent (node index.js). Always run in dry_run mode first. Always review the report before applying. When authoring a new rules file, follow the schema in AGENTS.md Section 8 exactly.",
  "tools": ["read_file", "write_file", "execute_command", "list_files"],
  "context_files": ["AGENTS.md", "config/llm-config.json", "rules/coras-migration-rules.json"]
}
```

Activate this mode from the Roo Code mode selector before starting a migration session.

---

## 10. How to Use with OpenCode

**OpenCode** is a terminal-first AI coding agent (`opencode` CLI). It operates from the command line, reads your project files, and can execute shell commands — making it ideal for scripted or CI-driven migration workflows.

### 10.1 Prerequisites

- OpenCode CLI installed (`npm install -g opencode` or follow opencode.ai docs)
- An API key configured (Anthropic or OpenAI)
- The `coras-migration-agent` repository cloned locally
- Node.js ≥ 18

### 10.2 Setup — OpenCode Configuration File

The agent ships with `opencode.json` at the repo root. This is the OpenCode project configuration:

**`opencode.json`** (in `coras-migration-agent/` root):

```json
{
  "$schema": "https://opencode.ai/config.json",

  "model": "claude-sonnet-4-6",
  // Use the same model as in llm-config.json for consistency.
  // OpenCode uses your ANTHROPIC_API_KEY or OPENAI_API_KEY env var — no key in this file.

  "instructions": "You are the Coras Framework Migration Agent assistant. Your role is to help engineer Coras framework version migrations across Angular applications. Always read AGENTS.md before taking any action to understand the current rules schema. Never directly edit Angular application source files — only operate the Node.js agent (node index.js). Run in dry_run mode first. Review reports before applying. When authoring rules files, follow the exact schema documented in AGENTS.md Section 8.",

  "context": [
    "AGENTS.md",
    "rules/coras-migration-rules.json",
    "config/llm-config.json",
    "index.js",
    "agents/migration-agent.js"
  ]
  // Files automatically loaded into OpenCode context at session start.
}
```

### 10.3 Setup — Credentials for OpenCode

OpenCode reads credentials from environment variables. Set them before starting a session:

```bash
# Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."

# OR OpenAI
export OPENAI_API_KEY="sk-..."

# IMPORTANT: Also set your project path for this app
export CORAS_PROJECT_ROOT="/path/to/your-angular-app"
```

Update `config/llm-config.json` to use the environment variable path:

```json
"migration": {
  "project_root": "${CORAS_PROJECT_ROOT}",
  "dry_run": true
}
```

Or just edit the JSON directly before each run — it takes 5 seconds.

### 10.4 Workflow A — Interactive OpenCode Session

Navigate to the agent directory and start OpenCode:

```bash
cd coras-migration-agent
opencode
```

OpenCode loads `opencode.json` and its context files automatically. Then chat:

```
> Read the AGENTS.md and tell me what the current active rules file targets.

> The Coras team just released v9.0.1. Here is the announcement:
  [paste announcement]
  [paste CHANGELOG entries]
  
  Create the rules file for this migration.

> Run the agent in dry-run mode against /projects/my-angular-app

> Show me the migration report summary.

> Apply the migration now (I've reviewed the report and it looks correct).
```

OpenCode will execute `node index.js --migrate-only` via shell, stream the output, and parse the report for you.

### 10.5 Workflow B — Non-Interactive / Scripted OpenCode

For running without interactive input (e.g., in a script):

```bash
# Dry run a specific app
opencode run "Run the Coras migration agent in dry-run mode against the project at \
  /projects/wholesale-portal using rules at ./rules/coras-migration-rules.json. \
  Show the migration summary."

# Apply migration
opencode run "Set dry_run to false in config/llm-config.json and run the migration \
  agent against /projects/wholesale-portal. Show the final report."

# Restore dry_run afterwards
opencode run "Set dry_run back to true in config/llm-config.json."
```

### 10.6 Workflow C — Batch All 75 Apps with a Shell Script + OpenCode

Create `scripts/batch-migrate.sh`:

```bash
#!/usr/bin/env bash
# Runs Coras migration dry-run across all registered applications
# Usage: bash scripts/batch-migrate.sh ./config/app-registry.json

APP_REGISTRY="${1:-./config/app-registry.json}"
RULES="./rules/coras-migration-rules.json"
REPORT_DIR="./reports/batch-$(date +%Y-%m-%d)"
mkdir -p "$REPORT_DIR"

echo "=== Coras Batch Migration — $(date) ===" | tee "$REPORT_DIR/batch.log"

# Read app list using jq
jq -r '.[] | "\(.name)|\(.path)"' "$APP_REGISTRY" | while IFS='|' read -r APP_NAME APP_PATH; do
  echo ""
  echo "--- Migrating: $APP_NAME ---" | tee -a "$REPORT_DIR/batch.log"

  # Update project_root in config for this app
  # (Uses node to do the JSON edit safely)
  node -e "
    const fs = require('fs');
    const cfg = JSON.parse(fs.readFileSync('./config/llm-config.json', 'utf8'));
    cfg.migration.project_root = '$APP_PATH';
    cfg.migration.dry_run = true;
    fs.writeFileSync('./config/llm-config.json', JSON.stringify(cfg, null, 2));
  "

  # Run migration agent
  node index.js --migrate-only 2>&1 | tee -a "$REPORT_DIR/batch.log"

  # Copy this app's report
  LATEST_REPORT=$(ls -t ./reports/migration-report-*.json | head -1)
  if [ -n "$LATEST_REPORT" ]; then
    cp "$LATEST_REPORT" "$REPORT_DIR/${APP_NAME}-report.json"
  fi

done

echo ""
echo "=== Batch complete. Reports in: $REPORT_DIR ===" | tee -a "$REPORT_DIR/batch.log"
```

Then ask OpenCode to summarise results:

```bash
opencode run "Read all JSON files in ./reports/batch-$(date +%Y-%m-%d)/ and produce 
a consolidated table showing: app name, files modified, errors, warnings, and whether 
any manual review is required. Format as a markdown table."
```

### 10.7 OpenCode Prompts for Common Tasks

Copy-paste these into an OpenCode session as needed:

**Author a new rules file from a version announcement:**
```
New Coras version released: [VERSION]. 
Announcement: [paste text]  
CHANGELOG entries: [paste entries]
Read AGENTS.md Section 8 for the schema, then create ./rules/coras-v{FROM}-to-v{TO}.json
```

**Run dry-run on a single app:**
```
Update config/llm-config.json to set project_root to [APP_PATH] and dry_run to true.
Then run: node index.js --migrate-only
Show me the report.
```

**Apply migration after approval:**
```
Set dry_run to false in config/llm-config.json, then run: node index.js --migrate-only
Show me the final change count.
Then set dry_run back to true.
```

**Run full pipeline (migrate + install + serve):**
```
Set project_root to [APP_PATH] in the config.
Run the full pipeline: node index.js
Keep me updated on each step.
```

**Validate rules file before batch rollout:**
```
Read rules/coras-migration-rules.json and check it against the authoring checklist 
in AGENTS.md Section 12. List any issues found.
```

---

## 11. CLI Reference & End-to-End Runbook

### 11.1 Full CLI Reference

```bash
# ─── Run Modes ─────────────────────────────────────────────────────────────
node index.js                            # Full pipeline: migrate + npm install + serve
node index.js --migrate-only             # Migration agent only
node index.js --build-only              # Build agent only (npm install + serve)

# ─── Config Overrides ─────────────────────────────────────────────────────
node index.js --config ./path/config.json   # Custom config file
node index.js --rules ./path/rules.json     # Custom rules file

# ─── npm shortcuts ────────────────────────────────────────────────────────
npm run migrate          # → node index.js --migrate-only
npm run build            # → node index.js --build-only
npm run dry-run          # → node index.js --migrate-only (dry_run must be true in config)
npm start                # → node index.js (full pipeline)

# ─── Debug ────────────────────────────────────────────────────────────────
DEBUG=1 node index.js    # Full error stack traces
```

### 11.2 Step-by-Step: Migrating One Application

```bash
# ── STEP 0: Angular core migration (only if Angular version is also bumping) ─
cd /path/to/app
ng update @angular/core @angular/cli --from=18 --to=19
cd /path/to/coras-migration-agent

# ── STEP 1: Configure ───────────────────────────────────────────────────────
# Edit config/llm-config.json:
#   migration.project_root = "../path/to/your-angular-app"
#   migration.dry_run = true   (ALWAYS start here)

# ── STEP 2: Dry run ─────────────────────────────────────────────────────────
node index.js --migrate-only
# Review console output and reports/migration-report-*.json

# ── STEP 3: Apply ───────────────────────────────────────────────────────────
# Edit config/llm-config.json:  dry_run = false
node index.js --migrate-only

# ── STEP 4: Validate build ──────────────────────────────────────────────────
node index.js --build-only
# Runs: npm install --legacy-peer-deps, then ng serve (or your configured command)

# ── STEP 5: Review diff and commit ──────────────────────────────────────────
cd /path/to/your-angular-app
git diff
git checkout -b chore/coras-migration-v9.0.1
git add -A
git commit -m "chore: migrate Coras framework 8.0.6 → 9.0.1 [automated]"
git push origin chore/coras-migration-v9.0.1
# Open PR → review → merge
```

### 11.3 Step-by-Step: Rolling Out Across All 75 Applications

```bash
# ── STEP 1: Verify rules file on a pilot app ──────────────────────────────
# Pick one representative app and do a full apply + build cycle.
# Fix any issues in the rules file before batch rollout.

# ── STEP 2: Build app-registry.json ─────────────────────────────────────
# One-time: create config/app-registry.json with all 75 apps

# ── STEP 3: Dry-run batch ───────────────────────────────────────────────
bash scripts/batch-migrate.sh ./config/app-registry.json
# Review consolidated report

# ── STEP 4: Apply batch ─────────────────────────────────────────────────
# Update batch-migrate.sh: set dry_run = false
# Run again
bash scripts/batch-migrate.sh ./config/app-registry.json

# ── STEP 5: Each team raises a PR ───────────────────────────────────────
# Each app team reviews the automated PR, runs their CI, and merges.
```

---

## 12. Rule Authoring Guide — Every Release

### 12.1 Trigger: When to Author a New Rules File

When the Coras team posts a version announcement on the internal channel.

### 12.2 Input Sources

| Source | What to Extract |
|---|---|
| Version announcement | New version number; package list; any `index.html` or `body` attribute instructions; compatibility notes |
| CHANGELOG.md | JIRA ticket IDs; one-line descriptions of each change; whether the change is a new feature (→ new rule) or a bug fix with no API change (→ no rule) |

### 12.3 Classification Table

For each CHANGELOG entry, use this table to decide what to create:

| Type of change | Rule needed? | Type |
|---|---|---|
| New input binding added (consumers should adopt it) | Yes | `html_rules` |
| Deprecated binding renamed | Yes — enabled:true | `html_rules` or `typescript_rules` |
| Import path changed | Yes | `typescript_rules` |
| New mandatory `index.html` attribute | Yes | `index_html_updates` block |
| New optional config property | Yes — enabled:true | `typescript_rules` |
| Bug fix, no API surface change | **No** | — |
| Internal refactor, no consumer impact | **No** | — |
| Structural layout / CSS change | Yes — **enabled:false** | `html_rules` (flagged for review) |

### 12.4 `ai_instruction` Writing Checklist

Before finalising each rule, confirm the `ai_instruction` answers all of these:

- [ ] **What to find:** specific selector, class name, attribute, or import path
- [ ] **What to change:** exact rename, attribute to add, value to set
- [ ] **What NOT to change:** explicitly stated — prevents LLM from over-reaching
- [ ] **Already present:** what to do if the new value already exists (skip, warn, or overwrite)
- [ ] **Safe default:** if a value cannot be inferred from context, use this fallback
- [ ] **`example_before` is real code:** from an actual Coras application, not made up
- [ ] **`example_after` is correct:** verified by running against a test app

### 12.5 `detect_pattern` Rules

- Must be a string that appears **literally** in the file
- Must be **specific enough** to avoid false positives (`coras-grid` yes, `grid` no)
- Must be **in the right layer** (`typescript_rules` patterns should appear in `.ts`, `html_rules` in `.html`)
- Test with: `grep -r "your-pattern" src/` before declaring it

### 12.6 Archive and Version the Rules File

```bash
# Before every batch rollout, archive the previous rules file
cp rules/coras-migration-rules.json \
   rules/archive/coras-v8.x-to-v9.0.1.json

# Set the new file as active
cp rules/coras-v9.x-to-v10.0.0.json \
   rules/coras-migration-rules.json

# Commit both
git add rules/
git commit -m "rules: add Coras v9 → v10 migration rules (archive v8→v9)"
```

---

## 13. Management Justification

### 13.1 Business Case

| Metric | Manual (before agent) | Automated (with agent) | Saving |
|---|---|---|---|
| Time per application | 3–5 engineer-days | < 30 minutes supervised | ~97% reduction |
| 75 applications | 225–375 engineer-days | ~2 engineer-days (rule authoring + review) | ~99% reduction |
| Human error rate | High — manual file editing across hundreds of files | Near-zero — deterministic rules + Git diff review | Significant risk reduction |
| Rollback capability | Ad hoc, manual, slow | Automatic timestamped backup before every apply | Instant |
| Audit trail | None to inconsistent | Full JSON report per app per run, traceable to JIRA ticket | Complete |
| Rules reuse | Zero — every migration rebuilt from scratch | Rules authored once, reused across all 75 apps | 75× leverage |
| Time to adopt a new Coras release across the estate | 4–8 weeks | 3–5 business days | 70–85% faster |

### 13.2 Risk Management

**Dry-run by default.** No file is ever modified without a human reviewing the report. The agent cannot skip this gate — `dry_run: true` is the default and must be explicitly overridden.

**Automatic backup.** Before any apply-mode run, the full project is copied to a timestamped directory. Rollback is a directory copy.

**Conservative LLM instruction.** Every rule is explicitly instructed to change only what is declared and to flag uncertainty rather than guess. The `detect_pattern` pre-filter means the LLM is only invoked on files where a relevant pattern is found.

**CI as the final gate.** The agent is not a replacement for CI. A passing `ng build` and test suite is always required before merging — the build agent step enables this.

**Versioned rules archive.** Every rules file is kept. Any migration can be reproduced or audited months later.

### 13.3 Strategic Value

Coras serves 75 applications and the estate grows. Without automation, every version release creates a coordination burden: 75 teams must be notified, each must manually apply the same set of changes, each must run their own QA. This does not scale.

The Prompt Stack Design approach makes the agent's behaviour **transparent and auditable**: every code change made by the AI is traceable to a specific rule in the rules file, which is in turn traceable to a JIRA ticket in the CHANGELOG. There is no "magic" — just AI applying human-authored rules consistently at scale.

This agent is the only viable path to keeping the entire application estate on a supported, secure, current version of the Coras Framework as its release cadence continues.

---

## 14. Governance & Contacts

| Role | Responsibility |
|---|---|
| **Coras Framework Team** | Authors `coras-migration-rules.json` for each release; publishes version announcement; owns the framework packages |
| **Application Engineers** | Run the agent against their application; review dry-run report; approve and commit |
| **Tech Lead / Architect** | Reviews rules file before batch rollout; signs off on agents being run at scale |
| **DevOps / CI** | Ensures `ng build` and `tsc --noEmit` are CI acceptance gates post-migration |

### 14.1 Agent Repository Ownership

The `coras-migration-agent` repository is owned by the Coras Framework Team. Application engineers clone it but do not commit to it. Rules files submitted by the team are the authoritative source of truth.

### 14.2 Contacts

- **Rules file questions / new feature requests:** Coras Framework Team channel  
- **Agent bugs or feature requests:** raise a ticket in the agent repository  
- **Per-application migration issues:** raise with your application's tech lead  

---

*This PSD is versioned alongside the agent code. Update it whenever the agent architecture, rule schema, or toolchain integration changes. The rules file is the living artefact; this document is the stable contract.*

---

**Document end — AGENTS.md v2.0**
