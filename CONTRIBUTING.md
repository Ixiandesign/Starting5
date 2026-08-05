# Contributing to Starting5

Thanks for wanting to contribute. This is a small open-source project — keep changes focused and don't be afraid to open an issue to discuss an idea before writing code.

## Workflow

1. Fork/branch from `main`. Use a prefixed branch name: `feat/…`, `fix/…`, `chore/…`, `docs/…`, `refactor/…`, `test/…`.
2. Write tests before implementation ([test-driven development](https://en.wikipedia.org/wiki/Test-driven_development)). A PR that adds behavior without a failing-then-passing test for it will be asked to add one.
3. Commit using [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`).
4. Open a PR against `main`. CI (lint, typecheck, unit tests, e2e tests, build) must pass before merge.

## Local setup

```bash
npm install
npm run dev
```

See [README.md](README.md) for the full script list.

## Code style

ESLint + Prettier are enforced via a pre-commit hook (Husky + lint-staged) — formatting issues are caught before they reach CI. TypeScript runs in strict mode.

## Reporting bugs / suggesting features

Use the GitHub issue templates. For security issues, see [SECURITY.md](SECURITY.md) instead of opening a public issue.
