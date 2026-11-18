# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

dotpm (plugin id `project-manager`) is an Obsidian plugin that does project management over plain Markdown notes: Table, Gantt timeline and Kanban views, a local HTTP/MCP API, and single-file HTML export. A project is a note with `pm-project: true`; each task is its own note with `pm-task: true`, stored in a `<project>_tasks/` folder next to the project note.

## Commands

pnpm workspace on Node 24 (`packageManager` pins pnpm 11).

```sh
pnpm install
pnpm dev                 # watch build; writes main.js/styles.css into $VAULT_PATH/.obsidian/plugins/project-manager (repo root if unset)
pnpm build               # production build (viewer template, then plugin JS and CSS)
pnpm check               # oxlint + oxfmt --check + tsc, run in parallel
pnpm check:submission    # eslint with eslint-plugin-obsidianmd, --max-warnings 0 (Obsidian review rules)
pnpm fix                 # oxlint --fix + oxfmt
pnpm test                # vitest run
pnpm test:coverage
pnpm build:packages      # build the publishable @dotpm/core, @dotpm/api, @dotpm/viewer dist
```

Single test file or test name:

```sh
pnpm vitest run src/store/ProjectStore.test.ts
pnpm vitest run -t "daysBetween"
```

CI (and the PR template) requires `pnpm check`, `pnpm check:submission` and `pnpm test` to pass. Formatting: 120 cols, no semicolons, single quotes, no trailing commas.

## Architecture

### Layers and their boundaries

- **`packages/core` (`@dotpm/core`)** – the domain with no host dependency: types, dates (Temporal polyfill), i18n, filtering/sorting/grouping, the dependency scheduler, task-tree ops, and the YAML pipeline (`YamlParser` → `YamlHydrator` → `YamlSerializer`). It must not import `obsidian` or `@dotpm/ui` (enforced by oxlint `no-restricted-imports`).
- **`packages/api` (`@dotpm/api`)** – the API contract: `DomainApi` interface and wire resources (`contract.ts`, `resources.ts`), the HTTP router, the MCP handler and a client. `./testing` exports a fake `DomainApi`.
- **`packages/ui` (`@dotpm/ui`, private)** – primitives, composites, Gantt pieces and read-only snapshot renderers (`views/snapshot*.ts`). It may not import `obsidian`. Instead the host injects a `UiPlatform` via `setPlatform()` (icons, buttons, menus, notices). Obsidian's DOM helpers (`createDiv`, `empty`, `setText`…) are polyfilled by `dom-shim.ts` outside Obsidian, with `dom-platform.ts` as the browser `UiPlatform`.
- **`packages/viewer`** – the standalone page used by HTML export. `scripts/build-viewer.mjs` bundles it into `packages/viewer/dist/viewer.html`, which the plugin build inlines as `__VIEWER_TEMPLATE__`. The plugin build only warns if the template is missing (exports then refuse), so run `pnpm build:viewer` first or use `pnpm dev`/`pnpm build`.
- **`src/`** – the Obsidian plugin itself. `src/platform.ts` (`installObsidianPlatform`) is where Obsidian's YAML codec, locale and UI components get wired into core/ui at startup. core/ui expect `setYamlCodec`, `setLocale` and `setPlatform` to have been called first.

Import aliases in `src/`: `#main`, `#store`, `#store/*`, `#ui/*`, `#modals/*`, `#test/*` (see `package.json` `imports`). Workspace packages resolve to their `src/` via `exports`, so no build step is needed between them during development.

### Plugin runtime (`src/`)

- **`main.ts` (`PMPlugin`)** wires everything: settings, `VaultIndex`, `ProjectStore`, view registration, commands, undo/redo stacks, migrations and the startup sweep, the `LocalApiServer`, and the markdown-leaf swap that opens pm notes in dotpm views.
- **`store/VaultIndex.ts`** is a lightweight index built from Obsidian's metadata cache (`ProjectRef`/`TaskRef`, the project parent tree, archived state). It answers listing/lookup questions without loading full projects.
- **`store/TaskSource.ts`** is the persistence interface that views, modals, commands and the API program against (`plugin.store`). **`store/ProjectStore.ts`** implements it over the markdown files: it caches one `Project` instance per path, writes frontmatter-only changes via `processFrontMatter` and body changes via `vault.process`, records its own writes (`markSelfWrite`) so vault events don't trigger redundant reloads, and notifies views through `onProjectChanged`. Views and modals must read status/priority palettes through `store.configFor(project)` (per-project overrides over global settings), never from settings directly.
- **`store/ProjectScope.ts`** – a view can span a project, its subtree, a folder, the whole vault, or a hand-picked set (`ScopeSpec`).
- **`views/`** – `PMViewRouter` opens views by type. `ProjectView` hosts the Table/Gantt/Kanban sub-views (`SubView`). Dashboard, overview, edit, task and release-notes views sit alongside it. `views/styleguide/` is a dev-only component gallery compiled in only when `__STYLEGUIDE__` is true (non-production builds, or `STYLEGUIDE=1`).
- **`api/`** – `LocalApi` implements `@dotpm/api`'s `DomainApi` on top of the store. `LocalApiServer` serves it (desktop only, `127.0.0.1`, bearer token). `ChangeLog` feeds change polling. API writes go through the same store code as the UI.
- **`migration.ts`** upgrades older vault formats (for example tasks embedded in project frontmatter → one file per task, legacy refs and board config) at startup.
- **`integrations/`** – TaskNotes import and shared palettes.

