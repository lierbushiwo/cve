# Repository Guidelines

## Project Structure & Module Organization
This repository is documentation-first. The root contains Markdown write-ups such as `README.md`, `cve.md`, and case-specific reports. Keep new notes in the root unless a future `docs/` or category folder is introduced. Name files by vulnerability or product, for example `php-request-smuggling.md` or `CVE-2026-xxxx.md`.

## Build, Test, and Development Commands
There is no build pipeline in this repository. Use lightweight checks before submitting changes:

- `rg --files` lists the current document set and helps confirm naming.
- `Get-Content .\README.md` previews Markdown content in PowerShell.
- `git diff --check` catches trailing whitespace and malformed patch hunks.
- `markdownlint "**/*.md"` is recommended if you have `markdownlint` installed locally.

## Coding Style & Naming Conventions
Write concise Markdown with clear headings, short paragraphs, and fenced code blocks for payloads or reproduction steps. Save files as UTF-8 to avoid garbled text. Keep filenames descriptive and stable. Prefer exact versions, paths, and request samples over long narrative explanation.

## Testing Guidelines
Validation is manual and document-focused. Re-run any reproduction commands you document, verify request and response samples, and make sure snippets are copy-pasteable. When adding a proof of concept, include the expected environment, such as `php:8.0.6-cli`, and describe the observable result.

## Commit & Pull Request Guidelines
The current history is minimal (`Initial commit`, merge commits), so use clearer commit messages going forward. Prefer imperative, scoped subjects such as `docs: add php -S request smuggling write-up`. Pull requests should include a short summary, affected files, reproduction status, and screenshots only when they add evidence that text cannot convey. Link related CVEs or advisories when available.

## Security & Content Notes
Do not commit live targets, credentials, or sensitive internal data. Redact tokens, private IPs, and session values unless they are disposable lab artifacts. Keep exploit content reproducible in isolated environments.
