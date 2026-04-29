# Style and Architectural Guide Pack

This folder contains a practical style guide for projects of ISAA:

- Monorepo with `pnpm` workspaces
- TypeScript everywhere
- Nuxt 4 + Vue 3 (web)
- Hono 4 + Zod/OpenAPI (api)
- Shared package for cross-app contracts

The goal is to keep standards easy to enforce in this project now, and reusable in future projects with the same technologies.

## Quick Start

[`coding-style.md`](./coding-style.md) | [`architecture-style.md`](./architecture-style.md) | [`logging-guide.md`](./logging-guide.md) | [`unit-testing-guide.md`](./unit-testing-guide.md) | [`migrations-guide.md`](./migrations-guide.md) | [`rbac-standards-guide.md`](./rbac-standards-guide.md)

## Documents

- [`coding-style.md`](./coding-style.md) - Day-to-day coding conventions with concrete examples and Mermaid diagrams.
- [`architecture-style.md`](./architecture-style.md) - Architectural conventions with module templates, sequence diagrams, and decision flows.
- [`logging-guide.md`](./logging-guide.md) - Logging standards
- [`unit-testing-guide.md`](./unit-testing-guide.md) - Unit testing standards, patterns, and anti-patterns.
- [`migrations-guide.md`](./migrations-guide.md) - Directus migration mechanism, workflow, and safety rules.
- [`rbac-standards-guide.md`](./rbac-standards-guide.md) - RBAC guide


## How to use

1. Adopt [`coding-style.md`](./coding-style.md) immediately for all new code.
2. Use [`architecture-style.md`](./architecture-style.md) when creating or changing modules/features.
3. Stay consistent with [`logging-guide.md`](./logging-guide.md).
4. Follow [`unit-testing-guide.md`](./unit-testing-guide.md) for unit test structure and coverage expectations.
5. Apply [`migrations-guide.md`](./migrations-guide.md) for all Directus schema/data changes.
6. During code review, require at least one referenced rule from each guide for non-trivial PRs.