Build-time globals: `__STYLEGUIDE__`, `__VIEWER_TEMPLATE__`, `__CHANGELOG__` (`CHANGELOG.md` is embedded for the in-app release notes). They are declared in `src/env.d.ts` and stubbed in `vitest.config.ts`.

### Styles

CSS lives in `src/styles/*.css` with `index.css` as the entry. It is bundled by lightningcss through `scripts/build-styles.mjs` into `styles.css`, not by tsdown.

### Tests

Vitest runs `src/**/*.test.ts` and `packages/*/src/**/*.test.ts` in a node environment. `obsidian` is aliased to `test/obsidian-stub.ts`. `test/setup.ts` installs the `yaml` codec, the DOM shim and the DOM platform. Store tests build an in-memory vault with `makeFakeApp()` from `#test/fakeVault`. Tests that need a DOM opt in with a `// @vitest-environment happy-dom` comment on the first line.

## Conventions

### Code

- Strings shown in the UI must be sentence case, and command ids/names must not include "command", the plugin id or the plugin name. `oxlint` and `check:submission` enforce these and the other `obsidianmd/*` rules (for example `window.setTimeout` over bare `setTimeout`, no static style assignment).
- If a UI component changes, update the styleguide gallery (`src/views/styleguide/`) too.
- User docs for the API, CLI and TaskNotes integration are in `docs/`. The `dotpm` CLI (`@dotpm/cli`) is not in this repo.

### Translations

- UI strings go through `t()`/`tn()` from `@dotpm/core`. `packages/core/src/locales/en.json` is the source of truth (it defines the `MessageKey` type), and other locales fall back to English per key at runtime.
- In practice every locale file carries every English key. When you add or rename a key, update all 14 files in `packages/core/src/locales/`, keeping them sorted alphabetically like `en.json`.
- `i18n.test.ts` fails if a locale has a key English lacks, uses different `{placeholders}` than English, or misses a plural category its language needs (`Intl.PluralRules`). Plural keys need the language's own forms: `.one`/`.other` for English, `.few`/`.many` for Polish and Russian, only `.other` for Japanese, Korean and Chinese.
- The language follows Obsidian's own setting (`getLanguage()`). The maintainers turned down a separate language selector in PR #343. A new language is added as a JSON file in `locales/` plus an entry in `catalogs` in `i18n.ts`.

### Commits, branches and PRs

- Commit messages and PR titles are a short, lowercase, imperative sentence in plain English describing the effect on the user, with no Conventional Commits prefix. Examples: `keep dragged table column widths and let the task column resize`, `fall back to status columns when a board groups by a missing field`.
- Branch names are kebab-case descriptions such as `fix-table-column-resize` or `timeline-zoom`. Work goes through PRs into `main`.
- PR descriptions follow `.github/PULL_REQUEST_TEMPLATE.md` (What / How to check / Checklist). Maintainer PRs are often a single prose paragraph covering what was wrong and what changes now.
- Keep each PR to a single change, and open an issue first for anything larger than a fix. Labels in use: `enhancement`, `considering`, `i18n`, `dependencies`. Dependabot opens dependency PRs. Vulnerable transitive dependencies are pinned through `overrides` in `pnpm-workspace.yaml`.

### Changelog and releases

- User-visible changes get an entry under `## [Unreleased]` in `CHANGELOG.md` (Keep a Changelog: `### Added`/`Changed`/`Removed`/`Fixed`). Entries are user-facing sentences, not code descriptions. Minor releases also start with a `### Highlights` section.
- `CHANGELOG.md` is bundled into the plugin (`__CHANGELOG__`). After an update, `ReleaseNotesView` shows every released section since the last version the user ran, so the release headings must stay in the `## [x.y.z] - YYYY-MM-DD` format that `parseChangelog` reads.
- A release PR (`release x.y.z`) bumps the version in `manifest.json`, `package.json` and `versions.json`, which maps the version to `minAppVersion`. It also turns `[Unreleased]` into the versioned section.
- `README.md` and `README.zh-CN.md` are kept in sync, so any README change goes into both.
- The plugin is released by pushing a bare version tag such as `2.6.0`. It must match `manifest.json` and have an entry in `versions.json`. npm packages are released with `core/<version>` or `api/<version>` tags, which must match that package's `package.json`.
