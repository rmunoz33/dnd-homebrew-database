# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a D&D 5th Edition database repository forked from [5e-bits/5e-database](https://github.com/5e-bits/5e-database), extended with homebrew content. It contains JSON data files for game content (races, classes, spells, monsters, etc.) that are loaded into MongoDB. The database supports both 2014 and 2024 editions of D&D 5e rules.

This repo is the data layer for the Fables and Sagas project — it feeds MongoDB Atlas, which is queried directly by `dnd-homebrew-frontend`.

## Node Version

Requires **Node.js 22.x** (pinned to 22.14.0 in `.nvmrc`).

## Commands

```bash
npm run lint                    # ESLint (includes JSON file linting)
npm test                        # Vitest — validates JSON data integrity
npm run coverage                # Tests with V8 coverage report
npm run build:ts                # Compile TypeScript scripts to scripts/built/

npm run db:refresh              # Full reload: drop all collections, reload from JSON (requires MONGODB_URI)
npm run db:update               # Incremental: only files changed in the most recent git commit (requires MONGODB_URI)
```

### Running a Single Test File

```bash
npx vitest run src/2014/tests/tables.test.js
npx vitest run src/2024/tests/tables.test.js
```

## Architecture

### Data Files

- **`src/2014/`** — 25 JSON files (complete 2014 SRD + homebrew)
- **`src/2024/`** — 14 JSON files (partial 2024 SRD + homebrew)
- Naming convention: `5e-SRD-{CollectionName}.json`
- Each file contains an array of game objects with `index`, `name`, `url`, and `source` fields

### Collection Naming

JSON files map to MongoDB collections with edition prefixes derived from the parent directory:
- `src/2014/5e-SRD-Ability-Scores.json` → `2014-ability-scores`
- `src/2024/5e-SRD-Skills.json` → `2024-skills`

### Source Attribution

Every item has a `"source"` field:
- `"srd"` — Official SRD content
- `"homebrew-base"` — Custom homebrew content
- Future variants: `"homebrew-space"`, `"homebrew-christmas-edition"`, etc.

### Database Scripts (`scripts/`)

- **`dbRefresh.ts`** — Drops all collections and reloads everything from JSON. Use when you need a guaranteed full sync.
- **`dbUpdate.ts`** — Uses `git diff` to find files changed in the latest commit and updates only those collections. Used by CI. **Will miss changes from earlier commits** — use `db:refresh` if changes span multiple commits.
- **`dbUtils.ts`** — Shared utilities: collection name extraction, index management, MongoDB URI validation.
- **`update/gitUtils.ts`** — Git diff parsing for incremental updates.
- **`update/processor.ts`** — Processes individual file additions, modifications, and deletions.

Scripts compile to `scripts/built/` via `npm run build:ts` (CommonJS output, ES2015 target).

### What Tests Validate

Tests are data integrity checks on JSON files, not unit tests on code:

1. **Duplicate indices** — Every `index` value must be unique within its JSON file
2. **Broken links** — Every `url` reference in any nested object must point to an existing entry with matching `name` and `index`

When adding or editing JSON entries, ensure index uniqueness and that all cross-references (`url`, `name`, `index` in nested objects) match existing entries.

## Adding Homebrew Content

1. Add entries to JSON files in `src/2014/` or `src/2024/`
2. Set `"source": "homebrew-base"` (or appropriate variant)
3. Use unique kebab-case `index` values
4. Ensure all `url` references point to existing entries
5. Run `npm test` to validate data integrity
6. Test with `npm run db:refresh` before committing

### Copyright Considerations

- SRD content is safe to include
- Homebrew must be sufficiently different from copyrighted WotC material
- Change names, mechanics, and flavor text to create legally distinct content

## Commit Message Format

This project uses Semantic Release:
- `feat(scope): description` — Minor release
- `fix(scope): description` — Patch release
- `perf(scope): description` with `BREAKING CHANGE:` footer — Major release

**Important**: Do NOT include AI attribution in commit messages or PR descriptions.

## CI/CD Pipeline

GitHub Actions (`.github/workflows/ci.yml`):
1. **On PRs**: Lint + tests + PR title validation
2. **On push to main**: Lint, tests, `db:update` deploy, semantic release, Docker container build

The deploy step requires `MONGODB_URI` as a GitHub repository secret.

Manual full refresh: Actions tab → "Manual Database Refresh" → Run workflow.

## MongoDB

- Set `MONGODB_URI` env var or use `.env` file (not committed)
- Production uses MongoDB Atlas; local dev can use local MongoDB

### MCP Integration

This project has MongoDB MCP configured for Claude Code, enabling direct database queries, CRUD operations, schema inspection, and Atlas cluster management.

## Upstream Sync

This repo tracks [5e-bits/5e-database](https://github.com/5e-bits/5e-database) via a git remote named `upstream`. To incorporate upstream SRD updates:

```bash
git fetch upstream
git diff main..upstream/main -- src/    # Review upstream changes to data files
git merge upstream/main                 # Merge (resolve conflicts with homebrew additions)
```

After merging, run `npm test` to verify data integrity and `npm run db:refresh` to reload MongoDB.
