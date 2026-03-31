# Repository Guidelines

## Project Structure & Module Organization

This repository is a source snapshot centered on [`src/`](/E:/xiazai/claude-code/src), with high-level context in [`README.md`](/E:/xiazai/claude-code/README.md). Treat it as a Bun-based TypeScript CLI/TUI codebase rather than a fully packaged app.

- `src/main.tsx`: CLI entrypoint and Ink bootstrapping
- `src/commands/`: user-facing slash commands
- `src/tools/`: tool implementations and execution plumbing
- `src/components/`: Ink/React UI components
- `src/services/`: API, telemetry, sync, and background services
- `src/utils/`: shared helpers, parsers, and platform utilities
- `src/bridge/`, `src/remote/`, `src/plugins/`, `src/skills/`: integration-heavy subsystems

Keep edits scoped to the owning module. Cross-cutting changes should include a short rationale in the PR.

## Build, Test, and Development Commands

This snapshot does not include `package.json`, lockfiles, or CI config, so there is no guaranteed local build pipeline in-repo.

- `rg --files src`: inspect the codebase quickly
- `rg "symbolName|commandName" src`: locate implementations and call sites
- `git diff -- src/path/to/file.ts`: review focused changes before commit

If you restore the missing runtime files externally, prefer Bun-based workflows because the README identifies Bun as the intended runtime.

## Coding Style & Naming Conventions

Use TypeScript-first conventions already present in `src/`.

- Preserve existing indentation and import ordering in touched files
- Use `PascalCase` for React components and classes
- Use `camelCase` for functions, variables, and hooks; hooks should start with `use`
- Use `kebab-case` for command folders and feature directories when already established
- Keep modules small and avoid mixing UI, command logic, and low-level utilities in one file

## Testing Guidelines

No test suite is included in this snapshot. Until one is restored:

- validate changes by tracing call sites with `rg`
- review adjacent modules for regressions
- document manual verification steps in the PR

If you add tests later, place them near the feature or under a dedicated `test/` tree and use names like `featureName.test.ts`.

## Commit & Pull Request Guidelines

Follow the commit style already visible in Git history:

- `init: add source code from src.zip`
- `docs: rewrite README in English`

Use concise Conventional Commit prefixes such as `docs:`, `fix:`, `refactor:`, and `feat:`. PRs should include a short summary, affected paths, manual validation notes, and screenshots only for terminal UI changes.

## Security & Handling Notes

This repository appears to be a leaked source snapshot. Do not add secrets, tokens, or proprietary binaries. Prefer analysis, documentation, and narrowly scoped code changes over broad repackaging.
