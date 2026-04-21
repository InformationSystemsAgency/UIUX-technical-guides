# Unit Testing Guide

This guide explains why unit tests are important and how to write them effectively in NewMoon.

## 1) Why unit tests matter

Unit tests give fast feedback on business logic and protect refactors.

Benefits:

- catch regressions before integration/e2e
- document intended behavior through examples
- make refactoring safer
- reduce debugging time in production issues
- improve confidence when changing shared logic

## 2) Testing in this repository

- Root and web/shared tests: Jest
- API unit tests: Jest (`apps/api`)
- API integration tests: Vitest (`apps/api`, separate config)

Use unit tests for pure logic and small service/handler behavior with mocked boundaries.

## 3) What to unit test first

Priority order:

1. business rules and transformations (`*.service.ts`, `*.mappers.ts`, `utils`)
2. validation edge cases
3. error mapping/handling branches
4. cache key builders and query builders

Lower priority for unit tests:

- UI snapshots with low business value
- trivial getters/setters

## 4) Test structure pattern

Use Arrange / Act / Assert in each test:

```ts
it('returns null when service is missing', async () => {
  // Arrange
  mockFetchService.mockResolvedValue(null);

  // Act
  const result = await getService('hy', 'missing-id');

  // Assert
  expect(result).toEqual({ service: null });
});
```

## 5) Naming conventions

Test names should describe behavior:

- `returns ... when ...`
- `throws ... when ...`
- `maps ... to ...`

Avoid vague names like:

- `works correctly`
- `should pass`

## 6) Mock boundaries, not internals

Mock only external boundaries:

- Directus SDK calls
- Meilisearch calls
- network/fetch
- current time/randomness

Avoid mocking internal private helpers unless unavoidable. Prefer testing via public function behavior.

## 7) Example: service-layer unit test

```ts
import { getPage } from './content.service';
import { executeContentQuery } from './content.service';

jest.mock('./content.service', () => {
  const actual = jest.requireActual('./content.service');
  return {
    ...actual,
    executeContentQuery: jest.fn(),
  };
});

describe('getPage', () => {
  it('returns null page when CMS has no match', async () => {
    (executeContentQuery as jest.Mock).mockResolvedValue({ pages: [] });

    const result = await getPage('hy', '/missing', 'missing');

    expect(result).toEqual({ page: null });
  });
});
```

## 8) Coverage expectations

For changed code in a PR, include tests for:

- happy path
- at least one important failure/edge path
- any new branch logic

Do not chase 100% line coverage blindly. Prefer meaningful branch and behavior coverage.

## 9) Unit test checklist (PR)

- [ ] New/changed business logic has unit tests.
- [ ] Test names describe behavior clearly.
- [ ] External dependencies are mocked at boundary level.
- [ ] Success and failure paths both covered.
- [ ] No flaky timing/network dependency in unit tests.

## 10) Commands

From repo root:

```bash
pnpm test
pnpm test:watch
pnpm test:coverage
```

For API package:

```bash
cd apps/api && pnpm test
```

Use integration tests separately when validating cross-module flows:

```bash
cd apps/api && pnpm test:integration
```
